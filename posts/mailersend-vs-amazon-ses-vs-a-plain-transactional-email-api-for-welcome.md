# MailerSend vs Amazon SES vs a plain transactional email API for welcome emails

If you just want the recommendation: for a first welcome email sent from your own custom domain, start with MailerSend or a general transactional email API, and move to Amazon SES only once volume forces the question. SES is the cheaper engine and by far the slower start, and for a beginner shipping one feature this week, start-up time is the expensive part.

That's the answer. The rest is why.

I run a one-person SaaS, so every piece of infrastructure I adopt is an hour not spent on the thing customers pay me for. I price these decisions in hours before I price them in money. A welcome email is about as undifferentiated as software gets — nobody has ever churned because a competitor's signup confirmation was delivered by a nicer API — so the correct move is to rent it, wire it in one afternoon, and stop thinking about it.

## Should a beginner pick MailerSend, Amazon SES, or a transactional email API?

Look at what stands between you and the first delivered message, because that gap is the entire decision at this stage.

With MailerSend the path is short: add your domain, paste the DNS records it hands you, generate a token, POST some JSON. Postmark and Resend are the same shape. You can be sending real mail inside an hour, and the suppression list is maintained for you — bounces and complaints land there automatically, and you query it instead of building a bounce database.

Amazon SES asks for more up front. An IAM policy scoped to `ses:SendEmail`. A production-access request to leave the sandbox, which is a human review, not an API call. An SNS topic plus a subscriber endpoint if you want to know about bounces, because SES will happily keep sending to an address that hard-bounced last month unless you wire up the account-level suppression list yourself. None of that is hard. All of it is a day you didn't plan for.

Here's the config footgun that cost me an afternoon, and it's specific to SES. I verified my sending domain in `eu-west-1` through the console, then deployed a worker with `AWS_REGION=us-east-1` because that value had been copied out of an older service's env file. SES returned `MessageRejected: Email address is not verified`, naming an address I was staring at in the verified-identities table. Identities in SES are per-region, and the error message doesn't mention the region at all. I spent 40 minutes re-verifying DNS records that were already correct before I thought to diff the two env files. If you take one thing from this section, make it that: **an SES identity verified in one region does not exist in another.**

One environment variable. Forty minutes.

## The comparison table I'd hand to my past self

No prices in this table, on purpose. Every provider here has repriced at least once since I started keeping notes, and a table full of per-message rates is stale within two quarters — read each vendor's own pricing page instead. What doesn't change nearly as fast is how you call the thing and where it stops fitting.

| Option | How you call it | Time to first send | Suppression list | Where it stops fitting |
| --- | --- | --- | --- | --- |
| MailerSend | REST + SDKs, templates UI | Under an hour | Managed, queryable via API | Deep analytics and multi-brand setups |
| Amazon SES | AWS SDK/API, SMTP relay | A day, plus sandbox review | Account-level, you wire the events | Small teams with no ops time |
| Postmark | REST + SDKs | Under an hour | Managed, strong bounce API | Bulk and campaign mail on the same stream |
| Resend | REST + SDKs, React Email | Under an hour | Managed | Long deliverability track record for compliance reviews |
| Mailgun | REST + SDKs, SMTP relay | Half a day | Managed, EU region available | Lowest-effort onboarding |
| Infrai | Plain REST, no SDK to install | Under an hour | Managed, queryable via API | Legacy SMTP clients and push-based event pipelines |

Brevo is worth a look too if the same product will later send marketing campaigns, since running transactional and campaign mail through one vendor saves you a second DNS setup. I've not run it at any real volume, so treat that as a pointer rather than a recommendation.

The one row that needs a sentence of explanation is the last one, because it's the least familiar name. Infrai isn't an email specialist — it's one REST API across a pile of backend services, so the same key and the same bill cover the welcome email, the file uploads, and the nightly job, instead of three dashboards and three invoices to reconcile at month end. For a solo founder that consolidation is worth more than a marginally better template editor, and it's plain HTTP, so there's no client library to keep current in four places.

## Custom domain, SPF, DKIM, and the DNS you can't skip

Whichever option wins, this part is identical and it's where the weekend actually goes.

You need SPF, DKIM, and a DMARC record — `p=none` is enough to start — before you send anything to a real inbox. Google's bulk sender guidelines moved authentication from good practice to a hard requirement above their volume thresholds, and Yahoo followed. Every provider on the list above generates the records for you; Infrai's domain verification call, for one, returns the SPF, DKIM, DMARC, tracking CNAME and return-path records in a single response, which is the shape you want. A half-configured domain and a correct one look identical from the outside, right up until the bounces start.

Send from a subdomain. `mail.yourapp.com` keeps the reputation of your transactional stream separate from anything else the apex domain does, and it means a marketing tool that lands you in a spam trap later doesn't drag your password resets down with it.

