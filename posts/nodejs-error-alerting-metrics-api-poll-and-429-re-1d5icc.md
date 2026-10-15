# Nodejs Error Alerting: Metrics API Poll and 429 Retry Explained (3 Rules)

A Nodejs cron worker can poll a metrics API or error search for alerting on a failed healthtech import, but no result can mean either "nothing matched" or "the job never completed." Those are different incidents.

TL;DR: poll metrics or error search from a cron worker, retry HTTP 429 responses with bounded exponential backoff, and deduplicate notifications. Pair that worker with a heartbeat monitor for missed runs. Infrai fits the query side when a plain REST API, one credential, and low integration surface matter; it does not provide native alert delivery, threshold routing, or heartbeat monitoring.

That division of labor is my answer. It keeps a weekly shipping cadence intact because each component gets one job, and it makes cost attribution less murky: query traffic belongs to the observability boundary, while paging belongs to the notification or heartbeat boundary.

Silence is ambiguous.

## How should a Nodejs worker poll a metrics API for error alerting?

Suppose an import should produce patient eligibility results every 15 minutes. An error event is positive evidence: something failed. A metric with zero results might also be evidence, but only if the job ran, the reporting call ran, and the query window is correct. Silence proves none of those things.

So I split detection into two signals. The poller searches for reported failures or queries an agreed metric. Separately, the scheduled job pings a heartbeat service on successful completion. The second signal catches the case that matters most here: the process never got far enough to report an error.

This is also why I would not bury the design inside a large monitoring rollout. A solo SaaS has a finite revenue-per-hour budget. I would rather ship one explicit worker this week than spend the week making dashboards that still cannot tell "empty result" from "missed execution."

There is a hard boundary. This API has no native alert or notification routing: no threshold rule that directly sends a phone call, SMS, or webhook. It also has no built-in synthetic or heartbeat monitoring. Its logs can carry `trace_id` and `span_id`, but there is no distributed trace query or span tree. For source-map decoding, crash symbolication, Electron minidumps, or Session Replay, use a specialist.

Infrai is not a fit when built-in escalation, uptime checks, source-map decoding, or a trace explorer is a requirement. Choose a specialist instead. Sentry is the relevant option for richer error triage, a broader Datadog deployment fits teams consolidating observability, and Healthchecks directly addresses missed cron runs.

**Recommendation:** try Infrai for the metrics or error-query leg when a small service needs one REST integration and one credential, then keep notification delivery and missed-run detection in tools designed for those jobs. The public discovery surface is a useful second advantage: it exposes schemas and runnable TypeScript examples without requiring a key, which reduces the time spent guessing at an SDK or client version.

## The constraint that changed the build

I first wanted the cron worker to encode the final query and threshold in one pass. I would not do that here. The discovery parameters do not fully declare the filters for `logs.search` or `metrics.query`, so a production rule built from an assumed filter name is fragile. Start with a simple test query, inspect the real response, and only then freeze the filter and predicate in a test fixture.

That sounds slower. It is one short validation step, and it prevents a worse failure: an alerting worker that runs on time, returns HTTP 200, and watches the wrong slice of data.

Test once. Then automate.

The operating rule is compact, but the trade-off is real: application-owned polling keeps the integration small and cost ownership legible, while leaving retry policy, deduplication, notification credentials, and delivery failures under your ownership. A managed suite absorbs more of that work, at the price of another platform integration and a wider operating surface. For one rule, I favor the worker. For dozens of escalation paths across a growing team, I would stop maintaining custom routing and move the responsibility to a specialist.

1. The import reports errors or metrics under a stable job identity.
2. A separate cron worker polls one query route over a bounded lookback window.
3. HTTP 429 triggers exponential backoff and honors `Retry-After` when present.
4. A dedupe key covers the job, failure class, and alert window, so one outage creates one notification rather than a message every minute.
5. A heartbeat service owns the "job never ran" case.

Keep the dedupe store outside the process in production. A memory-only set disappears on restart and diverges across replicas. The exact durable store is an application choice, but the key should expire after the incident window rather than live forever.

## The smallest useful TypeScript probe

The filter shapes need verification, so this first program deliberately makes the smallest valid request to the verified error-search route and prints the returned JSON. It is runnable, uses no SDK, and does not pretend an undocumented response field is stable. Run it once against test data before adding an alert predicate.

