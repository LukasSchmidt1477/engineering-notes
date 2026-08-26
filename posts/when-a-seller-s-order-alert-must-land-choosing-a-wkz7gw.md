# When a Seller's Order Alert Must Land: Choosing a Transactional Email API Over SMTP Relay

Pick the transactional email API whose delivery outcome you can write back into your own database within a minute of the send. Everything else on the shortlist is negotiable. For a fintech marketplace telling a seller that a new order just landed, the absence of an SMTP relay is not a downgrade — the sender is application code you were going to write anyway.

| Starting point | What actually decides it | Verify this before committing |
| --- | --- | --- |
| New service, the alert leaves from your own order handler | How fast a delivery outcome gets back into your database | Event transport, replay window, retry semantics on your receiver |
| Legacy back office that already speaks SMTP | Cost of rewriting a mail path nobody owns | Whether the candidate keeps a relay at all |
| A failed alert must trigger a second channel | Minutes from bounce to your fallback decision | Whether bounce classification is machine-readable |
| Regulated sender, order amounts in the body | What the receiving domain will accept from your domain | Domain authentication and reporting, not vendor features |

That is a decision aid, not a ranking, and the middle column is where most of these comparisons go wrong. Developers shopping for SendGrid alternatives tend to start from a feature grid: templates, suppression lists, a sandbox mode, a dashboard somebody can screenshot. Every serious candidate has all of that. What separates them, once money is attached to the message, is how much of the delivery record ends up inside your own system, and how long it takes to get there.

## Should a marketplace service skip the SMTP relay and send order alerts through a transactional email API?

Yes, when the code that decides to notify the seller is code you own.

A relay exists to keep compatibility with software you cannot change: a CMS, a packaged accounting suite, a report generator that only knows host, port, and credentials. Your order handler is none of those. Configuring SMTP for one outbound message adds a second failure domain — TLS negotiation, connection pooling, a queue that isn't your queue — and buys you nothing you can point at in an incident review.

The gain from an HTTP-first sender is narrower than the pitch decks suggest, and it's worth saying plainly. You get a synchronous accept-or-reject on a request you can log with your own trace id, and you get a delivery record addressed by an identifier you chose rather than one the transport handed you. SMTP gives you the first through a 250 response and a queue id, and the second only if you go parsing log lines to find it.

The catch is that the work doesn't disappear. It relocates. Bounce handling, suppression, per-seller rate limits, retry budgets, the decision about what happens when a seller's mailbox is full during a settlement window — all of that becomes your application code. A relay lets you pretend otherwise for a while, because the mail server absorbs the retries silently and you never see the queue depth until someone complains.

A welcome email can be three hours late and nobody files a ticket. An order alert can't. That difference is the entire reason delivery reliability, and not the rate card, should drive this particular choice.

## Delivery reliability is a bookkeeping problem before it becomes a vendor problem

Accepted is not delivered. Every API on the market returns something like a 202 with a message id, and that response means one thing only: the request was well-formed and queued. The seller's mail server hasn't spoken yet. It may not speak for another forty seconds, or another six hours if the receiving side is doing greylisting.

So the architecture question is not which provider accepts faster. It's whether your system can answer, for any given order, the question "did the seller ever actually get told?"

The pattern that survives contact with production is boring: commit the order and an outbox row in the same transaction, let a worker claim the row and send it, and store the provider's message id back on that row. Now every notification has a local record with a lifecycle — pending, accepted, delivered, bounced, abandoned — and the provider is one column in it rather than the system of record. A dashboard that only your vendor can render is not observability. It's a support ticket with extra steps.

Two mechanisms decide whether that record stays honest under retries.

The first is idempotency. A send that times out on your side may or may not have been accepted on theirs, and retrying blindly is how sellers get four copies of the same order alert during a network blip. The IETF's Idempotency-Key header field draft describes the contract most senders converge on: the client picks a stable key, the server replays the original outcome instead of performing the action twice. Derive that key from your own order id, never from a timestamp or a random value generated inside the retry function.

The second is how delivery events get back to you. Push means an HTTPS endpoint you have to authenticate, monitor, and keep reachable — real operational weight for a one-person shop, and the thing that breaks first when you move regions. Polling means a scheduled read with a durable cursor and a bounded lag you control. I'm not sure there's a universal answer; the deciding question is what your product does with a bounce. If the answer is "flag the seller account and fall back to an in-app alert within the settlement window," a poll every 30 seconds is fine. If the answer is "cancel the payout hold," you want push, and you should verify the vendor's retry policy for your endpoint before you build against it.

Postmark, Amazon SES, and SendGrid all expose delivery events; the engineering differences that matter are the retry window on failed event deliveries, whether bounces arrive classified or as raw diagnostic text, and how long the event history stays queryable. Those three properties, not the feature matrix, are what you'll be reading at 2 a.m.

## Authentication comes first, and no API can paper over it

Provider choice cannot fix a domain that receiving servers don't trust. This is the part developers skip and then blame the vendor for.

SPF (RFC 7208) publishes which hosts may send for your domain, and it has a hard ceiling: no more than 10 DNS mechanism lookups during evaluation, after which the check returns permerror. Chain three vendors through nested include statements and you will hit it. DKIM (RFC 6376) signs the message so the receiver can verify it wasn't altered and that the signing domain vouches for it. DMARC (RFC 7489) ties those results to the domain in the From header through alignment, tells receivers what to do when alignment fails, and — the underrated half — asks them to send you aggregate reports.