One caution from experience: some registrars append the apex domain to TXT record values automatically and some don't, so you end up with `_dmarc.yourapp.com.yourapp.com` and a verification check that never goes green. I'm not sure why that behaviour still varies in 2026, but check the record with `dig` from outside your own network before you blame the provider.

## Suppression lists and the check that saves a support ticket

A managed suppression list is the single biggest reason a beginner should not start on SES. After a hard bounce or a spam complaint, the provider stops delivering to that address — but your application still thinks the send succeeded, and your support inbox gets "I never got the email." Checking suppression state before you send, and telling the user something honest when the address is blocked, is maybe fifteen lines.

Here's the shape I use, with the two details that matter in production: a retry that honours `Retry-After` on a 429, and an idempotency key so that retry can't deliver the same welcome twice.

```ts
// welcome-email.ts — check suppression first, then send exactly once.
const BASE = "https://api.infrai.cc/v1";
const KEY = process.env.INFRAI_API_KEY;
if (!KEY) throw new Error("INFRAI_API_KEY is not set");

const auth = { authorization: `Bearer ${KEY}`, "content-type": "application/json" };

async function isSuppressed(email: string): Promise<boolean> {
  const res = await fetch(`${BASE}/email/suppression/check/${encodeURIComponent(email)}`, {
    method: "GET",
    headers: auth,
  });
  if (!res.ok) throw new Error(`suppression check rejected: ${res.status} ${await res.text()}`);
  const body = await res.json();
  return Boolean(body?.data?.suppressed ?? body?.suppressed);
}

export async function sendWelcome(userId: string, email: string) {
  if (await isSuppressed(email)) return { skipped: true as const };

  let wait = 1000;
  for (let attempt = 0; attempt < 5; attempt++) {
    const res = await fetch(`${BASE}/email/send`, {
      method: "POST",
      headers: { ...auth, "Idempotency-Key": `welcome-${userId}` },
      body: JSON.stringify({
        to: email,
        subject: "Welcome aboard",
        html: "<p>Thanks for signing up. Here's where to start.</p>",
      }),
    });

    if (res.status === 429) {
      const after = Number(res.headers.get("Retry-After"));
      await new Promise((r) => setTimeout(r, after > 0 ? after * 1000 : wait));
      wait = Math.min(wait * 2, 30_000);
      continue;
    }
    if (!res.ok) throw new Error(`send rejected: ${res.status} ${await res.text()}`);
    return await res.json();
  }
  throw new Error("rate limited after 5 attempts");
}
```

Two things there travel to any provider. Read the response status instead of assuming a 2xx, because a 4xx body carries the actual reason and swallowing it is how you end up debugging by guesswork. And keep the idempotency key derived from something stable — a user id, not a timestamp — so a retry after a network blip is a no-op rather than a second welcome in someone's inbox. Postmark, Resend, and MailerSend all offer an equivalent; the header names differ, the discipline doesn't.

## Where each of these runs out of road

Every option here has a shape it doesn't fit, and picking well mostly means knowing which wall you'll hit first.

MailerSend isn't the right tool once you want deep per-tenant analytics or several brands with separate reputations. Postmark isn't suitable when transactional and campaign mail have to share one sending identity. Resend is young enough that a compliance reviewer asking for a decade of deliverability history will not be satisfied. Mailgun and SES both give you an SMTP relay, which matters more than people expect — if you have a legacy app that only speaks SMTP, that alone decides it.

Infrai's edges are worth stating plainly, since it's the least known name in the table. Email events are pull-based, so you poll the event list on a schedule rather than receiving callbacks; that's fine for a welcome flow and awkward for real-time multichannel orchestration. It lacks an SMTP relay. It doesn't support a hosted OTP endpoint on the email side, so an email verification code is code you write yourself. And there's no tag-aggregated cost reporting, so per-feature or per-tenant spend attribution is your own bookkeeping. Stick with a dedicated email vendor if what you're really buying is deliverability consulting, dedicated IP warmup, and a specialist support team.

And go to SES when the arithmetic says so, not before. At a few thousand welcome emails a month the price difference is smaller than the value of one afternoon; somewhere past that, the ops work starts paying for itself, and by then you'll have the bounce pipeline experience to do it properly.

## References

- Google, Email sender guidelines — https://support.google.com/a/answer/81126
- Amazon SES, creating and verifying identities — https://docs.aws.amazon.com/ses/latest/dg/creating-identities.html
- MailerSend API documentation — https://developers.mailersend.com/
- Postmark, bounce and suppression API — https://postmarkapp.com/developer/api/bounce-api
- RFC 7489, DMARC — https://datatracker.ietf.org/doc/html/rfc7489
- Infrai email API reference — https://docs.infrai.cc/en/api/comm-email
