# Hanging fetch and Axios Calls: Trace a Delayed Transactional Password Reset

Short answer: give the password-reset request a short application deadline, enqueue one durable delivery job, and diagnose a delayed transactional email from that job instead of making the browser wait on `fetch` or Axios.

The constraint that changes the design is the reset link's short useful life. A message that arrives after expiry has no value, while a second message can confuse the customer and invalidate the link they opened first. Delivery reliability therefore starts before the email provider call: the application needs one reset operation, one expiry, and one controlled path from acceptance to delivery.

For a one-person SaaS, this is a revenue-per-hour choice. I want to ship weekly, so I outsource the undifferentiated transport work but keep the recovery state in my own database. The web request stays dull. The worker gets the complexity.

## Trace one reset across three clocks

First, separate three clocks that dashboards often collapse into one. The browser has a response budget. The worker has an outbound API deadline. The reset credential has an expiry. Extending the first clock because the second one is slow gives the customer a worse page without making the message more likely to arrive. Extending the credential expiry to hide delivery lag changes the security decision. Those are separate controls.

A timeout also answers a narrow question: this process stopped waiting. It does not establish the final delivery outcome. The remote system may have accepted work before the connection ended, or the request may never have reached it. Treating every timeout as a clean failure creates duplicate sends; treating it as success hides missing mail. The honest state is `unknown` until the attempt is reconciled or its reset credential expires. `fetch` and Axios differ in configuration, but the useful contract is the same. The transport adapter must accept a cancellation signal, return a provider message identifier only after a confirmed acceptance, and distinguish a normal rejection from an elapsed local deadline. Keep that adapter below the job boundary. The browser handler should create the reset operation and enqueue its job, then return the same neutral response regardless of whether an account exists. It should never expose transport timing as an account-existence clue. Consider an illustrative budget: the page response is allowed 250 ms, the worker's outbound attempt is allowed 4,000 ms, and the reset credential expires after the product's chosen short window. These aren't universal targets. Your mileage may vary. The useful part is their independence. If an outbound call reaches 4,000 ms, the worker records `unknown`; it does not hold the page open for another 4,000 ms, and it does not mint a fresh reset operation. A later status observation can attach the provider message ID to the original attempt.

No resend yet.

Log boundaries, not secrets. An internal operation ID, attempt number, queue age, elapsed milliseconds, result class, and provider message ID are enough to trace most delays. A reset token and a raw email address do not belong in ordinary logs. I'm not sure one alert threshold fits every product; the missing input is the remaining useful lifetime when a job starts. Alert on the share of jobs consuming too much of that lifetime, not on a fashionable round number.

## How can Node.js fetch and Axios isolate a hanging password reset API request?

The following TypeScript keeps commercial API details behind a port. The HTTP handler can acknowledge the request after `jobs.enqueue` commits. A worker calls `deliver`, and the attempt store preserves the difference between accepted, rejected, and unknown. The adapter can be implemented with `fetch` or Axios without leaking either client through the application.

```ts
type ResetJob = {
  operationId: string;
  recipientRef: string;
  resetUrl: string;
  expiresAt: string;
};

type DeliveryResult =
  | { kind: "accepted"; messageId: string }
  | { kind: "rejected"; reason: string };

interface DeliveryPort {
  sendPasswordReset(job: ResetJob, signal: AbortSignal): Promise<DeliveryResult>;
}

interface AttemptStore {
  beginOnce(operationId: string): Promise<boolean>;
  markAccepted(operationId: string, messageId: string): Promise<void>;
  markRejected(operationId: string, reason: string): Promise<void>;
  markUnknown(operationId: string): Promise<void>;
}

async function deliver(
  job: ResetJob,
  delivery: DeliveryPort,
  attempts: AttemptStore,
  deadlineMs = 4_000,
): Promise<void> {
  const ownsAttempt = await attempts.beginOnce(job.operationId);
  if (!ownsAttempt) return;

  const controller = new AbortController();
  const timer = setTimeout(() => controller.abort(), deadlineMs);

  try {
    const result = await delivery.sendPasswordReset(job, controller.signal);

    if (result.kind === "accepted") {
      await attempts.markAccepted(job.operationId, result.messageId);
      return;
    }

    await attempts.markRejected(job.operationId, result.reason);
  } catch (error: unknown) {
    if (error instanceof Error && error.name === "AbortError") {
      await attempts.markUnknown(job.operationId);
      return;
    }
    throw error;
  } finally {
    clearTimeout(timer);
  }
}
```

`beginOnce` is the important line. It needs a durable uniqueness rule on `operationId`, not an in-memory flag. Queue redelivery, two browser clicks, and a process restart must converge on the same logical reset operation. The delivery adapter should also use the transport's documented deduplication facility when one exists, but application uniqueness remains necessary because a provider cannot infer that two newly generated payloads represent the same recovery action.

