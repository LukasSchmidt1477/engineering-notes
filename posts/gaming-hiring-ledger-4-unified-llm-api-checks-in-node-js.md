# Gaming Hiring Ledger: 4 Unified LLM API Checks in Node.js

Short answer: choose a unified LLM API with one key only if the backend can attribute every model call to a tenant, preserve the usage record returned for that call, and reject invalid rubric scores before they reach a hiring workflow. A shorter credential list is useful. A trustworthy tenant ledger is the actual decision.

For a one-person SaaS that scores gaming job candidates against a fixed rubric, I would optimize for revenue per engineering hour. The backend should have one narrow model interface, one validation path, and one append-only usage path. It should not pretend that OpenAI, Claude, and Gemini are interchangeable beneath the surface. Ship the small boundary this week; keep provider-specific behavior inside adapters.

One key isn't the goal.

That distinction matters in a US/EU deployment. Region, retention, contract, and model availability requirements belong in tenant policy, not in a clever fallback chain. I'm not sure any static vendor matrix stays accurate for long; an accepted provider list, checked against current contracts and documentation, resolves that uncertainty better than an article can.

## How should a simple Node.js backend unify one LLM API key?

Start with the billing question, not the HTTP client. For each candidate-scoring request, the application already knows the tenant, rubric revision, candidate record, and chosen policy. Carry those identifiers through the model boundary. When the response returns, write usage beside the same identifiers before returning the score. If usage is absent, mark the ledger row incomplete and keep that state visible. Don't silently estimate it from string length.

The concrete constraint is per-tenant cost visibility. A monthly provider invoice can answer what the whole service consumed, but it cannot by itself tell a solo operator which gaming studio, open role, or rubric revision caused the spend. The useful unit is one attempt: tenant, request ID, provider, model, input units, output units, latency, and outcome. Monetary cost can be derived later from a versioned rate table. Keeping raw usage means a pricing update does not rewrite history.

Four checks decide whether a unified API earns its place:

1. The caller supplies a tenant ID, and the gateway or application returns a request ID that can be reconciled.
2. The response exposes model identity and usage rather than hiding them behind one blended number.
3. Routing policy can constrain providers by tenant and region before a request is sent.
4. Structured output is validated in the application, with failed attempts recorded separately from accepted scores.

This is also where “best” becomes a bad selection criterion. OpenAI, Claude, and Gemini are three real model families, but the right model can differ by rubric, language, contract, and acceptable latency. A unified layer should expose those decisions. If it erases them, the simple backend is simple only on the diagram. One key reduces secret rotation work — useful when there is no platform team — yet it also enlarges the blast radius of that credential. Scope it, rotate it, keep it out of tenant-facing clients, and ensure tenant policy is enforced server-side. The browser should never choose an unrestricted provider or attach an untrusted billing label.

Keep that visible.

## The smallest working tenant ledger

The application boundary can stay deliberately boring. The code below does not assume a commercial gateway or a particular provider SDK. Each adapter translates its provider's request and response into the same internal types; the scoring service owns tenant policy, schema validation, and accounting.

```ts
type Region = "US" | "EU";
type Provider = "openai" | "claude" | "gemini";

type ScoreRequest = {
  requestId: string;
  tenantId: string;
  region: Region;
  provider: Provider;
  model: string;
  rubricVersion: string;
  candidateText: string;
  rubric: Array<{ criterion: string; maxPoints: number }>;
};

type ScoreResult = {
  total: number;
  reasons: Array<{ criterion: string; points: number; note: string }>;
};

type ModelResult = {
  provider: Provider;
  model: string;
  inputUnits: number;
  outputUnits: number;
  rawJson: unknown;
};

interface ModelAdapter {
  score(request: ScoreRequest): Promise<ModelResult>;
}

type LedgerRow = {
  requestId: string;
  tenantId: string;
  rubricVersion: string;
  region: Region;
  provider: Provider;
  model: string;
  inputUnits: number;
  outputUnits: number;
  durationMs: number;
  outcome: "accepted" | "rejected";
};

interface Ledger {
  append(row: LedgerRow): Promise<void>;
}

function parseScore(value: unknown, request: ScoreRequest): ScoreResult {
  if (typeof value !== "object" || value === null) throw new Error("invalid_score");
  const candidate = value as Partial<ScoreResult>;
  if (!Number.isFinite(candidate.total) || !Array.isArray(candidate.reasons)) {
    throw new Error("invalid_score");
  }

  const maximum = request.rubric.reduce((sum, item) => sum + item.maxPoints, 0);
  if (candidate.total! < 0 || candidate.total! > maximum) {
    throw new Error("score_out_of_range");
  }
  return candidate as ScoreResult;
}

async function scoreCandidate(
  request: ScoreRequest,
  adapters: Record<Provider, ModelAdapter>,
  ledger: Ledger,
): Promise<ScoreResult> {
  const startedAt = Date.now();
  const result = await adapters[request.provider].score(request);
  let outcome: LedgerRow["outcome"] = "rejected";

  try {
    const score = parseScore(result.rawJson, request);
    outcome = "accepted";
    return score;
  } finally {
    await ledger.append({
      requestId: request.requestId,
      tenantId: request.tenantId,
      rubricVersion: request.rubricVersion,
      region: request.region,
      provider: result.provider,
      model: result.model,
      inputUnits: result.inputUnits,
      outputUnits: result.outputUnits,
      durationMs: Date.now() - startedAt,
      outcome,
    });
  }
}
```

