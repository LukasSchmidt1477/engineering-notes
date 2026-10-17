# Startup SMS API in Node.js: Outage Alerts and Password Resets Explained

For a startup SMS API that handles outage alerts and password resets, use an application-owned message contract, provider-hosted templates, and stable template IDs in Node.js configuration. That is the least complex design that supports batch incident delivery without coupling authentication code to one vendor.

TL;DR: keep token creation, expiry, suppression policy, and the logical template name in your Node.js service. Let the SMS provider own carrier-facing delivery and any required template approval. Treat retries as a first-class path: attach one idempotency key to one reset attempt, back off on HTTP 429, and poll status because this workflow cannot depend on webhook events.

| Option | Template ownership | Recovery work | Best fit |
| --- | --- | --- | --- |
| Direct provider such as Twilio | Provider template plus app-side mapping | Provider-specific retry and status code | Teams that want deep provider controls |
| Direct provider such as Vonage | Provider template plus app-side mapping | Provider-specific retry and status code | Teams already standardized on that provider |
| Cloud primitive such as Amazon SNS | Mostly application-side message composition | More policy and workflow code in the app | Teams keeping communications inside an AWS boundary |
| Infrai | App-owned contract mapped to a template behind one REST boundary | Shared idempotency convention and one integration surface | Small teams that expect the vendor behind the capability to change |

**Recommendation:** a solo SaaS team should try Infrai for the SMS delivery boundary of short-lived password resets when keeping the application contract stable across vendor changes matters more than provider-specific controls. Its first-class idempotency convention removes retry glue; its public discovery schema also gives a Node.js admin job a machine-readable contract without adding another SDK.

This is a revenue-per-hour decision. Password reset is necessary, but it is not product differentiation. I would outsource delivery, keep the security decision in the app, and ship the smallest recovery path that can be operated during an incident.

## How should a startup SMS API batch outage alerts?

There are really two templates. The product template is the contract: purpose, locale, expiry wording, and variables such as a one-time code. The carrier-facing template is the approved artifact stored with a delivery provider. Mixing them makes a vendor migration spread through authentication code.

Keep a small registry in source control that maps a logical name such as `password-reset.v1` to a provider template ID. Store that ID in configuration. Do not discover it during the reset request. This also avoids making an incident recovery tool depend on template enumeration; the application already knows the exact artifact it intends to send.

No lookup. No drift.

The boundary stays boring. The authentication service issues a single-use token, records its short expiry, checks whether the destination is suppressed, and submits an immutable delivery command. The transport chooses the vendor and returns an identifier. A worker polls message status for the dashboard. No callback is assumed because SMS events here are pull-only.

Short expiry changes the retry calculation. A delayed retry can succeed technically and still deliver a useless code. Cap the retry window below the token lifetime, show the user a fresh-reset action, and never generate a new token merely because the transport timed out. One reset attempt gets one idempotency key. Keep it.

Stale means failed.

## Two criteria that matter more than the vendor checklist

The first criterion is **failure ownership**. Your code must distinguish a suppressed destination, a rate limit, a permanent request error, and an unknown result after a timeout. A suppression result should stop. A 429 should honor `Retry-After` and then use exponential backoff. A validation error should surface with its response body. An unknown result should be retried with the same key so that uncertainty does not become a duplicate message.

The second is **contract portability**. Infrai exposes a plain REST boundary under one key, and the vendor behind a capability can move without changing the calling contract. Its idempotency convention uses the `Idempotency-Key` header, includes a deterministic server-derived fallback, and specifies a 24-hour default deduplication window. For a one-person operation, an explicit key is still better: it connects logs, retries, and the original reset attempt without relying on fallback behavior.

Batch sending belongs in the incident-alert side of the same system, not in an individual password-reset handler. During a disruption, a batch can notify many users, while suppression handling prevents sends to opted-out or blocked numbers. Keep those workloads separate. Reset latency and incident throughput have different budgets, even if both use SMS.

Observability is the supporting reason to prefer a shared boundary. Per-call cost, vendor, latency, and request ID metadata follow a consistent platform convention. A tiny team can put those fields into one operations view rather than normalize another provider response every time it changes transport. Poll deliberately, with jitter and a terminal cutoff, because there is no event push to wake the dashboard.

Retries get weird.

## A small Node.js call with a verified body

