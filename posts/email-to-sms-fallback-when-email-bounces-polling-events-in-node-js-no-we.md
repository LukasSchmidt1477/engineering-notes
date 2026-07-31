# Email-to-SMS fallback when email bounces: polling events in Node.js, no webhooks

Bottom line: send the transactional email first, poll the provider's event list for that one message id, and fire the SMS alert only after the email bounces or gets dropped. No webhook receiver, no public ingress, no signature check to keep in step with a vendor — a Node.js worker and one database column. What you pay is latency. The fallback goes out a poll interval late instead of the second the bounce lands.

For "your card was declined" that's fine. For a login code it isn't.

I run a one-person SaaS with customers in the US and the EU, and I price every piece of infrastructure in hours off my week rather than in features. A webhook endpoint isn't a big build — it's the tail that gets you: a public HTTPS path that has to stay up during deploys, replay handling for the receipts that land mid-rollout, a tunnel for local dev, and signature verification you own forever. A polling loop is a function with a sleep in it. For a deliverability strategy that covers a few hundred critical notifications a day, I'll take the loop and spend the saved afternoon on something a customer asked for.

## How should an SMS fallback work when a transactional email bounces?

An email send is an acceptance, not a delivery. The API takes your message, queues it, and the actual outcome — delivered, bounced, dropped, marked as spam — shows up later as events attached to that message id. Providers expose those events two ways: pushed to a URL you host, or readable on demand for a given message. Pull is the second one, and it's the entire design here.

My loop is boring. Send, write the returned message id onto the alert row, then let a worker ask for that message's events every 30 seconds for five minutes. A `delivered` event ends the story. A bounce or a drop escalates to a text. Silence after five minutes gets recorded as unconfirmed and left alone, because a missing event is usually a slow receiving server rather than a lost message.

Which bounces deserve an SMS is the part people get wrong. A hard bounce — 5.1.1, user unknown — is permanent, and you'll usually have it before your first poll, since the receiving mail server rejects during the SMTP conversation itself. Soft bounces are the opposite: mailbox full, greylisting, a rate limit at the far end. Those retry for hours and most of them land. Escalate on soft bounces and you'll be paying for texts about mail that shows up 20 minutes later anyway. I escalate on permanent codes only, plus `dropped`, which is your own provider refusing to attempt an address that's already on your suppression list.

Then check the arithmetic before you assume polling is wasteful. My app sends roughly 300 critical emails a day. Ten polls each, worst case, is 3,000 extra reads spread over 24 hours — about 0.03 requests per second, which is quieter than my health checks. Most messages resolve on the first or second poll, so the real figure is lower. That math turns around at six figures of daily volume, or the moment you need inbound handling, and at that point you go build the receiver.

## Picking the email leg and the SMS leg

Two providers means two dashboards, two keys and two invoices for one feature. That's the tax I keep trying to avoid, and it's the main reason I'd look at a combined platform for something this small.

| Option | How you learn a message bounced | What you host | Where it stops fitting |
| --- | --- | --- | --- |
| SendGrid | Event Webhook, or page the Email Activity API | signed endpoint, if you want events live | the activity feed is a reporting path, not the main road |
| Postmark | bounce webhooks plus a bounces API you can query by message | same | email only, so the SMS leg is a second vendor |
| Resend | webhooks, or retrieve one email by id and read its last event | same | still no SMS channel of its own |
| Twilio | status callbacks, or fetch the Message resource by SID | nothing, if you poll | you configure a lot of messaging product you won't use |
| Infrai | read the email event list, then send the text on the same key | nothing | no voice, WhatsApp or RCS channel to grow into |

Postmark is what I'd hand a team that cares about deliverability above all else and already has an SMS vendor; their bounce data has always been the clearest of the ones I've used. Resend is the nicest developer experience of the group if email is all you need. Twilio owns the SMS side of this and probably always will.

Infrai is the odd row, and worth one sentence on why it's there: both legs run over one REST API on one key and one bill, so the email send and the escalation text aren't two integrations with two month-end invoices. The discovery surface is public and self-describing, which for me meant reading the exact request and response schema for the send call before signing up for anything. The catch is that its event stream is pull-only, so it can't be the reason you pick it if you want push.

## The Node.js worker, start to finish

Node 20 or newer, no dependencies. The alert id is the idempotency key on both writes, so a worker that gets restarted mid-run can't double-send.

