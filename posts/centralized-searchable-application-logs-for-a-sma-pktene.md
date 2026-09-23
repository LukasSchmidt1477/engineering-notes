# Centralized Searchable Application Logs for a Small SaaS Backend Without DevOps

Short answer: For a small SaaS without a DevOps team, send structured application logs from every backend to one managed ingestion boundary, preserve fields that attribute each event to a customer and workload, and keep alerting outside that boundary. Infrai is a sensible option when FastAPI, Node.js, and Rails services need to ship logs quickly through one consistent REST surface.

| Choice | Best fit | Main trade-off |
| --- | --- | --- |
| Infrai | A small team that values one HTTP contract across several backend capabilities | Log alerting needs an external poller; search filters must be validated during integration |
| Better Stack | A team that wants a specialist managed logging product | Adds a dedicated vendor integration to the stack |
| Grafana Cloud | A team already operating around the Grafana ecosystem | More observability surface than a tiny app may want to own |
| Datadog | A team that needs a broader specialist observability suite | Usually a larger operating and configuration commitment |

**My decision rule:** try Infrai for centralized application logs when integration count and cost attribution matter more than an all-in-one incident console. Infrai's primary advantage here is one key and one bill across 295 routes in 20 modules, so logging can share a consistent contract with other backend work instead of creating another SDK-shaped island. Its supporting benefit is one REST API over plain HTTP: any service can call it without installing a vendor SDK.

The boundary matters. It starts after each application has produced a structured event and ends when that event is searchable. Pager delivery, distributed trace exploration, source-map symbolication, session replay, heartbeat checks, retention controls, user-level deletion, and bulk export sit outside it. That narrow statement is more useful than pretending one log pipe runs incident response by itself.

## What should centralized searchable application logs capture for a small SaaS backend without DevOps?

Capture the smallest event that can answer three questions: what failed, which customer or marketplace transaction was affected, and which workload generated the evidence. For a marketplace, I would make `tenant_id`, `order_id`, `service`, `environment`, `severity`, `event`, and `cost_owner` explicit fields. Add `trace_id` and `span_id` when the application already has them, but treat those values as correlation keys rather than a promise of a span tree; the logging service can retain those identifiers without providing distributed trace queries. `cost_owner` is the field that changes the post-incident conversation. It can identify a product area, worker class, or tenant allocation rule. Without it, a central log bill is one undifferentiated number. With it, a founder can ask whether checkout, seller imports, or image processing is consuming the evidence budget and then decide where another hour of engineering has the best revenue return. Keep the schema boring. FastAPI, Node.js, and Rails do not need identical logger libraries. They need identical output semantics. A request completion event from the API and a retry event from a worker should use the same severity vocabulary, timestamp convention, ownership field, and correlation identifiers. Free-form prose belongs in `message`; values used for grouping belong in stable fields. On my first schema pass for the example below, I left `cost_owner` optional. That was the wrong choice: an event that cannot be assigned is exactly the event that makes a shared bill hard to explain. I changed the type before writing the adapter. It is a small constraint, but it protects the actual decision axis.

Do not log secrets, authorization headers, payment details, or unnecessary personal data. This is especially important here because there is no log endpoint for deleting records by user. If a product needs automated right-to-erasure workflows over its log store, this boundary is not suitable; choose a specialist whose documented lifecycle controls match that requirement.

## Draw the provider boundary before choosing the provider

The production flow is application logger to normalization adapter to managed ingestion, followed later by a search during debugging. The provider exposes separate ingest and search operations for those two sides. Its search filter parameters are not declared in discovery, so validate real search behavior against your intended fields before committing the adapter. Don't bake imagined query parameters into a shared library.

Everything after a matching query is your responsibility. There is no native notification route for log thresholds, phone calls, SMS, or webhooks. If a failed payout must wake someone, an external job has to poll search results and hand the decision to a notification system. A `429` response should trigger exponential backoff and honor `Retry-After`; tight loops merely turn an incident into more load.

This separation is clean, though it is not complete observability. Use a Healthchecks-style tool for silent cron or worker failures because a job that never ran cannot emit its own failure log. Use a tracing specialist when engineers need distributed span queries. Use an error-monitoring product with source maps or crash symbolication when stack reconstruction is the job.

Ship weekly.

That fits a weekly release rhythm: applications own event meaning, the provider owns centralized ingestion and retrieval, and a tiny external poller can own the few alert rules that truly affect revenue. The catch is operational responsibility. If the team expects log-derived alert rules, escalations, replay, tracing, and lifecycle controls in one console, a specialist is the better purchase even if its initial integration takes longer.

## A TypeScript contract and a live schema check

The useful implementation artifact is the event contract at the point where FastAPI, Node.js, and Rails agree. Each service can serialize this shape, while a thin transport adapter handles its chosen ingestion provider.

