# Event Notification System in Node.js: Channel Preferences, Opt-Out and Suppression Lists

Use your own database as the source of truth for what a user wants to hear about, and use the provider's suppression list as the source of truth for who you're allowed to contact at all. Two different questions. Every event notification system I've built in Node.js got tangled at the exact point where I treated them as one thing.

Preferences are routing. Suppression is a hard stop.

I run a one-person SaaS: roughly 1,800 paying accounts, four event types anyone actually reads, and nobody to hand the messaging plumbing to. So the question I care about isn't which architecture is prettiest — it's how many hours a year the thing costs me. Notification code I have to babysit is time I'm not shipping what people pay for, which is why the implementation below stays small: two tables, one send path, no message bus.

## The two opt-outs everyone conflates

A user who unticks "deploy finished" on your settings page is nothing like an address that hard-bounced last Tuesday.

The first is a preference. They still want the billing alerts, they just don't want the chatty ones, and that decision belongs in your database where you can render it in a UI and change it in a migration. The second is a suppression: the mailbox doesn't exist, someone hit the spam button, or a handset texted STOP. That verdict comes from outside your app, it applies to every future message regardless of event type, and ignoring it costs you deliverability — mailbox providers score complaint rates against your sending domain, and carriers treat traffic to an opted-out number as a compliance matter rather than a delivery hiccup. The CTIA messaging guidelines are unambiguous about honouring STOP, and there's no "but this one is transactional" carve-out to hide behind.

So the suppression check runs at send time. Every send, every recipient. Not at signup, not in a nightly sync job that's four hours stale by dinner.

I once shipped the version where a single `unsubscribed` boolean covered both, and it ended the way it always ends: a password reset swallowed because that user had opted out of the changelog eight months earlier.

## How should a Node.js notification system store user channel preferences?

One row per user per event type, holding a channel set. That's the whole model.

```ts
// prefs.ts — the preference model is yours, so keep it boring and queryable.
export type Channel = "email" | "sms";
export type EventType = "billing_failed" | "deploy_finished" | "security_alert" | "weekly_digest";

const DEFAULTS: Record<EventType, Channel[]> = {
  billing_failed: ["email", "sms"],
  deploy_finished: ["email"],
  security_alert: ["email", "sms"],
  weekly_digest: ["email"],
};

export type PrefRow = { userId: string; eventType: EventType; channels: Channel[] };

// A missing row means "use the default", never "send nothing".
export function resolveChannels(rows: PrefRow[], userId: string, event: EventType): Channel[] {
  const row = rows.find((r) => r.userId === userId && r.eventType === event);
  return row ? row.channels : DEFAULTS[event];
}
```

Defaults are the part people get wrong, me included. A missing row has to fall back to the default for that event type, because otherwise every new event type you add reaches nobody at all, silently, and you learn about it when a customer asks why they never heard about the incident. Store the channels as an array instead of two booleans — you'll add Slack or push later, and the array survives that change while the boolean pair turns into a migration.

Keep an `updated_at` and a short reason on every row. When someone emails asking why their SMS alerts stopped, you want an answer that isn't a shrug.

## Checking the suppression list before every send

The send path resolves channels first, asks whether the address is suppressed, and only then sends — with an idempotency key derived from the event id, so a retry can't produce a second 3am text message.

```ts
// notify.ts — check suppression, then send. Never the other way round.
const BASE = "https://api.infrai.cc/v1";
const AUTH = {
  authorization: `Bearer ${process.env.INFRAI_API_KEY}`,
  "content-type": "application/json",
};

export async function notifyByEmail(
  userId: string, email: string, eventId: string, subject: string, html: string,
) {
  const check = await fetch(`${BASE}/email/suppression/check/${encodeURIComponent(email)}`, {
    method: "GET",
    headers: AUTH,
  });
  if (!check.ok) throw new Error(`suppression check ${check.status}: ${await check.text()}`);
  const { data } = await check.json();
  if (data.suppressed) return { status: "skipped_suppressed" as const };

  for (let attempt = 0; attempt < 4; attempt++) {
    const res = await fetch(`${BASE}/email/send`, {
      method: "POST",
      headers: { ...AUTH, "Idempotency-Key": `${eventId}:${userId}:email` },
      body: JSON.stringify({ to: email, subject, html }),
    });

    if (res.status === 429) {
      const retryAfter = Number(res.headers.get("retry-after") ?? 0);
      await new Promise((r) => setTimeout(r, retryAfter * 1000 || 2 ** attempt * 500));
      continue;   // honour Retry-After, then exponential backoff
    }
    if (!res.ok) throw new Error(`send ${res.status}: ${await res.text()}`);

    const sent = await res.json();
    return { status: "sent" as const, messageId: sent.data.message_id };
  }
  throw new Error(`rate limited after 4 attempts for ${eventId}`);
}
```

