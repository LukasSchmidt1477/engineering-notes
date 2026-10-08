# Text and Image Safety Explained — Moderation Behind One API Key

TL;DR: The simplest text and image moderation architecture puts one policy service, reached with one API key, between user content and the application. Do not confuse one credential with one undifferentiated chat model request. For a fintech system that turns sales calls into CRM actions, I would run fast text checks before generating actions, send images through a separate preparation path, and require structured output from both. Favor quality for anything that can change a customer record; favor latency only for reversible previews.

| Choice | Quality | Latency | Operational load | Best fit |
|---|---|---|---|---|
| One synchronous multimodal request | Context stays together | User waits for the slowest input | Low at first | Small, bounded submissions |
| Separate text and image checks, shared policy | Each modality gets tailored handling | Text can finish first | Moderate | Mixed comments, avatars, and uploads |
| Fully asynchronous review pipeline | Supports deeper checks and retries | No immediate final decision | Highest | Large uploads and high-risk actions |

**Recommendation:** use the middle option. It keeps the public contract simple without coupling a short comment to image decoding and network fetch time. One API key can authenticate the policy service. Internally, text and images should still have distinct timeouts, queues, and evidence.

## How should one API key handle text and image moderation?

An API key answers an identity question: which service may call the moderation runtime? It does not decide how bytes should be decoded, how long a caller should wait, or what evidence an auditor will need later. Those are policy and execution questions.

That distinction matters in a sales-call workflow. A transcript may contain an instruction such as “create a follow-up task,” while a shared screenshot may contain an account number or unrelated customer data. The eventual CRM action depends on both, but the inputs fail differently. Text can be truncated or lose speaker attribution. An image can be malformed, oversized, animated, or fetched from an unsafe location. Forcing those failures into a single opaque call makes retries expensive and explanations thin.

Keep the outside narrow: one authenticated moderation endpoint, one submission identifier, one decision shape. Behind it, dispatch by media type. The result is still one integration from the product team's point of view.

The boundary should own four jobs: normalize input, evaluate it, combine the evidence, and record the policy version. It should not create the CRM task itself. Moderation decides whether content may proceed; a later component decides what business action the content implies. That separation prevents a model response from becoming an authorization token by accident.

## Quality first where mistakes persist

The useful quality question is not “How accurate is the model?” in isolation. Ask what happens after a wrong decision. A blocked avatar can be retried. An incorrect CRM note may be read by an account team, copied into a risk review, or used to trigger another workflow. Persistence raises the cost of a false allow and a false interpretation.

I would therefore divide outputs into three states: `allow`, `block`, and `review`. Binary output looks simpler, but it hides uncertainty precisely where a small operation needs an escape hatch. The review state is not permission to proceed. It is a durable stop with enough reason codes for a person or a later process to resolve it.

Use a written policy fixture for each surface. A comment fixture should preserve speaker turns and nearby context. An avatar fixture should include the decoded image properties. An upload fixture should identify every extracted component rather than collapsing the whole file into a filename. Run the same fixtures whenever the policy, prompt, preprocessing, or model changes. Ship weekly, but make policy changes observable releases.

Do not store free-form model prose as the decision contract. Store structured evidence and validate it before any downstream action. If parsing fails, the safe result is `review`, not an optimistic allow.

Short inputs deserve context too. “Send it now” may be harmless in a call transcript and dangerous if the proposed CRM action sends regulated material to an unverified recipient. Moderation should receive the intended action as data, while the action executor independently enforces permissions.

## Latency belongs to the user-visible promise

Latency is not one budget. A comment preview, an avatar update, and a document-backed CRM action have different promises.

For a preview, return a provisional state and keep the control disabled until the final decision arrives. Server-Sent Events are a reasonable transport when the browser only needs one-way progress updates from the server. The browser's `EventSource` interface maintains that stream, and named events can distinguish preparation, evaluation, and completion. This changes delivery, not policy: a dropped connection must not turn an unfinished review into an allow.

For a persistent CRM action, wait for the final decision or queue the action. Slow is acceptable. Ambiguous is not.

This is also where image preparation earns its keep. Decode the upload on infrastructure you control, reject unsupported input, correct orientation, constrain dimensions, and produce a known output representation before evaluation. A Node image-processing library such as `sharp` exposes operations for resizing and format conversion. Preparation should be bounded by byte, pixel, and time limits chosen for the application. Those limits are configuration, not universal safety constants.

Do not fetch arbitrary image URLs from the moderation worker. Accept uploaded bytes through the application's existing upload boundary or retrieve only from controlled object storage. This keeps content acquisition separate from content judgment and makes retries deterministic.