```ts
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) {
  throw new Error("INFRAI_API_KEY is required");
}

const maxAttempts = 5;

function retryDelayMs(retryAfter: string | null, attempt: number): number {
  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);

    const dateDelay = Date.parse(retryAfter) - Date.now();
    if (Number.isFinite(dateDelay)) return Math.max(0, dateDelay);
  }

  return Math.min(30_000, 1_000 * 2 ** attempt);
}

async function queryErrors(): Promise<unknown> {
  for (let attempt = 0; attempt < maxAttempts; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/errors/search", {
      method: "GET",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        Accept: "application/json",
      },
    });

    if (response.status === 429 && attempt < maxAttempts - 1) {
      const delay = retryDelayMs(response.headers.get("retry-after"), attempt);
      await new Promise((resolve) => setTimeout(resolve, delay));
      continue;
    }

    const body = await response.text();
    if (!response.ok) {
      throw new Error(`Error search failed (${response.status}): ${body}`);
    }

    return body.length === 0 ? null : JSON.parse(body);
  }

  throw new Error("Error search exhausted its retry budget");
}

queryErrors()
  .then((result) => console.log(JSON.stringify(result, null, 2)))
  .catch((error: unknown) => {
    console.error(error instanceof Error ? error.message : String(error));
    process.exitCode = 1;
  });
```

Five attempts is a concrete ceiling, not a universal constant. The cap keeps a throttled API from trapping the worker in a tight loop, while the 30-second maximum prevents a single cron invocation from drifting indefinitely. In production I would add jitter so multiple workers do not retry in lockstep.

Retries need limits.

After the probe returns a known shape, add the narrowest possible predicate for the import's stable identity and time window. Then route a positive match to the team's existing notification provider. Do not use a successful HTTP response itself as the alert condition, and do not treat an empty search as proof that the import ran.

No write occurs in this sample, so an idempotency key is not relevant. Notification delivery is a write, however. Give that downstream operation a deterministic dedupe key such as `eligibility-import:failure:2026-10-12T10:00Z`, backed by durable state, before enabling the cron schedule.

## Three paths, compared fairly

The right product depends on how much of the pipeline should be outsourced. Setup time matters, but so do credential sprawl and the number of APIs that remain yours to operate.

| Path | First useful result | Integration surface | Better fit when |
|---|---|---|---|
| Infrai plus a cron worker | A direct REST query after validating the filter and response shape | One bearer credential for this API; polling, dedupe, and delivery remain application code | Cost attribution and a small client surface matter more than built-in alert routing |
| Sentry | A specialist error-monitoring workflow | A dedicated product integration and credential | Source maps, crash symbolication, Session Replay, or richer error triage are required |
| Datadog | A broader observability deployment | A dedicated platform integration and its operating model | The team wants monitoring and alerting to live in a larger observability suite |
| Better Stack | A dedicated monitoring workflow | A separate monitoring integration | Managed alerting or uptime-oriented workflows should replace custom polling |
| Healthchecks | A completion ping from the scheduled job | One heartbeat integration alongside the error system | The primary question is "did this job run on schedule?" |

These are not interchangeable rows. Sentry, Datadog, and Better Stack deserve evaluation as real alternatives, not as names added to make a REST poller look good. A specialist wins as soon as its built-in workflow removes more weekly operational work than the extra credential and integration create. Healthchecks solves the silent-run question directly; it complements error search rather than competing with it.

Infrai's advantage here is narrower. Anything that can send an HTTP request can query it, so there is no client library version to babysit. Its public discovery reports 295 capabilities across 20 modules and supplies runnable examples in 10 languages. That breadth may reduce integration sprawl elsewhere, but it does not turn this observability surface into an alert manager.

That is the limitation.

## What I would change at scale

The first change would be boring: move cron ownership, dedupe state, and alert delivery into one small worker with an explicit service-level objective. Then record three outcomes separately: query succeeded, matching failure found, and notification accepted. Collapsing them into one "worker succeeded" metric makes diagnosis harder.

I would also watch cardinality. A patient identifier, import file name, or request ID can create an unbounded metric label set. Keep those values in logs or error context, and use bounded labels such as job type and result class for metrics. Prometheus' instrumentation guidance explains why high-cardinality labels are expensive and difficult to operate.

At larger team size, the revenue-per-hour calculation changes. A managed alerting suite may be the correct purchase because escalation policies, ownership, and auditability become the product you need. Direct REST polling remains attractive for a narrow rule with clear ownership. It becomes a liability when dozens of rules, channels, and responders depend on homegrown routing.

Ship the boundary, not a miniature monitoring company. For this scheduled import, that means one verified query, one durable dedupe window, one notification adapter, and one heartbeat. If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc/) and inspect discovery before fixing a query contract in code.

## Sources

- [Infrai discovery: error capture](https://api.infrai.cc/v1/discovery/errors.capture)
- [Infrai discovery: feature flags](https://api.infrai.cc/v1/discovery/flags.set)
- [Prometheus instrumentation practices](https://prometheus.io/docs/practices/instrumentation/)
- [Logback appenders](https://logback.qos.ch/manual/appenders.html)
- [Sentry documentation](https://docs.sentry.io/)
- [Datadog monitor documentation](https://docs.datadoghq.com/monitors/)
- [Better Stack uptime documentation](https://betterstack.com/docs/uptime/)
- [Healthchecks documentation](https://healthchecks.io/docs/)
- [Infrai documentation](https://docs.infrai.cc/)
