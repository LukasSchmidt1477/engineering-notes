# Password Reset Email API Governance: 3 Controls for 429 Rate Limits

**TL;DR:** Own the password-reset template in the application repository, compile it into an immutable message artifact, and retry that artifact after a 429 response. Do not render current copy again, and do not create a new reset token. This makes a short-expiry reset message reproducible even when a copy release and a delivery retry overlap.

| Template ownership model | What a retry sends | Change control | Operational fit |
| --- | --- | --- | --- |
| Repository-owned, compiled per delivery | The same stored artifact | Code review and deployment | Small B2B SaaS shipping weekly |
| Provider-hosted, revision pinned | A recorded remote revision | Provider access and publishing controls | Frequent copy edits outside engineering |
| Provider-hosted, mutable alias | Whatever the alias resolves to later | Live dashboard state | Low-risk messages where exact reproduction is unnecessary |

**Recommendation:** use the first model for a short-lived password reset. Preserve three things before the first send: the security challenge identifier, a rendered message artifact, and a content fingerprint. A 429 then changes `nextAttemptAt`, not the link, subject, body, or template version.

The choice costs some storage and makes copy releases part of application delivery. I accept that trade because account recovery is a bad place for invisible runtime edits. For a solo SaaS, the revenue-per-hour calculation favors a small, inspectable mechanism over investigating which version a delayed worker actually sent. Ship weekly. Keep the audit trail boring.

## What should happen after a password reset email API returns 429?

The race is easy to miss. At 10:00, a worker renders a five-minute reset link and the email transport answers with HTTP 429. At 10:01, a copy edit changes the template. At 10:02, the worker retries by invoking the renderer again. The security challenge is unchanged, yet the resulting message can have a different subject, expiry wording, support address, or HTML structure.

That is a governance failure, not merely a delivery failure. Logs that say "template: password-reset" cannot establish what the user received when that name points to mutable content. Re-rendering also widens the work performed during recovery: the retry now depends on the current template engine, current localization data, and current application state.

Freeze the output earlier.

Copy drift is silent.

The stored artifact should contain the exact subject, text body, and HTML body approved for this delivery. Keep the reset bearer value out of general logs and metrics. The challenge record remains the authority for expiry and consumption; the delivery record holds only what the worker needs to submit the already-authorized message. A content fingerprint ties those bytes to an audit event without requiring operators to print sensitive content.

This boundary also answers the repeat-request question. A browser request during the product's cooldown is not a transport retry. It follows the public recovery policy and should not bypass it merely because a downstream service was busy. OWASP recommends consistent responses and timing for existing and nonexistent accounts, plus protections against excessive automated submissions. Those endpoint rules remain separate from the private worker's scheduling decision.

## The 2 criteria that decide template ownership

The first criterion is **reproducibility under delay**. Given a delivery identifier, can an operator determine the exact non-secret content submitted on every attempt? Repository ownership works when rendering occurs once and the resulting artifact is stored. A hosted template can also work, but only if the delivery record pins an immutable revision and the transport contract guarantees that revision remains addressable. A mutable template name is insufficient evidence.

The second criterion is **the copy-change blast radius**. A spelling fix should affect newly created deliveries, not messages already waiting behind a throttle. Repository review makes that boundary visible in a commit and deployment. Hosted editing can provide a faster path for a non-engineer, but access control, preview, approval, version retention, and rollback then live outside the repository. Someone still owns those controls.

This is the practical split: outsource transport because transport is undifferentiated; retain the content contract that carries a security action. It does not mean building an email delivery network. It means deciding where an approved reset message becomes immutable.

The `Retry-After` field adds a separate constraint. HTTP semantics allow it to carry either a delay in seconds or an HTTP date. A sender should honor a valid value, apply bounded backoff when it is absent or unusable, and stop scheduling when the reset challenge will no longer be valid. For a Node.js transactional email worker, the useful best practice is to normalize that rate-limit response at the adapter, then let application policy decide if a retry can still beat expiry. The same rule applies to a resend triggered after the cooldown. The precise fallback delay is local policy, so it belongs in configuration and tests rather than an article's supposedly universal constant.

## Store the artifact before transport

The focused TypeScript below models the publishing boundary. Rendering produces one artifact. The fingerprint covers the delivery-relevant content, and each retry passes the same artifact to the transport adapter.

