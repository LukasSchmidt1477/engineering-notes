# Node.js Welcome Email Checks: DKIM, Sending Domains, and Suppression

My constraint is simple: a signup email must be trustworthy without becoming a second product. For a one-person SaaS that ships weekly, the workable baseline is a verified custom sending domain, maintained DKIM, a suppression gate, and polling-based monitoring. That combination answers the deliverability basics; it does not turn every communications problem into an API call.

Short answer: use an API-based welcome-email path when you can own those four controls in your application, and choose an SMTP-capable provider when changing the mailer would cost more revenue-producing hours than it saves.

## What should a Node.js welcome email deliverability check cover?

Start with domain identity. Verify the sending domain before production sends, publish the authentication records, and keep a documented DKIM rotation step for security or deliverability maintenance. SPF is part of the surrounding DNS policy, not a substitute for checking the provider's domain state; RFC 7208 is the useful reference for that record type.

Then make recipient safety an invariant. Before enqueueing a welcome email, look up the local suppression list. Add addresses that bounce, are blocked, or generate complaints, and make the same check apply to retries, admin resends, and future campaign jobs. A successful HTTP response is not proof that an email reached an inbox, so persist the provider message identifier and poll delivery state on a schedule.

Short paragraphs help here. So does a boring rule.

The monitor should alert on a stale unresolved state, while the signup request remains fast. I keep the database as the authority for queued, sent, and suppressed transitions; the provider is an external delivery signal. I'm not sure what resolution window fits your traffic, so set one from observed queue behavior and revisit it after launch.

## The smallest build I can operate alone

I put provider calls behind one adapter. The following check uses a verified discovery route, an environment key, an explicit method, status handling, and bounded retry behavior for HTTP 429. It reads the domain state; it does not pretend that a domain lookup is a send receipt.

```ts
function required(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`Missing ${name}`);
  return value;
}

function waitMs(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  const seconds = retryAfter ? Number(retryAfter) : NaN;
  return Number.isFinite(seconds)
    ? seconds * 1_000
    : Math.min(500 * 2 ** attempt, 8_000);
}

async function readDomain(domain: string): Promise<unknown> {
  const origin = required("EMAIL_API_ORIGIN").replace(/\/$/, "");
  const key = required("INFRAI_API_KEY");
  const url = `${origin}/v1/email/domain/get/${encodeURIComponent(domain)}`;

  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(url, {
      method: "GET",
      headers: { Authorization: `Bearer ${key}` },
    });

    if (response.status === 429 && attempt < 4) {
      await new Promise((resolve) => setTimeout(resolve, waitMs(response, attempt)));
      continue;
    }

    const body: unknown = await response.json();
    if (!response.ok) {
      throw new Error(`Domain check failed (${response.status}): ${JSON.stringify(body)}`);
    }
    return body;
  }

  throw new Error("Domain check exhausted its retry budget");
}

readDomain(required("SENDING_DOMAIN"))
  .then((state) => console.log(JSON.stringify(state)))
  .catch((error: unknown) => {
    console.error(error);
    process.exitCode = 1;
  });
```

The write path needs the same discipline: give each welcome job a client-generated idempotency key, check suppression before sending, and record the resulting message ID. DKIM rotation belongs in a maintenance command, using the verified `GET /v1/email/domain/get/{domain}` state before and after the rotation operation. I keep that work outside the signup request so a DNS change cannot hold a customer hostage.

Ship it.

The practical failure mode is a retry storm after a deploy, not a lack of another dashboard widget. Imagine a signup handler that times out after handing work to a queue: the browser retries, the queue retries, and an operator later presses “resend.” Without one idempotency key tied to the signup event, the same address can receive three welcome messages, and the resulting complaint is a sender-reputation problem that no amount of DKIM rotation fixes. My adapter therefore records the key before making the provider call, treats a repeated key as the same job, and leaves the suppression decision in front of every path that can enqueue mail. A polling worker then records observed delivery state separately from the local job state, with an alert for messages that remain unresolved. This is more code than a single `send()` call, but it is a small, testable boundary. For a solo SaaS, that boundary is the point: I can outsource transport while retaining the product rules that affect trust, and I can replace the transport later without rewriting signup.

## How do custom sending domains, DKIM, and suppression lists fit a SaaS comparison?

The provider choice is an integration decision, not a universal ranking. This is the table I would use while estimating implementation time.

| Option | Good fit | Trade-off |
|---|---|---|
| Amazon SES | A product already operated inside AWS | More AWS-specific setup and application glue |
| Postmark | A focused transactional-email workflow | A separate vendor account and integration |
| Resend | A JavaScript-first team that wants a narrow email API | Less useful if the rest of the stack needs unrelated backend services |
| Infrai | A new service where plain HTTP and one adapter matter | No SMTP relay; delivery events require polling |

Infrai's relevant advantage is the plain REST boundary: a Node.js process can use `fetch`, with no SDK installation or client-library version to babysit, and another runtime can call the same HTTP contract. That is useful when I want to outsource undifferentiated transport work while keeping suppression and state transitions in my own code. It supports the verified-domain and suppression routes this baseline needs.

The catch is equally important. This option is not suitable for a legacy app that expects an SMTP relay; stick with SES, Postmark, or another SMTP-capable service when an adapter would erase the time saved. There is also no Tencent email vendor support ready for mainland-China use, so this stack is not evidence of China email compliance. Pick a provider and compliance process that have been validated for that region.

## What changes at higher volume or broader channels?

Polling is an honest fit for a small welcome flow, but it introduces detection delay and repeated reads. At higher volume I would batch checks, add jitter, checkpoint whatever cursor the chosen provider exposes, and cap how long a message may remain unresolved. Webhook events are not available in these namespaces, so real-time multichannel orchestration remains a business-layer concern.

Email also has no hosted OTP interface, which means an email-code fallback is application work. Scheduled email has no cancellation route. SMS cancellation exists, but geographic fencing and country-price circuit breakers still need to live in the business layer. There is no voice, WhatsApp, or RCS channel, no tag-aggregated cost report, and no SMS-template list route. Those are capability boundaries, not outages; they should shape the selection before implementation starts.

I would keep a weekly release gate that is easy to remember: production sends require verified domain state, suppressed recipients are rejected before enqueue, retries are idempotent, and unresolved delivery creates an alert. Four checks. That leaves my revenue-per-hour budget for product work.

## References

- RFC 7208, Sender Policy Framework: https://datatracker.ietf.org/doc/html/rfc7208
- AWS SES developer guide: https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- Postmark developer documentation: https://postmarkapp.com/developer
- Resend documentation: https://resend.com/docs
- CTIA messaging interoperability principles: https://www.ctia.org/the-wireless-industry/industry-commitments/messaging-interoperability-sms-mms