The adapters are intentionally missing from the example. Their job is mechanical translation, and their correct fields depend on the current provider or gateway contract. Copying a speculative endpoint would make the sample look complete while making it less durable. The interface is the part the application owns.

There is one sharp edge in this minimal version: if the adapter throws before returning a normalized result, the `finally` block cannot write its usage fields. Production code should create an attempt row before the call, then finalize it with the returned identity and usage. Consider a scoring operation that is sent once, receives a `429`, and is retried. The first attempt still needs its own row even though there is no normalized model result; the retry needs a second attempt number linked to the same operation; and only the accepted response should populate the candidate score. Recording a guessed zero for the first attempt would turn an unknown into a financial fact. Dropping the row would make retry rates invisible. The honest representation is an incomplete usage state that reconciliation can later resolve from an export or leave explicitly unknown. That gives timeouts, client-side validation failures, and rate limits a visible state without inventing consumption data. A `429` is an operational outcome, not evidence that zero units were billed. Keep the score and ledger separate as well. Candidate results may have a deletion schedule or restricted access, while aggregate usage may need a different lifecycle. The ledger needs identifiers for reconciliation, not the candidate's resume text, name, or generated rationale. Less sensitive data makes routine cost analysis easier to operate.

Unknown is a state.

## What would change at 100 tenants?

At small volume, a database transaction and a daily aggregation job are enough. At 100 tenants, I would still resist building a general model platform. I would add idempotency on `requestId`, an attempt/finalization state machine, a versioned price table, and a reconciliation job that compares internal totals with provider or gateway exports. These are accounting controls tied to the actual job. A visual routing editor is not.

Retries need special care. Reusing one request ID for several billable attempts makes the ledger lie; generating unrelated IDs loses the causal chain. Use a stable operation ID plus an incrementing attempt number. Store accepted and rejected output counts separately, then decide explicitly which attempts a tenant sees or pays for. The policy is a product decision, but the data needed to enforce it is an engineering decision.

Routing should remain inspectable. A tenant policy might permit a named set of providers in `EU`, while another permits a different set in `US`. Resolve that policy before calling the adapter and record the resolved provider and model afterward. Do not infer region from a model name. Do not let an automatic fallback cross the policy boundary merely because the first attempt was slow. Tests should use fixed adapter fixtures for three cases: valid structured output, invalid rubric totals, and a thrown client error before normalized usage exists. Then run a small live contract test for every enabled adapter during deployment. The fixture protects application behavior; the contract test catches request or response drift. They answer different questions.

This is the point where outsourcing undifferentiated work can make sense. LiteLLM is an open-source, self-hosted LLM gateway, so it is one example of a component that can sit behind the internal adapter. Direct provider integrations are another option. The application contract and ledger should survive either choice. Otherwise a gateway decision has leaked into hiring logic.

Ship weekly. Measure the rejected-output rate, unfinalized ledger rows, retry count, and usage by tenant and rubric revision. Those four views reveal more than a single blended spend chart because they connect cost to a workflow the business can change.

## Trade-offs and the decision rule

A unified gateway is suitable when credential consolidation and consistent accounting remove recurring operational work, while the gateway still exposes provider identity, usage, policy controls, and exportable records. It is not suitable when a tenant contract requires a direct provider relationship, when a required region or model is unavailable through the layer, or when the layer hides fields needed for reconciliation. In those cases, keep direct adapters for the constrained tenants.

There is another catch: one internal schema creates pressure to support only the common denominator. A candidate-scoring rubric may fit a small JSON result today, but a future workflow could rely on a provider-specific capability. The escape hatch should be explicit extension data inside a provider adapter, reviewed like any other dependency. It should not become an untyped bag passed through the whole application.

Self-hosting shifts control toward the operator and also adds upgrades, capacity planning, logs, and incident response to a one-person backlog. A managed layer shifts more of that work outward but requires careful review of data handling, regional deployment, usage exports, and credential scope. Neither is universally better. The revenue-per-hour test is whether the layer removes more maintenance than it creates without weakening tenant-level evidence.

My decision rule is plain: run one representative rubric through every allowed path, verify that accepted and rejected attempts reconcile to tenant usage, and inspect the region policy before adopting the layer. If any of those checks requires guesswork, keep the adapters direct and revisit after the interface is measurable. One key is an implementation detail. The ledger is the product control.

## References

- OpenAI, “Embeddings guide”: https://platform.openai.com/docs/guides/embeddings
- LiteLLM, open-source self-hosted LLM gateway: https://github.com/BerriAI/litellm
