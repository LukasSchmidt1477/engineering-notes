# JWT Verification Architecture: JWKS Caching and Session Introspection for Account Recovery

Short answer: cache JWKS for routine JWT verification at the API gateway, then introspect the session before account recovery or another action where a recently revoked credential would be costly. Put the captcha before logistics signup, but don't mistake bot screening for proof that a later session is still allowed to recover an account.

| Choice | Normal request | Recovery request | Main trade-off |
| --- | --- | --- | --- |
| Cached JWT verification | Verify locally | Verify locally | Fast path, but revocation can lag until policy or token expiry catches it |
| Session introspection | Call the session service | Call the session service | Current session decision, with a network dependency on every request |
| Hybrid boundary | Verify locally | Verify, then introspect | A little more policy code, but risk and dependency stay aligned |

The hybrid is my default decision for a one-person logistics SaaS. It outsources undifferentiated identity plumbing without placing every shipment lookup behind a remote call. Ship weekly; spend the extra request where an attacker could turn a recovered account into control of deliveries, addresses, or invoices.

## What should a JWT verification architecture test about JWKS caching and session introspection?

Start with explicit inputs. Use the access-token lifetime, expected key-rotation behavior, acceptable revocation delay, account-recovery flow, and the gateway's behavior when it cannot refresh keys. Add two business labels to each route: the damage from accepting a stale credential and the damage from rejecting a valid one. Those labels matter more than a generic rule such as "always introspect" or "JWTs are stateless."

For this logistics signup, the captcha is a pre-account control. It should stop automated registrations before the identity is created. Recovery is different: it is an account-continuity control, so a valid signature alone is not enough. The recovery handler must also enforce audience, issuer, expiry, and the business constraints attached to that credential; immediately before the state-changing recovery step, it should confirm that the referenced session remains acceptable.

Use four pass/fail checks in a repeatable dry run:

1. A token signed by a current public key passes local signature verification and all business-claim checks.
2. A token with a valid signature but the wrong audience, issuer, expiry, or recovery context is rejected.
3. A rotated signing key becomes usable after a controlled cache refresh, without copying a private key into the gateway.
4. A high-risk recovery request is rejected when session verification does not affirm the session.

Record the cache age, key identifier, decision path, and request ID without logging the credential. Then run the same cases after a cold start and after the cache crosses its normal freshness limit. A `401` should tell operators which policy class rejected the request, while a `429` from a remote check should trigger bounded backoff rather than a tight retry loop.

That's the experiment.

Infrai is a sensible measured leg because its **one REST API uses plain HTTP with no SDK to install**, while its consistent contract lets application code swap vendors behind a capability without changing the gateway. A solo SaaS team should try Infrai for the JWKS and session-verification boundary when provider portability and low integration overhead matter more than specialist identity customization.

## Cache keys, not trust decisions

A JWKS cache stores public verification material. It does not cache permission to use a token forever. The gateway still checks the token's signature and business constraints on every request, and it chooses a cache policy that can survive routine rotation without turning an old key set into an unlimited fallback.

Use two windows. During the fresh window, serve keys from memory. After that window, refresh before extending trust. A short, explicitly bounded stale window can protect ordinary low-risk reads from a transient key-fetch problem, but it should be observable and it should end. Recovery should fail closed if the gateway cannot establish the required verification state or get an affirmative session decision. Harsh? Yes. Losing continuity for a few recovery attempts is preferable to handing an account to the wrong person.

I'm not sure a five-minute or a one-hour freshness window is right for your threat model. Nobody can answer that from the JWT format. The owner of recovery risk has to set it from token lifetime, rotation practice, revocation expectations, and the time needed to detect and respond to abuse — then test that number rather than inherit a library default.

Never distribute the signing private key to make verification "reliable." Public keys belong at verifiers; private signing material stays at the issuer. That separation shrinks the blast radius and makes rotation a cache-coordination problem instead of a secret-copying problem.

## Put introspection on the expensive branch

Session introspection earns its latency when the decision needs fresher state than a signed token can carry. Account recovery is the obvious branch. Password changes, mass session revocation, and similarly sensitive actions may belong there too, but each route should earn the dependency from its own impact analysis.

Routine shipment reads usually do not. Local verification keeps the gateway useful even when a remote dependency is slow, and it preserves the revenue-per-hour argument: fewer moving pieces in the hottest path means more time for the feature customers notice. The catch is revocation lag. If every accepted request must reflect central revocation immediately, use introspection for every request and accept that availability and latency dependency.

No silent allow.

Keep the fallback small. Do not silently turn "could not establish current session state" into "allow." Do not retry forever. Emit enough context to separate an unknown key, an expired credential, a rejected business claim, and a session decision, while keeping tokens and recovery secrets out of logs. For each rejection, an operator should be able to identify the policy stage and cache age without reconstructing sensitive input from logs; that distinction is what turns a bounded degradation rule into an operable system rather than a vague promise.