Those reports are the only vendor-independent evidence you will ever have about your own mail. They come from the receiving side. Read them before you conclude that a provider is the reason your order alerts aren't landing.

Bounce classification runs on the same standards footing. RFC 3463 enhanced status codes let you distinguish 5.1.1 (mailbox doesn't exist, suppress permanently) from a 4.x transient condition (retry with backoff). A provider that surfaces the enhanced code is handing you a decision you can automate; one that hands back a prose diagnostic string is handing you a regex to maintain. And complaint feedback from mailbox providers arrives as ARF reports (RFC 5965), which for transactional mail usually signals that a seller has forgotten they signed up — worth routing to support rather than silently suppressing an account that is owed money.

## A minimal outbox worker and where the retry boundary sits

Here's the worker half of that outbox, in TypeScript, provider-agnostic on purpose. The signup path already wrote the row; this function only decides what happened to one attempt.

```ts
type OutboxRow = { id: string; sellerEmail: string; orderId: string; attempts: number };

type Outcome =
  | { kind: "sent"; providerId: string }
  | { kind: "retry"; waitMs: number }
  | { kind: "dead"; reason: string };

const endpoint = process.env.MAIL_ENDPOINT;
const token = process.env.MAIL_TOKEN;

export async function deliver(row: OutboxRow): Promise<Outcome> {
  if (!endpoint || !token) throw new Error("MAIL_ENDPOINT and MAIL_TOKEN must be set");

  const res = await fetch(endpoint, {
    method: "POST",
    headers: {
      authorization: `Bearer ${token}`,
      "content-type": "application/json",
      // Same value on every attempt, derived from the order — a timeout can
      // never mail the seller twice.
      "idempotency-key": `order-alert-${row.orderId}`,
    },
    body: JSON.stringify({
      to: row.sellerEmail,
      subject: `New order ${row.orderId}`,
      text: "A buyer placed an order. Settlement runs at 18:00 UTC.",
    }),
  });

  if (res.status === 429 || res.status >= 500) {
    const after = Number(res.headers.get("retry-after"));
    const waitMs = Number.isFinite(after) && after > 0
      ? after * 1000
      : Math.min(30_000, 500 * 2 ** row.attempts);
    return { kind: "retry", waitMs };
  }

  const body = await res.text();
  if (!res.ok) return { kind: "dead", reason: `${res.status} ${body.slice(0, 200)}` };

  return { kind: "sent", providerId: (JSON.parse(body) as { id: string }).id };
}
```

Three things in there are the whole point. The status is checked before the body is parsed, so a rejection never gets mistaken for a message object. Retry-After is honored when the server sends it and backoff caps at 30 seconds instead of climbing forever. And the caller — not this function — owns the state transition, which is what keeps the outbox row the source of truth.

Order `o_7741` makes it concrete. The transaction commits the order and an outbox row together, the worker claims the row under a lease, sends, and writes the returned id back. If the process dies mid-flight, the lease expires and another worker retries with the same idempotency key, so the seller still gets exactly one alert. When the delivery event for that provider id arrives — pushed or polled — it updates the row, and only then does the order count as notified. If the event never arrives, a query for rows accepted more than fifteen minutes ago with no terminal outcome finds it. That query is the real monitor. Alert thresholds on it are cheaper to maintain than a synthetic probe, and it keeps working when you change providers, because the schema belongs to you.

Keep the provider-shaped code in one adapter of about forty lines and resist wrapping every field in an abstraction you haven't needed yet. Portability comes from owning the record, not from a generic interface.

## When an SMTP relay is still the right answer

Stick with a relay when the sending code isn't yours. A packaged back office, a BI tool that emails scheduled reports, a self-hosted admin panel — rewriting those to speak HTTP is real work with no product outcome, and the relay is doing its job.

Stick with it, too, when the mail path has to run inside your own perimeter for data residency, or when the message body carries data you're not permitted to hand to a third party's template renderer. A self-hosted MTA plus a smarthost keeps that boundary crisp in a way an API-first vendor can't.

And if the underlying requirement turns out to be a second channel rather than better email, the comparison changes completely. An email API doesn't support SMS or push, and bolting on a separate messaging vendor reintroduces the credential sprawl you were trying to escape. Browser-side helpers don't close that gap either: the WebOTP API only autofills a one-time code from a specially formatted SMS on a bound origin — it doesn't send anything, and the server-side lifecycle of generating, storing, and expiring that code is still yours.

My test after all this is short. Can I answer "was the seller told?" from my own database, without logging into anyone else's dashboard? If yes, the choice is good enough — ship it and go build the thing customers are actually paying for.

## References

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://datatracker.ietf.org/doc/html/rfc7489)
- [RFC 7208: Sender Policy Framework (SPF) for Authorizing Use of Domains in Email](https://datatracker.ietf.org/doc/html/rfc7208)
- [RFC 6376: DomainKeys Identified Mail (DKIM) Signatures](https://datatracker.ietf.org/doc/html/rfc6376)
- [RFC 3463: Enhanced Mail System Status Codes](https://datatracker.ietf.org/doc/html/rfc3463)
- [RFC 5965: An Extensible Format for Email Feedback Reports (ARF)](https://datatracker.ietf.org/doc/html/rfc5965)
- [The Idempotency-Key HTTP Header Field (IETF draft)](https://datatracker.ietf.org/doc/draft-ietf-httpapi-idempotency-key-header/)
- [MDN: WebOTP API](https://developer.mozilla.org/en-US/docs/Web/API/WebOTP_API)
