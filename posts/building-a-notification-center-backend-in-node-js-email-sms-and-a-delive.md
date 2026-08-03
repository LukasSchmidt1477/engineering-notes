# Building a notification center backend in Node.js: email, SMS, and a delivery audit log

Use your own notification center backend — a small Node.js service with an events table, a deliveries table, and a polling API — when the delivery history and the audit log are things you or your users actually read. Otherwise reach for a hosted notification platform like Courier or Knock, wire two webhooks, and go back to shipping the thing people pay you for.

I run a one-person SaaS, so every infrastructure decision is an hour of my week against a feature that could close a deal. This one earned its hour. Support tickets that used to be "did the invoice email go out?" became a link to a delivery history page, and the guessing stopped.

Here's the version I'd build again, with the parts I got wrong the first time.

## What a notification center actually stores

Three tables. The split between them matters far more than which library or provider you pick, because it's what lets you re-send without lying about history.

An event is the fact that something happened: invoice paid, comment posted, export finished. It belongs to a user, it carries a payload, and it never changes. A delivery is one attempt to get that event to one channel — in-app, email, SMS — and it has a mutable status. The audit log is a third, append-only table that records every status transition anyone reported, including the ones the provider pushed back to you hours later.

```sql
create table notification_events (
  id          bigserial primary key,
  user_id     uuid        not null,
  kind        text        not null,
  payload     jsonb       not null default '{}',
  dedupe_key  text        unique,
  created_at  timestamptz not null default now()
);

create table notification_deliveries (
  id            bigserial primary key,
  event_id      bigint      not null references notification_events(id),
  channel       text        not null,          -- inapp | email | sms
  status        text        not null default 'queued',
  provider      text,
  provider_id   text,                          -- the id the provider gave back
  attempts      int         not null default 0,
  next_retry_at timestamptz,
  updated_at    timestamptz not null default now()
);

create table delivery_events (                 -- append-only audit log
  id          bigserial primary key,
  delivery_id bigint      not null references notification_deliveries(id),
  status      text        not null,            -- queued|sent|delivered|bounced|dead
  detail      text,
  at          timestamptz not null default now()
);

create index on notification_deliveries (status, next_retry_at);
```

The `dedupe_key` is the one column I'd defend in an argument. Write it as something the caller can compute — `invoice_paid:${invoiceId}` — and a retried webhook, a double-clicked button, or a replayed queue message all collapse into the same event instead of three copies of the same email. Everything downstream inherits that guarantee for free.

## Should my notification API push events or let clients poll for delivery history?

Poll. Almost always poll.

A bell icon that refreshes every 30 seconds needs one cursor-paginated endpoint and no connection state anywhere. Server-sent events are lovely, and I've used them for a live editor, but for notifications they buy you a few seconds of latency in exchange for load balancer timeouts, reconnect logic, and a per-user connection you now have to think about during a deploy. Cursor on `(created_at, id)`, never `offset` — offset pagination drifts the moment a new row lands between two requests, and users see duplicates.

```ts
import express from "express";
import { pool } from "./db.ts";

const app = express();

// GET /api/notifications?since=<iso>_<id>&limit=50
app.get("/api/notifications", async (req, res) => {
  const userId = (req as any).user.id;          // whatever your auth middleware sets
  const [ts, id] = String(req.query.since ?? "").split("_");
  const limit = Math.min(Number(req.query.limit ?? 50), 100);

  const { rows } = await pool.query(
    `select e.id, e.kind, e.payload, e.created_at,
            d.channel, d.status, d.updated_at
       from notification_events e
       join notification_deliveries d on d.event_id = e.id
      where e.user_id = $1
        and ($2::timestamptz is null or (e.created_at, e.id) > ($2::timestamptz, $3::bigint))
      order by e.created_at, e.id
      limit $4`,
    [userId, ts || null, id || 0, limit],
  );

  const last = rows.at(-1);
  res.set("Cache-Control", "no-store").json({
    items: rows,
    cursor: last ? `${last.created_at.toISOString()}_${last.id}` : req.query.since ?? null,
  });
});

app.listen(3000);
```

One endpoint, one index, no websocket budget. If the polling interval ever shows up in your database CPU graph, cache the unread count in Redis and let the full list stay on the slow path — the count is what gets requested 50 times per list fetch.

## Which email and SMS providers to wire underneath

This is the undifferentiated part, so I buy it. The choice is less about features than about which failure mode you want to own.

| Provider | What I reach for it for | Delivery events | The catch |
| --- | --- | --- | --- |
| Postmark | transactional email, opinionated setup | bounce/delivery/open webhooks, separate message streams | strict about mixing broadcast traffic into a transactional stream |
| Resend | small teams that want plain REST and React templates | per-email webhooks | younger ecosystem, fewer deliverability knobs than the incumbents |
| SendGrid | high volume with marketing and transactional together | signed event webhook, batched | big product surface; the console is a project of its own |
| Amazon SES | volume, if your stack already lives in AWS | configuration sets into SNS or EventBridge | region-scoped identities, sandbox until you request production access |
| Twilio | SMS in more than one country | per-message status callbacks | number provisioning and country rules are the actual work |
| Courier / Knock | not building this layer at all | their dashboard, their schema | you inherit their data model for preferences and history |