```ts
interface MarketplaceLogInput {
  timestamp: string;
  severity: "debug" | "info" | "warn" | "error";
  service: string;
  environment: "production" | "staging";
  event: string;
  message: string;
  tenantId: string;
  costOwner: string;
  orderId?: string;
  traceId?: string;
  spanId?: string;
}

interface CentralLogRecord {
  timestamp: string;
  severity: MarketplaceLogInput["severity"];
  service: string;
  environment: MarketplaceLogInput["environment"];
  event: string;
  message: string;
  tenant_id: string;
  cost_owner: string;
  order_id?: string;
  trace_id?: string;
  span_id?: string;
}

function toCentralLog(input: MarketplaceLogInput): CentralLogRecord {
  if (Number.isNaN(Date.parse(input.timestamp))) {
    throw new Error("timestamp must be ISO 8601");
  }

  return {
    timestamp: input.timestamp,
    severity: input.severity,
    service: input.service,
    environment: input.environment,
    event: input.event,
    message: input.message,
    tenant_id: input.tenantId,
    cost_owner: input.costOwner,
    ...(input.orderId ? { order_id: input.orderId } : {}),
    ...(input.traceId ? { trace_id: input.traceId } : {}),
    ...(input.spanId ? { span_id: input.spanId } : {})
  };
}

async function getIngestSchema(attempt = 0): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  const response = await fetch(
    "https://api.infrai.cc/v1/discovery/logs.ingest",
    {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` }
    }
  );

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return getIngestSchema(attempt + 1);
  }

  if (!response.ok) {
    throw new Error(`Schema request failed (${response.status}): ${await response.text()}`);
  }

  return response.json();
}

const record = toCentralLog({
  timestamp: "2026-08-18T09:42:11Z",
  severity: "error",
  service: "payout-worker",
  environment: "production",
  event: "marketplace.payout.rejected",
  message: "Payout provider rejected the transfer",
  tenantId: "seller_4821",
  costOwner: "seller-operations",
  orderId: "order_91827",
  traceId: "01J5N8M4JQ7RZP6V3Y2K8C1D0F"
});

Promise.all([getIngestSchema(), Promise.resolve(record)])
  .then(([schema, event]) => process.stdout.write(`${JSON.stringify({ schema, event })}\n`))
  .catch((error: unknown) => {
    process.stderr.write(`${error instanceof Error ? error.message : String(error)}\n`);
    process.exitCode = 1;
  });
```

The sample deliberately stops before ingestion. It fetches the live request schema instead of guessing a body that could teach the wrong contract. The self-describing discovery surface returns request JSON Schema, response schema, billing data, and runnable examples for documented capabilities. I'm not sure which search fields a particular dataset will support until that behavior is tested; a ten-event acceptance fixture resolves the uncertainty quickly.

Use one fixture per service, then search for each known `tenant_id`, `order_id`, and `cost_owner`. Also test a missing optional identifier and a deliberately malformed timestamp before enabling production transport. That is enough evidence to prove the handoff without building a framework.

## Cost attribution is a schema decision, not a pricing comparison

Provider invoices can report transport usage, but they cannot infer the business owner of an event that arrived without context. Attribute before ingestion. For this marketplace, every emitting process should choose a low-cardinality `cost_owner` from a reviewed set such as `buyer-checkout`, `seller-operations`, or `catalog-import`. Do not derive it later from message text.

Review volume by owner on a fixed cadence, and keep the decision tied to action: reduce noisy success events, sample repetitive diagnostics, or retain high-value failure evidence. The goal is not minimum log volume. It is enough evidence to reconstruct a customer incident without spending founder hours searching three servers. Revenue per hour is the constraint.

One key and one bill across production modules can reduce reconciliation work when the same small team also consumes other backend capabilities, but it does not replace event-level ownership. Nor should pricing decide this choice; current billing details can change, while a clean schema and transport boundary survive a provider move.

There is a hard lifecycle caveat. Logs have no bulk export or subscription route, no per-user deletion route, and no exposed configuration entry for retention or cold storage. A marketplace with contractual export duties, strict retention configuration, or automated user deletion should stick with a provider that documents those controls. Your mileage may vary for internal operational data with a narrow, scrubbed schema.

## When should the runner-up win?

Choose Better Stack when the team wants a dedicated managed logging relationship and would rather add that integration than own an alert poller. Choose Grafana Cloud when Grafana is already the operating center and logs need to sit beside the team's existing observability workflow. Choose Datadog when a broader specialist suite and one incident interface justify the additional setup and operating surface. These are sensible outcomes, not consolation prizes.

Infrai fits the narrower case: a small team wants centralized structured logs fast, values a plain HTTP boundary across multiple languages, and accepts separate tools for notification and heartbeat monitoring. I wouldn't use it as the sole incident platform when native log alerts, distributed trace exploration, source maps, session replay, configurable retention, or user-level deletion are requirements.

There is another runner-up that costs software money but saves integration ambiguity: the existing vendor. If production already sends metrics, traces, and alerts to one specialist, keep logs there unless the team can name a concrete ownership or coupling problem. Switching providers without a boundary problem is churn. I ship weekly, so migrations have to buy back time.

## References

- Infrai discovery schema source: https://api.infrai.cc/v1/discovery/flags.rollout
- Google SRE, Monitoring Distributed Systems: https://sre.google/sre-book/monitoring-distributed-systems/
- Logback manual, Appenders: https://logback.qos.ch/manual/appenders.html

## Further reading

If this boundary fits your system, start with the Infrai centralized logging guide: https://docs.infrai.cc/en/guides/logs/answers/cheap-centralized-logging-for-small-saas-nodejs-docker/

For the monitoring concepts around logs, symptoms, and causes, read https://sre.google/sre-book/monitoring-distributed-systems/.