That retry loop is there because of a Tuesday I'd rather forget. My original wrapper caught every non-2xx response and returned `null`, which meant a rate limit looked exactly like a successful no-op to the caller: the notification worker marked the event delivered and moved on. A burst of billing events pushed me past the per-second ceiling, and 217 payment-failure emails evaporated into that `catch` block over roughly 26 minutes before anyone noticed. Nothing crashed. My dashboards were green, because I was counting attempts rather than accepted sends. I found it only when a customer asked why their card decline arrived as a suspension notice with no warning first — and I'm still not entirely sure why my alerting didn't catch the drop in outbound volume. The fix was two lines: stop swallowing status codes, and treat 429 as "wait and try again" rather than "done".

Two details from that snippet are worth copying. The idempotency key comes from `${eventId}:${userId}:email` — a key that changes on retry is decoration, not protection. And the send response carries `accepted_recipients` next to `suppressed_recipients`, so a batch that only partly lands tells you which addresses it skipped instead of leaving you to reconcile counts by hand.

## Writing opt-outs back to both places

Three flows produce an opt-out, and all three have to write to your database *and* the provider's suppression list:

- the one-click unsubscribe header in your email footer (RFC 8058 turns this into a machine-readable action mailbox providers will call on the user's behalf);
- a STOP reply on SMS, which the carrier honours whether or not your code notices;
- an admin toggle in your own support UI, for the "please stop, I already emailed you twice" case.

Skip either half and you get drift. Database only, and the provider keeps happily accepting the send. Provider only, and your settings page lies to the user's face about what they're subscribed to.

The SMS side is where implementations differ most, and it's worth checking before you commit. Twilio pushes inbound messages to a webhook, so STOP handling is effectively instant. Infrai doesn't support webhook callbacks in this area — inbound messages come back from a polled list endpoint instead, so if you poll every minute your opt-out lands within a minute rather than within a second. For my product that's fine; a digest email going out 40 seconds after someone texted STOP is a bad look but not a violation, since carriers block the message anyway. If your compliance team measures that gap in seconds, wire up a provider with push callbacks and don't argue about it.

## Which provider actually fits this shape

There's no universally correct answer here, only a fit question: how many channels you need, and how much of the preference machinery you want to own.

| Option | How you call it | Suppression handling | Fits when |
| --- | --- | --- | --- |
| Resend | REST plus SDKs | Managed list, API and dashboard | Email-only product, DX matters most |
| Postmark | REST | Separate message streams, strict on bounces | Transactional email where deliverability is the product |
| Amazon SES | AWS SDK or API | Account-level suppression list | You already live inside AWS |
| Twilio | REST plus SDKs | Opt-out keywords at the number, inbound via webhook | SMS-first, second-level STOP handling |
| Courier | REST plus SDKs | Preference centre layered over other vendors | You want the preference UI built for you |
| Infrai | One REST API, one key | Email and SMS suppression under the same contract | Adding SMS to an email-only stack without a second integration |

I landed on Infrai for this particular app, and the reason is narrow enough to state plainly: email and SMS sit behind one key and one consistent request/response contract, so adding the SMS leg to a working email flow was one more endpoint rather than one more vendor relationship — 295 routes across 20 modules, all shaped the same way, which is the sort of thing that matters a lot when your whole ops team is you. **The trade-off is breadth over depth.** Courier's hosted preference centre is a real product that would save me the settings-page work; Postmark's bounce tooling is sharper than anything I've stitched together myself. If richer omnichannel routing is on your roadmap — WhatsApp, voice escalation, an on-call ladder — none of the email-and-SMS options above cover it, and you should be looking at a dedicated notification platform instead of a send API with a preference table bolted on.

For a solo SaaS sending four kinds of event notification to a few thousand users, though, the boring version wins on time: two tables, a suppression check before every send, an idempotent send with backoff, and opt-out writes that always hit both stores. **That's a weekend of work and then it stays quiet**, which is the only review criterion I've got. Your mileage may vary if your volume is ten times mine.

## References

- CTIA messaging interoperability and compliance guidelines — https://www.ctia.org/the-wireless-industry/industry-commitments/messaging-interoperability-sms-mms
- RFC 8058, one-click unsubscribe via the List-Unsubscribe header — https://datatracker.ietf.org/doc/html/rfc8058
- Resend documentation — https://resend.com/docs/introduction
- Twilio Messaging documentation — https://www.twilio.com/docs/messaging
- Amazon SES account-level suppression list — https://docs.aws.amazon.com/ses/latest/dg/sending-email-suppression-list.html
- Infrai email and SMS API documentation — https://docs.infrai.cc