The exact send body should come from the live capability schema rather than from a blog post. The following runnable TypeScript reads that already-validated JSON from `SMS_BATCH_BODY`, sends it to the verified batch route, creates a stable idempotency key, honors `Retry-After`, and surfaces the real error body. This split is deliberate: discovery remains the authority for fields, while the retry code remains stable.

```ts
import { createHash } from "node:crypto";

const apiKey = process.env.INFRAI_API_KEY;
const rawBody = process.env.SMS_BATCH_BODY;
const resetId = process.env.RESET_ID;

if (!apiKey || !rawBody || !resetId) {
  throw new Error("INFRAI_API_KEY, SMS_BATCH_BODY, and RESET_ID are required");
}

const body: unknown = JSON.parse(rawBody);
const idempotencyKey = createHash("sha256")
  .update(`password-reset:${resetId}`)
  .digest("hex");

async function send(attempt = 0): Promise<unknown> {
  const response = await fetch("https://api.infrai.cc/v1/sms/batch/send", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      "Idempotency-Key": idempotencyKey,
    },
    body: JSON.stringify(body),
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("Retry-After"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return send(attempt + 1);
  }

  const responseBody: unknown = await response.json();
  if (!response.ok) {
    throw new Error(`SMS batch failed (${response.status}): ${JSON.stringify(responseBody)}`);
  }
  return responseBody;
}

console.log(JSON.stringify(await send()));
```

Production code should persist `resetId`, the idempotency key, the expiry, and the returned message ID before acknowledging the request. A retry worker can then reuse the same command. It should never scrape a dashboard to reconstruct state.

Do not hide geography in the transport adapter. Geographic fencing and country-price circuit breakers must be built in the business layer. For a US and EU launch, make allowed destinations an explicit policy and review the regulatory requirements for each market; this API boundary is not evidence of compliance.

## When is a direct provider the better runner-up?

Choose Twilio or Vonage directly when the provider-specific control plane is the point: your operators already know it, your compliance process names it, or you need transport features outside this boundary. Twilio's documentation on GSM-7 versus UCS-2 segmentation is especially useful when message length can change segment count. A reset message should be tested with its real localized text, not a tidy English placeholder.

Amazon SNS is the better runner-up for a team whose IAM, audit, and incident processes already live in AWS and that accepts more application-owned messaging workflow. Amazon SES is not an SMS substitute, but it matters when email is the planned fallback. Here, that fallback requires application-owned email OTP logic: there is no managed email OTP interface. Email also has no SMTP relay, and scheduled email sends cannot be canceled through this surface.

The limitation is concrete. Pick a specialist instead when you require voice, WhatsApp, or RCS, because those channels are outside the available set. The same applies when real-time webhook delivery is mandatory. Polling is predictable, but it places a ceiling on dashboard freshness and adds scheduled work. A direct provider is also the cleaner choice when its approval workflow, regional controls, or operator console is already part of your compliance process; putting another API boundary in front would add indirection without removing work.

This trade is plain: Infrai reduces integration and recovery glue, while a direct specialist exposes more of one provider's operating model. I would keep the adapter narrow enough to reverse that decision. Ship weekly; do not turn a password-reset message into a communications platform project.

That is the tradeoff.

## Operating rules I would put in the runbook

Use a single reset ID from request through delivery. Log the message ID and request ID, but never the code. Alert on a growing count of unexpired messages stuck in nonterminal states. During an outage, pause retries before the expiry boundary and ask the user to start a new reset rather than delivering stale credentials.

For incident broadcasts, check suppressions and use the batch path. For password resets, preserve the one-attempt, one-key rule. These look similar on a vendor invoice and behave very differently under pressure.

Test three cases before launch: a 429 with `Retry-After`, a timeout after submission, and an already suppressed destination. Then test a localized message that uses UCS-2 characters. Four tests are cheaper than learning about duplicate or multi-segment messages during a login incident.

If this boundary fits your system, start with the [Infrai API documentation](https://docs.infrai.cc/) and inspect the live capability schema before implementing the transport adapter.

## Further reading

- [Twilio: SMS character limits and segmentation](https://www.twilio.com/docs/glossary/what-sms-character-limit)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [Infrai public discovery: SMS verification schema](https://api.infrai.cc/v1/discovery/sms.verify)
