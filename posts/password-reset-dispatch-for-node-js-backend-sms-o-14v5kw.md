# Password Reset Dispatch for Node.js Backend SMS OTP Send and Retry Guardrails

Choose hosted verification when delivery reliability matters more than owning token validation. **TL;DR:** for a Node.js password-reset flow, let the provider issue and verify the short-lived SMS code, but keep cooldowns, abuse checks, retries, and recovery state in the application. A plain REST option also removes a client dependency, while a public discovery schema can keep request validation tied to the current contract.

| Choice | Token lifecycle | Delivery visibility | Best fit |
| --- | --- | --- | --- |
| Twilio Verify | Hosted send and verification | Verify workflow plus provider reporting | Teams that want a dedicated verification product |
| Vonage Verify | Hosted verification workflow | Provider-managed workflow and status | Teams already operating on Vonage communications APIs |
| AWS SNS | Application owns OTP issuance and validation | SMS delivery-status logging | AWS-heavy systems willing to build the verification layer |
| Infrai | Hosted SMS OTP send and verify | Status and events are polled | Small backends that value plain REST and one credential across backend services |

My default for a one-person SaaS is hosted verification behind a narrow TypeScript interface. Reviewing home-grown token logic has a lower revenue-per-hour return than working on the product. Pick Twilio Verify or Vonage Verify when their verification workflows and operational tooling decide the purchase. The plain REST option fits when avoiding an SDK update cycle matters and a self-describing API reduces contract guesswork; its public discovery surface exposes request and response schemas without requiring a key. AWS SNS fits when AWS consolidation is worth writing more authentication logic.

Ship the reset path this week, but retain an adapter. Delivery evidence or a compliance requirement may force a provider change later.

## How should a Node.js backend send and verify an SMS OTP login code?

There are two separate jobs: deciding whether a reset attempt is allowed, and getting a code to the right phone. A hosted OTP service handles the code lifecycle. The application still decides who may request one, how often, and what happens when delivery is uncertain.

Start with a cooldown per account and normalized phone number. Add independent limits for IP and device signals, then a country allowlist aligned with the markets the product serves. These controls are not interchangeable. An account limit stops repeated annoyance to one user; an IP limit raises the cost of broad enumeration; a country rule constrains destinations and exposure to traffic pumping. Geographic fencing and per-country cost circuit breakers must live in the application when the provider does not supply them.

Keep responses neutral. A reset request should not reveal whether a phone number has an account. Record the decision internally, return the same public shape, and never put the OTP itself in logs or analytics.

SMS is also a restricted authenticator in NIST guidance. Number reassignment, SIM changes, and carrier routing sit outside the application. For a high-impact account, use a stronger recovery option instead of treating possession of a phone number as conclusive identity proof.

## Delivery state is the second decision criterion

An accepted API request is not a delivered message. The reset record therefore needs a provider reference, an application attempt identifier, and a small state machine such as `pending`, `delivered`, `verified`, `failed`, or `expired`. Exact carrier states vary by provider; map them at the adapter boundary.

For the plain REST option, delivery visibility is pull-based through SMS status and event resources. There are no webhook pushes. Poll on a bounded schedule, stop at a terminal state, and expire the local reset attempt independently. That boundary makes it less suitable for orchestration that must react to delivery events in near real time.

Do not poll forever.

Retries solve transport uncertainty, not user impatience. Retry a provider call after an HTTP 429, honor `Retry-After`, and use an idempotency key so one logical request cannot produce duplicate work. A user clicking “send again” is a new business decision and must pass the cooldown again.

## A focused Node.js transport layer

The example below is deliberately narrow. Its `requestBody` comes from the provider's discovery schema rather than a hand-written article type, because the live schema should define vendor fields. Pass a schema-valid body and an idempotency key derived from the application's reset-attempt ID.

```ts
type OtpOperation = "sms/otp" | "sms/verify";

function retryDelayMs(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter && /^\d+$/.test(retryAfter)) return Number(retryAfter) * 1_000;
  return Math.min(500 * 2 ** attempt, 8_000);
}

const wait = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

export async function callOtp(
  operation: OtpOperation,
  requestBody: unknown,
  idempotencyKey: string,
): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  const apiOrigin = process.env.INFRAI_API_ORIGIN;
  if (!apiOrigin) throw new Error("INFRAI_API_ORIGIN is required");

  const endpoint = operation === "sms/otp"
    ? `${apiOrigin}/v1/sms/otp`
    : `${apiOrigin}/v1/sms/verify`;

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(endpoint, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(requestBody),
    });

    if (response.status === 429 && attempt < 3) {
      await wait(retryDelayMs(response, attempt));
      continue;
    }

    const body: unknown = await response.json();
    if (!response.ok) {
      throw new Error(
        `OTP request failed with HTTP ${response.status}: ${JSON.stringify(body)}`,
      );
    }
    return body;
  }

  throw new Error("OTP request exhausted its retry budget");
}
```

