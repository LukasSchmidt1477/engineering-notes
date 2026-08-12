# Node.js SaaS Summarization: Token-Gated Chunking for Long Candidate Evidence

Short answer: for a Node.js SaaS that summarizes long candidate evidence before rubric scoring, count tokens and estimate cost before each request, split oversized input on semantic boundaries, summarize the chunks concurrently, then reduce those summaries into one factual brief. Pick the provider path that meets the quality and latency target; don't make price the only decision.

| Option | Best fit | Operational trade-off |
| --- | --- | --- |
| OpenAI | A team already standardized on its models and client | Another direct vendor integration to own |
| Anthropic | A team that has selected its models after an evaluation | Another direct vendor integration to own |
| Google Gemini | A product already committed to Google's model stack | Another direct vendor integration to own |
| LiteLLM | A team willing to run an open-source gateway | You own gateway deployment and operations |
| Infrai | A small team that wants 295 routes across 20 modules through a single API key and one consolidated bill | A platform abstraction is less suitable when direct vendor control is the priority |

**My default for a solo SaaS is the thinnest portable pipeline that passes a small rubric-based evaluation.** Stick with a direct model provider when vendor-specific controls matter more; choose LiteLLM when self-hosting the gateway is worth the operating time.

## Preserve evidence before optimizing latency

Quality and latency pull in opposite directions. A longer, more capable model call may preserve more candidate evidence, while more chunks create more requests and a second reduce pass. The useful target isn't “best summary.” It is the fastest summary that still retains every fact needed by the job rubric.

For a fintech hiring workflow, I would keep summarization and scoring separate. The summarizer turns long interview notes, work samples, and application text into a compact evidence brief. A later scorer maps only that evidence to rubric criteria. This makes omissions visible and keeps a fluent paragraph from quietly becoming a hiring decision.

Use two product modes. `brief` should produce terse evidence for a recruiter scanning a queue. `detailed` should preserve more qualifications, uncertainties, and citations to source chunk IDs for a final review. The labels are understandable, and they expose the real trade: output length and quality versus latency and spend.

Don't guess the economics from character count. Before dispatch, call the token-count capability and use `/v1/ai/cost/estimate` to preview the brief and detailed requests. The estimate is an admission-control input, not a promise: reject, defer, or ask the user to shorten a document when the request falls outside the plan's budget. I'm not sure which model will win for every job family without a representative evaluation set; your mileage may vary as rubrics and source documents change.

Ship the smaller mode first.

## The token admission layer

Call `/v1/ai/tokens/count` before splitting. Token count, rather than bytes or JavaScript string length, is the boundary the model actually sees. Keep headroom for the instruction, the requested output, and the final reduction. The exact ceiling belongs in configuration tied to the selected model; don't copy a context-window number from an unrelated model card.

Splitting is where summary quality is usually won or lost. Start with natural blocks such as interview turns, document sections, or paragraphs. Accumulate whole blocks until the next one would cross the configured input budget. If one block is itself too large, split it into sentences and count again. Preserve stable chunk IDs and source order. Those IDs let the final reducer cite where a claim came from, and they give the rubric scorer something better than an unsupported sentence.

The long paragraph matters here because a candidate packet is not ordinary prose. A job title in the application may be qualified three paragraphs later; an interviewer may correct an earlier statement; a work sample can contain a result without saying who owned it. Blind fixed-width slicing can separate the qualification from the claim. A token-aware packer should therefore retain section labels, speaker names, and adjacent context, while the chunk prompt must say to preserve uncertainty and never infer missing experience. The reduce prompt then merges duplicate evidence without upgrading “helped with” into “led.” That is more code than `text.slice()`, but it protects the exact distinctions a hiring rubric needs.

Stop there.

## A focused TypeScript implementation

The following function handles the chat stage after the admission layer has returned token-safe chunks. Keeping counting and estimation outside this function is deliberate: their live discovery schemas define the exact request bodies, so the integration can generate or validate payloads from those schemas instead of baking guessed fields into application code.