There is a sharp edge in the example: a non-timeout network exception is rethrown so the queue can apply its bounded retry policy. The queue must cap attempts and stop once `expiresAt` passes. Don't retry forever. A normal rejection should be stored with a sanitized reason and reviewed according to its class; it should not be converted into a timeout just because both cases prevent an immediate acceptance.

## Read the attempt ledger from left to right

Start at the application and move outward. Was exactly one reset operation committed? Did a job enter the queue? How old was it when a worker claimed it? Did the adapter obtain a confirmed acceptance and message ID before its deadline? Only after those questions should provider-side delivery events enter the investigation. This order prevents a queue backlog from being mislabeled as slow email transport.

A compact attempt record can carry `operation_id`, `created_at`, `expires_at`, `claimed_at`, `outbound_started_at`, `outbound_finished_at`, `state`, and `message_id`. Derived durations reveal the boundary at fault: creation to claim is queue delay; outbound start to finish is API time; confirmed acceptance to a later delivery event is downstream delivery time. Keep raw event payloads out of the hot request path. Store only what the support and audit workflow actually uses.

The failure branches need different actions:

- `queued`: add worker capacity or reduce unrelated work sharing that queue.
- `accepted`: do not send again merely because the customer refreshes the page; follow the documented event trail for the same message.
- `rejected`: fix the classified request or policy issue before another bounded attempt.
- `unknown`: reconcile the existing attempt when the chosen transport exposes a lookup mechanism; otherwise wait for the retry policy's controlled decision.
- `expired`: stop delivery and require a new recovery operation with a new credential.

Short expiry makes stale work visible. Before every outbound call, compare the job's expiry with the current time. If too little useful lifetime remains, end the job rather than sending a link that is likely to be dead on arrival. The precise cutoff is a product decision because the available evidence here does not establish a universal value.

Expiry wins.

Do not let a support agent solve uncertainty by repeatedly pressing send. Give support a view keyed by the internal operation ID: current state, age, attempt count, and a redacted message identifier. This turns “email is late” into a location in the pipeline. It also keeps diagnosis away from the public endpoint, where detailed errors could reveal whether an address is registered.

## Test the expiry race before production

This boundary makes testing cheap. Use a fake `DeliveryPort` that accepts, rejects, waits for cancellation, or throws before returning. Then assert the stored state and the number of calls. Test duplicate delivery of the same `operationId`. Test a job that starts after expiry. Test cancellation one millisecond before a fake acceptance. Those cases matter more than a happy-path snapshot of an email template.

Run the same contract tests against each transport adapter in a non-production environment. The test is about observable application states, not a vendor dashboard: one accepted attempt stores one identifier, a rejection stays rejected, cancellation becomes unknown, and an expired job never calls the adapter.

## When should this queue design change at scale?

At higher volume, split fresh sends and uncertain-attempt reconciliation into separate queues. Give expiring reset work priority over bulk communications, measure queue age by workload, and set a finite retry budget. Keep templates, credential generation, and transport adapters separate so a provider change doesn't rewrite account recovery. This is the kind of undifferentiated plumbing I would rather hide behind a small port than revisit during every weekly release.

The catch is operational ownership. A queue improves the request path, but it adds durable state, workers, retries, and reconciliation. It is not suitable when the application has no dependable job runner or when the transport cannot provide enough evidence to resolve uncertain attempts. For a tiny internal tool with no queue, a synchronous call with a strict deadline may be the more honest starting point; accept the slower page and keep the state machine. For a workflow that requires immediate, provider-confirmed delivery before continuing, use a synchronous orchestration path and budget the user experience around that requirement.

SMS fallback is not a free second channel. Message length and encoding affect segmentation: GSM-7 and UCS-2 have different character limits, and concatenated messages reserve characters for headers. A reset message that fits one encoding can become multiple segments in another. Keep the fallback copy short, test its actual encoding, and apply the same operation-level duplicate controls rather than firing SMS whenever email feels slow.

One more boundary matters. One-click unsubscribe is standardized for list mail by RFC 8058, but a password-reset message is an account-recovery transaction, not a campaign. Do not mix campaign subscription state into the reset worker merely because both jobs send email. Separate queues and policy modules keep a marketing backlog or suppression workflow from silently changing recovery behavior.

The decision rule is small: acknowledge account recovery quickly, preserve one durable operation, and spend the short expiry budget where delivery can still help. A hanging client call becomes an observable worker state, not a frozen page and not permission to send twice.

## References

- [RFC 8058: One-Click Unsubscribe](https://datatracker.ietf.org/doc/html/rfc8058)
- [SMS character limits and segmentation](https://www.twilio.com/docs/glossary/what-sms-character-limit)