The config footgun that actually cost me an evening was SES. My worker had `AWS_REGION=us-east-1` baked into the deploy environment, while the domain identity was verified in `eu-west-1`. Local runs went out fine, so I'd shipped it. Production came back with `Email address is not verified. The following identities failed the check in region US-EAST-1`, naming a domain I had verified weeks earlier and could see sitting green in the console. I assumed the region only changed latency and routing. It doesn't: identities, suppression lists, and configuration sets are all per-region, so a one-word env var turned into 45 minutes of re-reading DKIM records that were correct the entire time. Now the region goes in the same config object as the API key, and the worker logs both at boot.

Vonage and Plivo are reasonable SMS alternatives, and I'm not sure the difference matters below a few thousand messages a month — your mileage may vary once you're sending into markets with sender ID registration.

## The worker, the retries, and what never goes in the audit log

The worker claims rows with `for update skip locked`, calls one provider, and writes exactly one audit row per outcome. Nothing else. Keep it boring enough that you can read the whole file in a minute.

```ts
type Delivery = { id: number; channel: "email" | "sms"; to: string; subject: string; body: string };

const senders = {
  async email(d: Delivery): Promise<string> {
    const r = await fetch("https://api.resend.com/emails", {
      method: "POST",
      headers: {
        authorization: `Bearer ${process.env.RESEND_API_KEY}`,
        "content-type": "application/json",
        "idempotency-key": `delivery-${d.id}`,
      },
      body: JSON.stringify({
        from: "alerts@example.com",
        to: d.to,
        subject: d.subject,
        text: d.body,
      }),
    });
    if (!r.ok) throw new Error(`resend ${r.status} ${await r.text()}`);
    return (await r.json()).id;
  },

  async sms(d: Delivery): Promise<string> {
    const sid = process.env.TWILIO_ACCOUNT_SID!;
    const auth = Buffer.from(`${sid}:${process.env.TWILIO_AUTH_TOKEN}`).toString("base64");
    const r = await fetch(`https://api.twilio.com/2010-04-01/Accounts/${sid}/Messages.json`, {
      method: "POST",
      headers: { authorization: `Basic ${auth}`, "content-type": "application/x-www-form-urlencoded" },
      body: new URLSearchParams({
        To: d.to,
        From: process.env.TWILIO_FROM!,
        Body: d.body,
        StatusCallback: "https://example.com/hooks/twilio",
      }),
    });
    if (!r.ok) throw new Error(`twilio ${r.status} ${await r.text()}`);
    return (await r.json()).sid;
  },
};

export async function dispatch(d: Delivery): Promise<void> {
  try {
    const providerId = await senders[d.channel](d);
    await record(d.id, "sent", { providerId });
  } catch (err) {
    const attempts = await bumpAttempts(d.id);            // returns the new count
    const backoffMs = Math.min(2 ** attempts * 1000, 15 * 60_000);
    await record(d.id, attempts >= 5 ? "dead" : "queued", {
      detail: String(err),
      nextRetryAt: new Date(Date.now() + backoffMs),
    });
  }
}
```

Status callbacks close the loop. Twilio posts `queued`, `sent`, `delivered`, `undelivered` per message; the email providers post bounces and complaints. Each of those becomes another `delivery_events` row keyed by `provider_id`, which is why that column exists — the provider's id is the only join key you'll have when a webhook arrives 40 minutes after the send. Verify the signature on every hook, and treat an unknown `provider_id` as a no-op rather than an error, because retries from their side will outlive your deploys.

Now the part people skip. Your audit log is a place where secrets go to live forever, and a password reset link in `payload` is a working credential sitting in a table your entire support team can read. OWASP's guidance on forgot-password flows is blunt about not letting reset tokens leak into side channels, and a delivery history page is a side channel. Store the template id and the variables you'd be comfortable printing on a receipt; store a hash of the token if you need to prove one was sent. For anything that looks like marketing rather than a receipt, keep it on a separate provider stream with real unsubscribe handling — the FTC's CAN-SPAM guidance treats the unsubscribe path as non-optional, and mixing broadcast into your transactional stream is the fastest way to hurt the deliverability of mail people actually want.

## Where this design falls down

Postgres as a queue works until it doesn't. Somewhere north of a few hundred deliveries per second, the `skip locked` polling loop starts fighting autovacuum on a hot, constantly-updated table, and you'll be happier with SQS or Redis Streams carrying the work while Postgres keeps only the audit trail. That's a real trade-off, not a rewrite you should pre-buy at 40 notifications an hour.

It's also the wrong shape if what you need is a preference center. Per-user, per-channel, per-category opt-outs plus digests plus quiet hours is a product in itself, and rebuilding it costs weeks; stick with Courier or Knock if that's the requirement, and accept their schema. Same answer if you need in-app inbox UI components rather than a JSON endpoint.

And if your users expect chat-grade immediacy, polling doesn't support that feel — switch to SSE or a websocket for the in-app channel and keep the polling endpoint as the fallback for reconnects.

Everything else, I'd keep. Three tables, one worker, one endpoint, providers I can swap in an afternoon.

## References

- OWASP Forgot Password Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- FTC CAN-SPAM Act compliance guide: https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business
- Twilio Message resource and status callbacks: https://www.twilio.com/docs/messaging/api/message-resource
- Resend send-email API reference: https://resend.com/docs/api-reference/emails/send-email
- Postmark email API and message streams: https://postmarkapp.com/developer/api/email-api
- Amazon SES verified identities and regions: https://docs.aws.amazon.com/ses/latest/dg/verify-addresses-and-domains.html
- PostgreSQL SELECT ... FOR UPDATE SKIP LOCKED: https://www.postgresql.org/docs/current/sql-select.html
- MDN, using server-sent events: https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events
