# Media Signup Verification: NestJS Two-Factor SMS OTP Backend Replay Control

The operational constraint is simple: a media signup link has to survive an ordinary inbox delay without becoming a reusable credential. In a NestJS backend, I would keep two-factor authentication state separate from an SMS OTP fallback, issue one opaque single-use token, and make every attempt produce an audit event. The delivery channel can change later. The state machine should not.

That choice keeps the implementation small enough to ship this week. It also gives support a useful answer when a new subscriber says, “I clicked the link, so why am I still unverified?” The answer should come from an audit log, not a guess about the email provider.

Ship the boring path.

One token.

## How should NestJS two-factor authentication handle SMS OTP and links?

A verification link is a bearer credential during its short life. Treating it like a normal confirmation URL creates two separate problems: a link copied into a ticket can be replayed, and an expired link can be indistinguishable from a delivery failure.

SMS OTP is a different credential shape. It needs throttling by account, destination, and source address, plus an attempt counter and a short lockout policy. Recovery codes belong to account recovery, not to the normal signup path. A marketplace buyer verification flow may need those controls too, but this build is for media account signup and its delivery link.

I use a random token as the only value in the URL. The database stores a digest of that token, the account identifier, an expiry timestamp, and a consumed timestamp. It does not store the raw link. A request is accepted only when the digest matches, the record is unconsumed, and the current time is before expiry. Consumption happens in the same transaction as marking the account verified.

The 12-minute window is a policy value, not a protocol fact. It is long enough to tolerate a normal inbox delay and short enough to limit the life of a copied URL; changing it should be a reviewed decision tied to expiry and support data.

There is a useful distinction here. A second click after success is not necessarily an attack. It is often a subscriber refreshing a browser tab. The API can return a generic “already handled” result while the audit record preserves the reason. That keeps the user experience calm without weakening the one-use rule.

## Build log: the smallest working state machine

The account row needs a `verified_at` value. The verification table needs a unique token digest and an index that supports expiry cleanup. The audit table records events such as `issued`, `opened`, `consumed`, `expired`, and `rejected`, along with a request correlation ID. Do not put the token itself in logs, query strings copied into analytics, or exception messages.

The delivery worker receives a job containing an account ID and a verification ID. It loads the already-created message payload, sends it through the configured mail boundary, and records an attempt. A retry may send the same verification link again; it must not create a second valid token unless the user explicitly asks for a new link.

Here is the decision boundary in TypeScript. The repository methods stand in for a transaction-capable data layer, so the security property is visible without coupling the example to a framework.

```ts
type Verification = {
  id: string;
  accountId: string;
  tokenDigest: string;
  expiresAt: Date;
  consumedAt: Date | null;
};

type VerificationResult =
  | { kind: "verified"; accountId: string }
  | { kind: "already-handled" }
  | { kind: "invalid" };

interface VerificationStore {
  findByDigest(digest: string): Promise<Verification | null>;
  consumeIfOpen(id: string, now: Date): Promise<boolean>;
  markAccountVerified(accountId: string, now: Date): Promise<void>;
  audit(event: string, data: Record<string, string>): Promise<void>;
}

export async function consumeLink(
  rawToken: string,
  now: Date,
  store: VerificationStore,
  digest: (value: string) => string,
): Promise<VerificationResult> {
  const verification = await store.findByDigest(digest(rawToken));

  if (!verification || verification.expiresAt <= now) {
    await store.audit("rejected", { reason: "missing-or-expired" });
    return { kind: "invalid" };
  }

  const claimed = await store.consumeIfOpen(verification.id, now);
  if (!claimed) {
    await store.audit("rejected", { reason: "already-consumed" });
    return { kind: "already-handled" };
  }

  await store.markAccountVerified(verification.accountId, now);
  await store.audit("consumed", { verificationId: verification.id });
  return { kind: "verified", accountId: verification.accountId };
}
```

The important operation is `consumeIfOpen`. It must be an atomic conditional update, not a read followed by an update in application code. Two browser requests can arrive together. Only one should claim the row. I would test that race directly with parallel calls and assert one `verified` result plus one `already-handled` result.

The endpoint should not reveal whether a token belonged to an account. A generic invalid response limits account and token enumeration. Redirecting to a short status page after the mutation also avoids making the verification endpoint a place where clients retry a state-changing request.

This design is a poor fit for an offline-first product that cannot depend on a reachable inbox or phone. In that case, the product needs a separately reviewed recovery method; adding more retries to this endpoint would be the wrong fix.

## Delivery reliability is a state problem

The delivery provider is one hop. It is not the source of truth. The source of truth is the verification record and its audit history.

The backend should record the SMS OTP path separately from the link path. Mixing them makes a throttling alert look like a delivery incident and makes an audit log hard to interpret.

For each issue, I want a correlation ID that follows the enqueue, send attempt, provider response, and later click. The click cannot prove that the message was delivered, but it can prove that the token reached a browser. Those are different signals. Track them separately.

The useful operational counters are modest: issued links, send attempts, accepted provider responses, rejected tokens, expired tokens, and successful consumptions. Add latency from issue to consume. A rising expiry rate with normal provider acceptance points to message delay, confusing inbox placement, or a short validity window. A low consumption rate with normal click traffic may point to a redirect or cookie problem after the token was accepted.

Keep payloads out of logs. Keep phone numbers and email addresses out of metrics labels. A hashed account identifier can help join events, but a short-lived correlation ID is usually enough for support. Retention should follow the service’s data policy; the audit trail is not a reason to keep message content forever.

## Tests that catch the expensive mistakes

I would start with four tests, then add provider-specific tests only at the delivery boundary:

- An expired token is rejected and never changes the account.
- The same valid token used twice produces one account mutation.
- Two concurrent consume requests cannot both claim the token.
- A resend creates a new message attempt without silently invalidating a still-valid token, unless that revocation is an explicit product rule.

The last case deserves a decision. Revoking old links on resend reduces the number of live credentials, but it can strand a message that arrived late. Allowing several live links improves delivery tolerance, while each token remains single-use. For a small media service, I would allow a bounded number of active links and make the newest issuance visible in the audit record. The exact bound is a policy choice, not a universal security constant.

Recovery codes are also a trade-off. They help a verified account recover from a lost phone, but they are long-lived bearer secrets and need one-time consumption, hashing, and clear regeneration rules. They are not a reason to weaken SMS OTP throttling.

Run the tests with a fixed clock. Use a clock at the boundary: exactly at expiry, one millisecond before it, and one millisecond after it. Time bugs hide in defaults.

## What changes at scale?

At higher signup volume, I would separate the transactional issuance path from the delivery worker, add an idempotency key to the enqueue operation, and move cleanup to a scheduled job. The public consume path should remain boring. It reads one digest, claims one row, and redirects.

I would also make the status page useful without making it revealing. “This link is no longer valid” is enough. A signed support flow can issue a fresh link after a user proves control of the account through a separate, rate-limited action. Recovery is a product decision with security consequences, not an exception handler.

The revenue-per-hour test is practical here. I do not need a bespoke messaging abstraction before the first verified signup. I do need a durable state transition, an atomic claim, observable delivery attempts, and a runbook for expired links. Outsource the undifferentiated delivery hop when it is cheaper in engineering attention; keep the account decision and its evidence in the application boundary.

## Sources

- https://www.rfc-editor.org/rfc/rfc9106
- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html
- https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/303
- https://mustache.github.io/mustache.5.html
- https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview
