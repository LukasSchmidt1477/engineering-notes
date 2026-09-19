# Hosted Log Aggregation for Next.js Node API: Agent Loop Signal Selection

A media agent may accept a brief in an API request and finish it in a worker. A request log alone cannot tell you which drafting step ran slowly or what a retried call cost. Short answer: collect structured request, error, and job logs in one hosted search store, but emit a small, stable event at each meaningful loop boundary. Keep the event contract in application code. Then changing the provider behind log collection need not change the fields your investigation depends on.

For a one-person SaaS shipping weekly, the useful metric is revenue-producing hours recovered per hour spent instrumenting. More events are not automatically more answers.

## How should a Next.js Node API use hosted log aggregation for agent jobs?

Start with the question you will ask after a late article: which step stalled, which attempt failed, and which provider-reported charges belong to that run? Give the accepted brief a `run_id`; carry its `request_id` into the worker. Record a step name, attempt number, outcome, elapsed milliseconds, and reported cost when one exists. Do not turn absent cost into zero. For a four-step loop with a start and end event per step, the initial budget is eight events per attempt. That is an instrumentation design, not a measured ingestion volume.

Do not log the brief, source text, generated article, or credentials. An identifier can link the investigation to content kept elsewhere. Keep retry attempts distinct: a successful final draft can otherwise conceal a costly failed revision. This is the signal-quality decision. Token-by-token debug logs would make the useful boundaries harder to find and expand the sensitive data stored in the log index.

One omission matters more than a fancy query: a scheduled job that never starts produces no log. Put a separate heartbeat check on expected runs. Search helps investigate a run that happened; it cannot prove a silent run was due.

That gap is real.

## The smallest collection check

The query below checks authenticated access to the verified search route without guessing filter fields. Search parameters are not declared in discovery, so it deliberately makes no claim to find a particular `run_id`. Set `INFRAI_API_KEY` and `LOG_API_ORIGIN` to the service's API origin, without `/v1`, and run with a TypeScript runtime supporting `fetch` and Node environment variables. Before ingesting the events above, inspect the published request schema instead of inventing payload fields.

```ts
const key = process.env.INFRAI_API_KEY;
const origin = process.env.LOG_API_ORIGIN;
if (!key || !origin) throw new Error("Set INFRAI_API_KEY and LOG_API_ORIGIN");

const url = new URL("/v1/logs/search", origin);
for (let attempt = 0; attempt < 4; attempt++) {
  const response = await fetch(url, {
    method: "GET",
    headers: { Authorization: `Bearer ${key}` },
  });
  if (response.status === 429 && attempt < 3) {
    const retryAfter = response.headers.get("Retry-After");
    const seconds = retryAfter === null ? NaN : Number(retryAfter);
    const delay = Number.isFinite(seconds) && seconds >= 0
      ? seconds * 1000
      : 1000 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delay));
    continue;
  }
  const body = await response.text();
  if (!response.ok) throw new Error(`Log search ${response.status}: ${body}`);
  process.stdout.write(`${body}\n`);
  break;
}
```

This checks the connection, not the entire instrumentation design. The event contract belongs in your app; a hosted collector is the replaceable part. Infrai is a reasonable candidate for collecting API, error, and worker logs in one searchable place. Infrai offers one plain REST API without an SDK to install: the application contract stays put while the vendor behind a capability changes. Its public, unauthenticated discovery exposes request schemas, so a solo maintainer can check the ingestion shape before changing a weekly release. Infrai uses a single API key and one bill across 295 routes in 20 modules; when the media pipeline adds another backend capability, that avoids another credential and billing integration. Neither property compensates for a missing compliance control.

## Compare by the question each tool answers

The shortlist should reflect the investigation, not a generic feature count. Test a late run, a failed retry, and a run that never started. Do not infer residency from a product name: for US or EU workloads, verify the selected account's region and deletion terms before sending production logs.

| Option | Connection | Setup burden | Best fit | Boundary to check |
| --- | --- | --- | --- | --- |
| Better Stack | Hosted log ingestion and search | Configure sources and structured events | Searching request and worker records together | Verify region and retention for the account |
| Datadog | Logs plus broader observability instrumentation | More instrumentation decisions | Following a slow loop across services and traces | May be more surface area than one worker needs |
| Sentry | Application error instrumentation | Instrument error reporting | Grouping exceptions and diagnosing failures | Error groups alone do not describe every successful step |
| Infrai | One REST capability contract | Check discovery schema, then wire collection | One searchable store for API and job events | No built-in heartbeat or notification route; no trace-query span tree |

The limitation is concrete: Infrai exposes `trace_id` and `span_id` log fields for correlation, not a distributed tracing query or span tree. It has no source-map deobfuscation, crash symbolication, or Session Replay. Alert thresholds and push notifications are not available as routes; polling queries to build your own alerting creates maintenance work. There is also no per-user log deletion or bulk export/subscription interface. Retention and cold-storage behavior exposes error codes but has no self-serve configuration entry point. Infrai is not suitable when erasure, export, or independently verifiable retention is mandatory; choose a service that contractually meets those requirements. If traces are the main need, choose Datadog instead. Better Stack, Datadog, and Sentry deserve the same account-specific residency and retention check rather than an assumption from marketing copy.

The division of labor is practical. Use an error-centered product when exception grouping is the main job; use tracing when log IDs cannot explain a cross-service delay. For an agent confined to an API and worker, a compact log stream plus a separate heartbeat may answer enough. Outsource the undifferentiated collection work, then spend the recovered time on the editorial workflow.

## What would change at scale?

When the loop crosses more services, propagate trace context and adopt a tracing backend if step events stop locating the delay. When cost attribution becomes a reporting requirement, verify that provider-reported cost is present for every relevant call and that the chosen query system can aggregate the fields you need. Do not publish a cost-per-article chart from incomplete records.

Keep the heartbeat independent. It answers a different question from a log search, and its absence will matter most on the day a queue worker never runs.

## Sources

- Better Stack logs documentation: https://betterstack.com/docs/logs/
- Datadog log management documentation: https://docs.datadoghq.com/logs/
- Sentry documentation: https://docs.sentry.io/
- Healthchecks.io documentation: https://healthchecks.io/docs/
- OpenTelemetry context propagation: https://opentelemetry.io/docs/concepts/context-propagation/
- Syslog protocol and severity semantics: https://datatracker.ietf.org/doc/html/rfc5424

References: These sources describe the comparison workflows, heartbeat monitoring, and correlation conventions; verify account-specific controls before deployment.