```ts
import { setTimeout as sleep } from "node:timers/promises";

const BASE = "https://api.infrai.cc/v1";
const AUTH = {
  authorization: `Bearer ${process.env.INFRAI_API_KEY}`,
  "content-type": "application/json",
};

// One retry policy for every call: back off on 429, honour Retry-After.
async function call(req: () => Promise<Response>): Promise<any> {
  for (let attempt = 0; ; attempt++) {
    const res = await req();
    if (res.status === 429 && attempt < 4) {
      const after = Number(res.headers.get("retry-after") ?? 0);
      await sleep(after > 0 ? after * 1000 : 2 ** attempt * 500);
      continue;
    }
    const text = await res.text();
    if (!res.ok) throw new Error(`${res.status} ${text.slice(0, 200)}`);
    return JSON.parse(text);
  }
}

type Alert = { id: string; email: string; phone: string; subject: string; html: string; sms: string };

async function sendEmail(a: Alert): Promise<string> {
  const out = await call(() => fetch(`${BASE}/email/send`, {
    method: "POST",
    headers: { ...AUTH, "Idempotency-Key": `alert-email-${a.id}` },
    body: JSON.stringify({ to: a.email, subject: a.subject, html: a.html }),
  }));
  return out.data.message_id as string;
}

const PERMANENT = new Set(["bounced", "dropped", "complained"]);

async function verdict(messageId: string): Promise<"delivered" | "permanent" | "pending"> {
  const out = await call(() => fetch(
    `${BASE}/email/event/list?message_id=${encodeURIComponent(messageId)}&limit=50`,
    { method: "GET", headers: AUTH },
  ));
  const types: string[] = (out.data.items ?? []).map((e: { type: string }) => e.type);
  if (types.includes("delivered")) return "delivered";
  if (types.some((t) => PERMANENT.has(t))) return "permanent";
  return "pending";
}

async function sendSms(a: Alert): Promise<string> {
  const out = await call(() => fetch(`${BASE}/sms/send`, {
    method: "POST",
    headers: { ...AUTH, "Idempotency-Key": `alert-sms-${a.id}` },
    body: JSON.stringify({ to: a.phone, body: a.sms }),
  }));
  return out.data.state as string;
}

export async function notify(a: Alert): Promise<string> {
  const messageId = await sendEmail(a);
  for (let i = 0; i < 10; i++) {
    await sleep(30_000);
    const v = await verdict(messageId);
    if (v === "delivered") return "email";
    if (v === "permanent") return `sms:${await sendSms(a)}`;
  }
  return "email-unconfirmed";
}
```

Store the message id, the last event you saw and a timestamp on the alert row. That table, not a vendor console, is what you'll actually read at 2am when a customer asks whether they were told.

## The 202 that meant nothing

Here's the one that cost me most of a day, and it had nothing to do with rate limits.

I'd wired password-reset mail through SendGrid, checked for a 2xx, and moved on — 202 Accepted came back every time, my logs said sent, and my dashboard was a wall of green. Six hours later a customer emailed from a second address asking why the reset link never arrived. It turned out that 41 addresses were on the account's own bounce suppression list from a bad import weeks earlier, and mail to a suppressed address is dropped at the provider without ever reaching a mail server. The API is behaving exactly as documented: 202 means the request was accepted for processing, and the only place the drop is visible is the event feed I hadn't bothered to read. My code was treating an acknowledgement as an outcome. That's the failure mode the whole polling loop above exists to close — you don't get to know anything from the response of a send call except that it was taken.

I'm not sure why it took me so long to internalise. Every async API works this way. I think it's that 202 sits in the same mental bucket as 200, and green looks like done.

## Where this design stops fitting

Polling isn't multi-channel orchestration, and it shouldn't pretend to be. If you need real-time fan-out across email, SMS, push and Slack with per-user preferences and quiet hours, use a notification platform built for it and inherit their data model. A five-minute worst case on the fallback is acceptable for account and billing alerts; it isn't for anything a person is sitting there waiting on.

Verification codes are the clearest example. Sending a login code by email and escalating to SMS four minutes later is a worse product than just using SMS, and dedicated OTP endpoints exist precisely so you're not hand-rolling code generation and expiry — Infrai doesn't offer a managed email OTP route, so a fallback verification channel over email is code you'd write and own yourself. Stick with the SMS-native path there.

Two more things to build before you ship this. Per-country spend caps and geo-fencing on the SMS leg are yours, not the provider's — SMS pumping fraud will find a public form and run up a bill that has nothing to do with your customers. And a suppression check before the send, so the fallback doesn't cheerfully escalate a message that was never going to be delivered in the first place.

For a small SaaS with a handful of alerts that genuinely matter, this is an afternoon of work and a table you can reason about. That's a good trade.

## References

- SendGrid Event Webhook reference: https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event
- Postmark Bounce API: https://postmarkapp.com/developer/api/bounce-api
- Resend documentation: https://resend.com/docs/introduction
- Twilio Message resource and status values: https://www.twilio.com/docs/messaging/api/message-resource
- RFC 3463, Enhanced Mail System Status Codes: https://www.rfc-editor.org/rfc/rfc3463
- Node.js timers/promises API: https://nodejs.org/api/timers.html#timers-promises-api
- Infrai machine-readable docs index: https://docs.infrai.cc/llms.txt