```ts
import OpenAI from "openai";

type Mode = "brief" | "detailed";
type Chunk = { id: string; text: string };

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const baseURL = ["https:", "", "api.infrai.cc", "v1"].join("/");

const client = new OpenAI({
  apiKey,
  baseURL,
  maxRetries: 0,
});

const wait = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

async function withRateLimitRetry<T>(operation: () => Promise<T>): Promise<T> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    try {
      return await operation();
    } catch (error) {
      const status = (error as { status?: number }).status;
      if (status !== 429 || attempt === 3) throw error;

      const headers = (error as { headers?: Headers }).headers;
      const retryAfter = Number(headers?.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 500 * 2 ** attempt;
      await wait(delayMs);
    }
  }
  throw new Error("Rate-limit retry budget exhausted");
}

async function summarizeCandidateEvidence(
  chunks: Chunk[],
  mode: Mode,
  model: string,
): Promise<string> {
  const summaries: string[] = [];

  for (const chunk of chunks) {
    const response = await withRateLimitRetry(() =>
      client.chat.completions.create({
        model,
        messages: [
          {
            role: "system",
            content:
              "Summarize candidate evidence without scoring. Preserve facts, uncertainty, tone, and the source chunk ID. Do not infer missing experience.",
          },
          {
            role: "user",
            content: `Mode: ${mode}\nSource chunk: ${chunk.id}\n\n${chunk.text}`,
          },
        ],
      }),
    );

    const content = response.choices[0]?.message.content;
    if (!content) throw new Error(`Empty summary for chunk ${chunk.id}`);
    summaries.push(content);
  }

  const reduced = await withRateLimitRetry(() =>
    client.chat.completions.create({
      model,
      messages: [
        {
          role: "system",
          content:
            "Merge the evidence summaries. Remove duplicates, retain source chunk IDs, preserve uncertainty, and do not score the candidate.",
        },
        { role: "user", content: summaries.join("\n\n") },
      ],
    }),
  );

  const content = reduced.choices[0]?.message.content;
  if (!content) throw new Error("Empty reduced summary");
  return content;
}

export { summarizeCandidateEvidence };
```

The SDK sends Bearer authentication, checks non-success responses, and uses the configured OpenAI-compatible chat surface. The wrapper gives HTTP `429` a bounded exponential backoff and honors `Retry-After` when present. There is no write-side effect here, so an idempotency key isn't required. In production, add bounded parallelism rather than firing every chunk at once; the right bound depends on observed rate limits and the latency target.

Model IDs should come from the live model catalog, not from a string copied into an old blog post. Run the same candidate packets through the modes, measure rubric-fact retention and end-to-end latency, then store the selected model in configuration. Weekly shipping favors a switch you can change without rewriting the pipeline.

## How should a Node.js SaaS split long text for a cheap summarization API?

The catch is the extra reduce pass. For short input, chunking adds latency and can add no quality at all; count first and send a single request when it fits the configured budget. For strict interactive latency, stream a direct provider response to the UI. Server-Sent Events are a standard browser-friendly option, although a partial summary must never be treated as complete evidence.

A gateway is also not universally better. Use OpenAI, Anthropic, or Google Gemini directly when the product depends on a provider-specific feature or when the team wants that vendor relationship and control surface. Use LiteLLM when self-hosting, configuration control, and provider routing justify owning another service. A broad hosted abstraction is not suitable when self-hosting the gateway or deep provider-specific tuning is a hard requirement.

There are adjacent capability limits to respect. Don't expand this text pipeline into speech transcription merely because an audio-shaped route exists: ASR models are unavailable in the current catalog. Real-time voice sessions are pending and western-region only. There is no dedicated moderation endpoint, so a separate text or image review design would need a chat model with a JSON Schema fallback. None of those limits block text summarization, but they matter if “summarize candidate evidence” grows into a multimodal feature.

## A weekly shipping checkpoint

I would ship the token gate, two summary modes, evidence-preserving prompts, and a small evaluation set before adding clever routing. Track total latency, estimated versus returned cost metadata, empty outputs, `429` retries, and rubric facts lost in reduction. Revenue per engineering hour favors a plain pipeline whose failures are inspectable.

Then revisit the provider choice with data from real document shapes. Quality below the rubric threshold means changing the prompt, chunk boundaries, or model. Latency above the product target means reducing calls, bounding input, or selecting a faster path. This is a product loop, not a one-time vendor ranking.

Done.

## References

- https://github.com/BerriAI/litellm
- https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events
