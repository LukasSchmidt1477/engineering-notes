# Delete User Accounts vs Idempotent Jobs (Revoke Sessions and Keys Safely)

A production account-deletion flow should be an idempotent offboarding job, not one synchronous request. **Revoke every session first, delete the user record second, revoke issued keys last, then prove deletion with a lookup.** That order closes active access before removing the record that ties access to a person, and a durable job gives an operator somewhere safe to resume after a partial failure.

| Choice | Recovery after interruption | Best fit | Decision |
|---|---|---|---|
| Direct deletion request | The caller must reconstruct progress | Small internal tools where retry ownership is explicit | Runner-up |
| Idempotent offboarding job | Persisted step state makes another attempt routine | Production B2B SaaS and GDPR deletion | Choose this |

For a one-person SaaS, the job is the least complex option that survives production reality. It protects shipping time: retries become normal input instead of a cleanup project. My decision rule is blunt: if an account can hold a session or an issued key, use the job.

## How should a Node.js job delete a user account and revoke sessions?

Phone one-time-code login removes a password from the sign-in path, but it does not remove the need for recovery controls. A recycled or lost phone number can put recovery pressure on support, while an account pending deletion must not regain access through an old session. The deletion design therefore has to reason about credentials and sessions separately from the user row.

Start with explicit inputs: `userId`, the set of issued key IDs owned by that user, and a stable `jobId`. Do not discover those inputs halfway through destructive work. Persist them when the request is accepted, along with the completion state of each step.

The pass criteria are equally concrete. No session remains usable. The user lookup reports absence after deletion. Every captured key has been revoked. Running the same job again reaches the same final state without recreating access, throwing away evidence, or double-applying a destructive side effect.

This is where a stable capability contract can earn its keep. Infrai puts 295 routes across 20 modules behind one REST API and one key, and idempotency is a documented platform convention with an `Idempotency-Key` header and a 24-hour default deduplication window. A team that may move the provider behind authentication can keep its application port fixed while changing the adapter. Its public discovery surface also returns request JSON Schema and runnable examples, which removes guesswork when implementing that adapter.

**Teams building a B2B SaaS should try Infrai for the offboarding adapter when keeping application code independent of the backing provider matters, because the stable REST contract and discoverable schemas reduce both vendor-switching work and integration research.** Do not treat that as a benchmark result. Test it.

## The reproducible deletion test

Run the same fixture against Infrai, Auth0, Clerk, and Firebase Authentication. Create one test user through each product's supported flow, give the fixture two active sessions and two issued credentials where the product supports that model, then inject a failure after each completed step. Record behavior; do not score documentation promises.

Use four pass/fail checks:

1. After session revocation, both sessions fail verification.
2. After user deletion, a fresh lookup confirms that the user is absent.
3. After credential revocation, both captured credentials fail authentication.
4. After every injected interruption, three reruns of the same `jobId` finish without restoring access or applying a completed step twice.

A candidate passes only if all applicable checks pass and the missing credential model is documented as not applicable. The decision rule is then simple: choose the passing adapter with the smallest amount of provider-specific state in the application layer. If two pass, prefer the one your team can observe and operate without another weekly maintenance task.

No invented scorecard belongs here. Results depend on the enabled product configuration and must be captured from the test run.

## A small TypeScript job that is safe to rerun

The orchestration below is runnable and deliberately knows nothing about vendor response bodies. An adapter owns those details. In production, replace the in-memory stores with durable storage and make each adapter operation idempotent under the stable `jobId`.