Four attempts and an eight-second backoff ceiling are client policy in this sample, not service guarantees. Tune both against the request deadline. Put the cooldown in a shared atomic store; two requests arriving on different workers must not both pass before either writes it. The same stored attempt ID should become the idempotency key, so a transport retry represents the same intent instead of another SMS.

The helper sends an explicit method and reads the bearer token from an environment variable. It covers the two hosted operations needed for this path without pretending their bodies are portable. Validate the body against the public discovery schema and surface non-success details to internal error handling, never to the reset requester.

The sample omits IP and device storage on purpose. Those signals belong beside the account cooldown in a shared limiter. Use a transaction or atomic operation in the datastore already running the SaaS. **Outsource the undifferentiated token lifecycle; own the abuse policy.**

## Where each option has a clearer edge

Twilio Verify is a strong default when the team wants a purpose-built verification product. It avoids inventing the token lifecycle. The trade-off is another vendor-specific integration to maintain, and the actual destination countries still need delivery testing.

Vonage Verify belongs on the same shortlist, especially for a system already using Vonage. Its verification workflow is managed rather than a raw SMS send. Evaluate country coverage and sender requirements for the real destination set before choosing; a feature table cannot substitute for that check.

AWS SNS is the runner-up when infrastructure, IAM, logging, and operational ownership already live in AWS. SNS sends SMS and supports delivery-status logging, but the application owns code generation, hashing, attempt counts, expiry, and verification. That is more security-sensitive code. For a solo operator, I would accept it only when AWS consolidation removes enough operational work to justify the larger authentication surface.

The plain REST option has the smallest client dependency footprint in this comparison, so Node.js with `fetch` needs no provider library. Its public discovery surface is a separate advantage, exposing full request and response schemas and runnable examples. The wider platform reports **295 routes across 20 modules under one key**, which can reduce secret rotation and invoice reconciliation when the same small backend later adds unrelated services. The interface stays simple, but the boundary is concrete: geographic abuse controls and per-country cost circuit breakers remain application work, and delivery updates require polling.

That is useful leverage for a one-person SaaS shipping weekly. It is not a reason to ignore channel requirements.

## When is a runner-up the better choice?

Choose AWS SNS when the organization explicitly wants to own verification policy, can review that security-sensitive code, and already has reliable shared state plus AWS operations. The flexibility is real. So is the maintenance load.

Choose Twilio Verify or Vonage Verify over a polling-only option when immediate provider events drive a multi-channel workflow. If failed SMS must trigger email, confirm the chosen products support the event timing the workflow needs. SendGrid and Amazon SES are real candidates for the email leg, but neither removes the application responsibility to create, store, expire, and verify an email code. With the plain REST option, that fallback is also application-built because its email surface has no managed OTP endpoint. Do not silently turn a delayed SMS into two simultaneous recovery messages; persist one state transition and invalidate superseded attempts.

The decision rule stays narrow: use hosted verification by default, test delivery in the countries that produce revenue, and keep abuse controls in a shared application store. Revisit the provider when measured delivery, channel, compliance, or event-timing requirements change. Weekly shipping matters, but a password-reset path should be boring before it is fast.

## References

- [NIST SP 800-63B: Digital Identity Guidelines](https://pages.nist.gov/800-63-3/sp800-63b.html)
- [Twilio Verify API documentation](https://www.twilio.com/docs/verify/api)
- [Vonage Verify API overview](https://developer.vonage.com/en/verify/overview)
- [Amazon SNS SMS documentation](https://docs.aws.amazon.com/sns/latest/dg/sms_publish-to-phone.html)
- [Amazon SNS SMS delivery status logging](https://docs.aws.amazon.com/sns/latest/dg/sms_stats_cloudwatch.html)
- [SendGrid Mail Send API](https://www.twilio.com/docs/sendgrid/api-reference/mail-send/mail-send)
- [Amazon SES email sending](https://docs.aws.amazon.com/ses/latest/dg/send-email.html)