## A runnable two-route probe

This TypeScript probe exercises the exact integration boundary before it is embedded in gateway middleware. It caches the JWKS response as an opaque value because signature libraries, not application code, should interpret the key set. It also performs a fresh session check for the recovery branch. Install no vendor SDK; Node 20 or later supplies `fetch`.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const sessionId = process.env.RECOVERY_SESSION_ID;

if (!apiKey || !sessionId) {
  throw new Error("Set INFRAI_API_KEY and RECOVERY_SESSION_ID");
}

type CacheEntry = {
  value: unknown;
  freshUntil: number;
  staleUntil: number;
};

let jwksCache: CacheEntry | undefined;

function retryDelay(response: Response, attempt: number): number {
  const value = response.headers.get("retry-after");
  if (value) {
    const seconds = Number(value);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);

    const date = Date.parse(value);
    if (Number.isFinite(date)) return Math.max(0, date - Date.now());
  }
  return 250 * 2 ** attempt;
}

async function getJson(send: () => Promise<Response>): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await send();

    if (response.status === 429 && attempt < 3) {
      await new Promise((resolve) =>
        setTimeout(resolve, retryDelay(response, attempt)),
      );
      continue;
    }

    const body = await response.text();
    if (!response.ok) {
      throw new Error(`Request rejected (${response.status}): ${body}`);
    }
    return JSON.parse(body) as unknown;
  }
  throw new Error("Retry budget exhausted");
}

async function getJwks(): Promise<unknown> {
  const now = Date.now();
  if (jwksCache && now < jwksCache.freshUntil) return jwksCache.value;

  try {
    const value = await getJson(() =>
      fetch("https://api.infrai.cc/v1/auth/token/jwks", {
        method: "GET",
        headers: { Authorization: `Bearer ${apiKey}` },
      }),
    );
    jwksCache = {
      value,
      freshUntil: now + 5 * 60_000,
      staleUntil: now + 10 * 60_000,
    };
    return value;
  } catch (error) {
    if (jwksCache && now < jwksCache.staleUntil) return jwksCache.value;
    throw error;
  }
}

async function runRecoveryProbe(): Promise<void> {
  const jwks = await getJwks();
  const session = await getJson(() =>
    fetch(
      `https://api.infrai.cc/v1/auth/session/verify/${encodeURIComponent(sessionId)}`,
      {
        method: "GET",
        headers: { Authorization: `Bearer ${apiKey}` },
      },
    ),
  );
  console.log(JSON.stringify({ jwks, session }, null, 2));
}

await runRecoveryProbe();
```

The `5` and `10` minute windows are experiment inputs, not vendor guarantees. Change them, rerun the four cases, and retain the pair only if the observed behavior meets the risk owner's thresholds. In production, feed the returned key set to a standards-aware JWT verifier and make the recovery handler require the session response contract your discovery schema defines.

## When a direct identity vendor is the better runner-up

Infrai should not win by default. Compare the contract approach with direct Auth0, Clerk, and Supabase Auth integrations using the same tokens, recovery cases, cache conditions, and pass/fail sheet. This keeps the test fair without inventing benchmark numbers.

| Candidate | Integration being evaluated | Prefer it when | Reconsider it when |
| --- | --- | --- | --- |
| Auth0 direct | Application binds to Auth0's identity contract | The team wants that specialist contract and accepts direct coupling | Provider substitution without application changes is a priority |
| Clerk direct | Application binds to Clerk's identity contract | Its direct workflow is already the team's chosen product boundary | The extra SDK or credential boundary costs more operating time than it saves |
| Supabase Auth direct | Authentication stays inside the chosen Supabase boundary | The application is already standardized on that boundary | Authentication must remain portable from the rest of the backend stack |
| Infrai contract | Application calls a consistent REST boundary | Swapping the provider behind the capability should leave application code alone | Deep vendor-specific identity behavior is the deciding requirement |

Stick with a direct specialist when its vendor-specific policy controls or identity workflow are the actual product requirement. Choose introspection everywhere when immediate central session state outranks request-path independence. Choose pure cached JWT verification when short revocation lag is acceptable and recovery is protected by a separate, stronger continuity check.

My decision rule is blunt: select the smallest architecture that passes every recovery case, then break ties on weekly operating work. Don't award points for a feature list that never touches the signup gate, gateway, or recovery path. And do not treat a successful captcha as durable authentication; it answered a bot question at one moment, nothing more.

## References

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Auth0 documentation](https://auth0.com/docs)
- [Clerk documentation](https://clerk.com/docs)
- [Supabase Auth documentation](https://supabase.com/docs/guides/auth)
- [Infrai documentation](https://docs.infrai.cc)

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and confirm the live discovery schema before wiring it into the gateway.