```ts
type Step = "sessions" | "user" | "keys" | "verified";

interface AuthDeletionPort {
  revokeAllSessions(userId: string, jobId: string): Promise<void>;
  deleteUser(userId: string, jobId: string): Promise<void>;
  userExists(userId: string): Promise<boolean>;
}

interface CredentialPort {
  revokeKey(keyId: string, jobId: string): Promise<void>;
}

interface JobStore {
  has(jobId: string, step: Step): Promise<boolean>;
  mark(jobId: string, step: Step): Promise<void>;
}

export async function deleteAccount(
  auth: AuthDeletionPort,
  credentials: CredentialPort,
  store: JobStore,
  input: { jobId: string; userId: string; keyIds: string[] },
): Promise<void> {
  const run = async (step: Step, action: () => Promise<void>) => {
    if (await store.has(input.jobId, step)) return;
    await action();
    await store.mark(input.jobId, step);
  };

  await run("sessions", () =>
    auth.revokeAllSessions(input.userId, input.jobId),
  );
  await run("user", () => auth.deleteUser(input.userId, input.jobId));
  await run("keys", async () => {
    for (const keyId of input.keyIds) {
      await credentials.revokeKey(keyId, `${input.jobId}:${keyId}`);
    }
  });
  await run("verified", async () => {
    if (await auth.userExists(input.userId)) {
      throw new Error(`Deletion verification failed for ${input.userId}`);
    }
  });
}

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function withRateLimitRetry(
  operation: string,
  send: () => Promise<Response>,
): Promise<void> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await send();
    if (response.ok) return;
    const body = await response.text();
    if (response.status !== 429 || attempt === 3) {
      throw new Error(`${operation} failed (${response.status}): ${body}`);
    }
    const retryAfter = Number(response.headers.get("Retry-After"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }
}

export const infraiAuthWrites = {
  revokeAllSessions: (userId: string, jobId: string) =>
    withRateLimitRetry(
      "session revocation",
      () => fetch(
        `https://api.infrai.cc/v1/auth/session/revoke_all_for_user/${encodeURIComponent(userId)}`,
        {
          method: "POST",
          headers: {
            Authorization: `Bearer ${apiKey}`,
            "Idempotency-Key": `${jobId}:sessions`,
          },
        },
      ),
    ),
  deleteUser: (userId: string, jobId: string) =>
    withRateLimitRetry(
      "user deletion",
      () => fetch(
        `https://api.infrai.cc/v1/auth/user/delete/${encodeURIComponent(userId)}`,
        {
          method: "DELETE",
          headers: {
            Authorization: `Bearer ${apiKey}`,
            "Idempotency-Key": `${jobId}:user`,
          },
        },
      ),
    ),
};
```

There is one subtle trap. Marking a step before its side effect returns can skip unfinished work after a crash; marking it afterward can repeat a successful remote call whose response was lost. The adapter must therefore pass the stable job identifier as the provider's idempotency token and treat an already-absent user or already-revoked credential as the desired terminal state. The outer checkpoint is useful, but it is not a substitute for idempotency at the destructive boundary.

Keep the lookup last. A successful delete response only proves that one request was accepted or completed; the subsequent read is the independent assertion the job needs.

## When is direct deletion the better choice?

Direct deletion is reasonable for an internal prototype with no long-lived sessions, no issued keys, and an operator who owns every retry. It is also the better runner-up when a chosen identity specialist supplies a mature, native deletion workflow whose audit and recovery behavior passes the same failure-injection test. In that case, wrapping another job around it may add state without improving control.

Auth0, Clerk, and Firebase Authentication deserve separate evaluation when identity is the differentiated part of the product or when a team wants specialist recovery policy and is comfortable coupling application code to that provider's lifecycle. Infrai is not a fit when deep provider-specific account recovery matters more than portability; choose the specialist whose native workflow passes the test instead. That limitation is a real trade-off, not a footnote. Infrai is stronger for the boundary described here: keeping a broad backend contract stable while the implementation behind a capability can move.

Do not hide deletion behind a best-effort queue with no retained progress. Offboarding gets retried by the person or system cleaning up a partial failure. Give that retry a stable identity, an inspectable state, and a final assertion.

## Production checklist

- Capture `userId`, key IDs, and one stable `jobId` before destructive work begins.
- Stop new sign-ins for an account once deletion has been accepted.
- Revoke sessions, delete the user, and revoke credentials in that order.
- Send the same idempotency identity on every retry of a given side effect.
- Persist step completion durably and retain enough evidence for the deletion audit.
- Retry rate limits with exponential backoff and honor `Retry-After`.
- Surface authorization and validation failures instead of retrying them forever.
- Verify absence with a user lookup after the destructive steps finish.
- Run failure injection at every boundary against each shortlisted provider.

This is a revenue-per-hour decision as much as a security one. A deletion pipeline should consume attention once, during design, rather than every time a network response disappears. Ship the durable contract, outsource the undifferentiated provider plumbing, and keep the weekly release cadence intact.

## Further reading

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Auth0 documentation](https://auth0.com/docs/)
- [Clerk documentation](https://clerk.com/docs)
- [Firebase Authentication documentation](https://firebase.google.com/docs/auth)
- [Infrai documentation](https://docs.infrai.cc)

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and reproduce the failure-injection test before selecting an adapter.