```ts
import { createHash } from "node:crypto";

type ResetCopy = {
  subject: string;
  text: string;
  html: string;
};

type MessageArtifact = ResetCopy & {
  deliveryId: string;
  challengeId: string;
  recipient: string;
  expiresAt: Date;
  contentSha256: string;
};

type SendResult =
  | { kind: "accepted"; receiptId: string }
  | { kind: "throttled"; retryAt?: Date }
  | { kind: "rejected"; reason: string };

interface TransactionalTransport {
  send(artifact: MessageArtifact): Promise<SendResult>;
}

function fingerprint(copy: ResetCopy): string {
  return createHash("sha256")
    .update(JSON.stringify([copy.subject, copy.text, copy.html]))
    .digest("hex");
}

function buildArtifact(input: {
  deliveryId: string;
  challengeId: string;
  recipient: string;
  resetUrl: string;
  expiresAt: Date;
}): MessageArtifact {
  const copy: ResetCopy = {
    subject: "Reset your password",
    text: `Reset your password: ${input.resetUrl}`,
    html: `<p><a href="${input.resetUrl}">Reset your password</a></p>`,
  };

  return {
    ...input,
    ...copy,
    contentSha256: fingerprint(copy),
  };
}

async function attemptDelivery(
  artifact: MessageArtifact,
  transport: TransactionalTransport,
  now: Date,
): Promise<SendResult | { kind: "expired" }> {
  if (now >= artifact.expiresAt) return { kind: "expired" };
  return transport.send(artifact);
}
```

The database write matters more than the interface polish. Persist the artifact and an outbox record in the same application transaction that authorizes the delivery. A worker claims a due outbox row with concurrency control, submits it once per claim, and records the normalized result. If the result is throttled, scheduling code computes the next eligible attempt from the transport response and local backoff policy. It must compare that time with `expiresAt` before committing another attempt.

Do not use the content hash as a secret-management scheme. A reset URL can have low-entropy surrounding text and must be protected as a bearer credential. Encrypt sensitive payloads at rest according to the application's threat model, restrict access, and delete them when retention is no longer justified. The fingerprint is useful for equality and audit correlation; it does not make the artifact safe to expose.

Tests should cross release boundaries, not just exercise the happy path. Create an artifact with template revision A, simulate a 429, deploy revision B, then assert that the retry still submits A's fingerprint and bytes. Run two workers against one due row and confirm that the claim mechanism prevents concurrent submission. Move the clock to the expiry boundary and verify that no transport call occurs. Finally, issue repeated public requests inside the cooldown and confirm that the response does not reveal account existence or worker state.

That suite has more value than a large mock of a provider SDK. It protects the part the application owns.

## When the runner-up earns the job

Provider-hosted templates are the better choice when copy must change independently of application releases and the organization already operates real publishing controls. “Real” means a preview of both text and HTML, named immutable revisions, scoped editor access, approval history, and rollback. Record the chosen revision beside the delivery before its first attempt. Retry by revision, never by a mutable friendly name.

Limitation: repository-owned artifacts are a poor fit when engineers cannot be the publishing gate for urgent copy or regulatory changes. They also add encrypted payload storage, retention work, and another schema to migrate. In that setting, choose a hosted, immutable template revision with audited publishing controls; the mutable-alias model still cannot reproduce a delayed security message.

There is another legitimate case: identity recovery is delegated as a whole. If an external identity system owns challenge creation, expiry, consumption, template publishing, and delivery, splitting template control back into the SaaS may add coordination without adding useful authority. The boundary should encompass the complete recovery lifecycle. Partial ownership, where one system changes expiry while another renders expiry claims, creates two sources of truth.

Mutable aliases remain suitable for messages whose exact historical copy does not matter.

A password reset with a short deadline is not such a message.

The decision rule stays narrow: freeze security-message content before the first transport attempt. Treat a 429 as scheduling information for that frozen artifact. This keeps copy governance independent from transport availability and lets a one-person team spend release time on product work rather than reconstructing dashboard state.

## References

- https://www.rfc-editor.org/rfc/rfc9110.html#name-429-too-many-requests
- https://www.rfc-editor.org/rfc/rfc9110.html#field.retry-after
- https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- https://datatracker.ietf.org/doc/html/rfc8058
- https://developer.mozilla.org/en-US/docs/Web/API/WebOTP_API