## A minimal TypeScript contract

The implementation below is intentionally provider-neutral. The adapter owns the remote call; the policy layer owns validation and combination. Text and images can use different adapters while sharing one credential supplied to the process at runtime.

```ts
type Verdict = "allow" | "block" | "review";
type Surface = "call_comment" | "avatar" | "attachment";

type Evidence = {
  verdict: Verdict;
  reasons: string[];
  policyVersion: string;
};

type ModerationInput =
  | { kind: "text"; surface: Surface; text: string }
  | { kind: "image"; surface: Surface; bytes: Uint8Array; mediaType: string };

interface ModerationAdapter {
  evaluate(input: ModerationInput, credential: string): Promise<unknown>;
}

function parseEvidence(value: unknown): Evidence {
  if (typeof value !== "object" || value === null) {
    throw new Error("Moderation output must be an object");
  }

  const record = value as Record<string, unknown>;
  const verdicts = new Set<Verdict>(["allow", "block", "review"]);
  if (!verdicts.has(record.verdict as Verdict)) {
    throw new Error("Unknown moderation verdict");
  }
  if (!Array.isArray(record.reasons) ||
      !record.reasons.every((reason) => typeof reason === "string")) {
    throw new Error("Moderation reasons must be strings");
  }
  if (typeof record.policyVersion !== "string") {
    throw new Error("A policy version is required");
  }

  return {
    verdict: record.verdict as Verdict,
    reasons: record.reasons,
    policyVersion: record.policyVersion,
  };
}

function combine(results: Evidence[]): Evidence {
  const verdict: Verdict = results.some((result) => result.verdict === "block")
    ? "block"
    : results.some((result) => result.verdict === "review")
      ? "review"
      : "allow";

  return {
    verdict,
    reasons: [...new Set(results.flatMap((result) => result.reasons))],
    policyVersion: results.map((result) => result.policyVersion).join("+"),
  };
}

async function moderateAll(
  adapter: ModerationAdapter,
  inputs: ModerationInput[],
  credential: string,
): Promise<Evidence> {
  try {
    const raw = await Promise.all(
      inputs.map((input) => adapter.evaluate(input, credential)),
    );
    return combine(raw.map(parseEvidence));
  } catch {
    return {
      verdict: "review",
      reasons: ["evaluation_incomplete"],
      policyVersion: "unresolved",
    };
  }
}
```

The important behavior is the last branch. A timeout, malformed response, or unavailable evaluator produces an incomplete review. It does not silently publish a comment or write a CRM action.

In production, the submission record also needs immutable input references, timestamps, adapter request identifiers, and the policy configuration used for the decision. Keep raw sensitive content only as long as the product's retention and audit obligations require. The compact `Evidence` object is an application contract, not a complete audit record.

## When the runner-up architecture is better

Choose one synchronous multimodal request when submissions are small, the user is already waiting for a final answer, and joint context materially affects the decision. It can be a good fit for a bounded sales-call summary with one attached screenshot. Set an explicit request limit. Reject excess input before evaluation rather than trimming it invisibly.

Choose the fully asynchronous pipeline when uploads require extraction, multiple images, human review, or long-running analysis. It adds queue state, idempotency, cancellation, and dead-letter handling. That is real ownership cost. A one-person SaaS should pay it only when the product promise needs it, because every hour maintaining queue edges is an hour not spent shipping the differentiated workflow.

The middle architecture remains the default because it allows independent retries. If avatar preparation fails, the text comment does not need to be evaluated again. If the policy changes, stored normalized inputs can be reevaluated under a new version without replaying the entire product action. This is outsourcing the undifferentiated work to narrow adapters while retaining the policy decisions that define the product.

Its limitation is coordination. The service must define what happens when text is allowed but an image is blocked, and it must preserve a single submission state across two evaluations. It is a poor fit when the decision depends on fine-grained relationships between words and pixels in the same item; in that case, the synchronous joint-context option is clearer. The downside of that runner-up is coupled latency and less selective retry behavior. This is a trade-off, not a universal ranking.

No architecture removes the need for deployment discipline. Roll out policy versions separately, compare decisions on fixed fixtures, monitor the rates of all three verdicts by surface, and alert on parse failures and incomplete evaluations. Track latency by stage rather than averaging text and image work together. The average conceals the path users actually feel.

The simplest useful design is not a single call. It is a single product boundary with explicit internal stages. Quality wins for durable CRM changes; latency wins for reversible feedback; structured output keeps either choice from leaking model prose into business authority.

## Sources

References:

- MDN, “Using server-sent events”: https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events
- sharp documentation: https://sharp.pixelplumbing.com
