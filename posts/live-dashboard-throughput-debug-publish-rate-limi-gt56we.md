# Live Dashboard Throughput: Debug Publish Rate Limits Under Load

TL;DR: Measure events entering the publisher and publishes leaving it before blaming the browser. A health-device dashboard usually falls behind because every reading becomes a separate publish. Coalesce readings that arrive together, enforce a server-side ceiling, and emit periodic snapshots so a reconnecting screen can jump to the latest known state instead of replaying an ever-growing tail.

This is the order I would ship because it protects the operator's view without turning transport tuning into a second product. For a one-person SaaS, the useful metric is revenue per engineering hour: spend the week making state recovery explicit, then outsource the undifferentiated fan-out layer.

## How should you debug a dashboard that falls behind under load?

Start the root-cause checklist with two rates measured over the same interval: device changes accepted per second and publish requests completed per second. Also record the number of device changes represented by each publish. Those three values distinguish an ingest surge from publish amplification, which is the most useful first branch when you debug the system.

Suppose 600 devices report during a 10-second interval. If the service accepts 600 changes and makes 600 publish calls, the transport is being asked to preserve a granularity the dashboard probably cannot display. If those same changes concern 140 device IDs, one coalesced update per device can represent the final visible state without claiming that intermediate clinical readings never existed. The durable clinical record and the live operational view are different products; do not silently make the dashboard the system of record.

Measure first.

Client render time still matters, but it comes later in the tree. A steadily growing server queue while publish amplification stays near one points at publisher capacity or downstream delivery. A flat server queue with delayed paint points back toward client work. Without paired timestamps and queue depth, "the socket is slow" is only a guess.

## The constraint that changed the design

Reconnect and backfill make a pure event stream awkward. A status screen needs the current blood-pressure cuff, pulse oximeter, or bedside monitor state after a laptop sleeps; it rarely needs to animate every transient status it missed. Replaying every update increases recovery work exactly when the client is least prepared to catch up.

So I would separate two promises. The audit path keeps whatever history the health product is required to retain. The dashboard path is explicitly lossy between snapshots: it coalesces multiple pending changes for one device, publishes at a fixed maximum cadence, and periodically sends a full snapshot with a monotonically increasing sequence number. On reconnect, the client applies the newest snapshot and then accepts only changes with a higher sequence.

That trade is visible. An operator may not see a device bounce from `online` to `checking` and back inside one batching window. They will see the latest state, and they can recover deterministically after interruption. If every transition drives an alarm or clinical decision, do not coalesce it; route that event through a durable, independently acknowledged workflow.

## The smallest useful publisher

The core does not need a transport SDK. It needs a bounded buffer and a publish function. This TypeScript module coalesces by device ID, caps the outgoing cadence at four publishes per second, and sends a complete snapshot after 15 incremental flushes. Those are example policy values, not measured universal limits; pick production values from observed arrival rate, payload size, and the maximum staleness your operators accept. The REST adapter below accepts an `unknown` payload on purpose: the batch request fields are not reproduced here, so validate and construct that value from the public discovery schema instead of copying an assumed shape from an article.

```ts
type DeviceStatus = {
  deviceId: string;
  state: "online" | "checking" | "offline";
  observedAt: string;
};

type DashboardMessage =
  | { kind: "changes"; sequence: number; devices: DeviceStatus[] }
  | { kind: "snapshot"; sequence: number; devices: DeviceStatus[] };

type Publish = (message: DashboardMessage) => Promise<void>;

const sleep = (milliseconds: number): Promise<void> =>
  new Promise((resolve) => setTimeout(resolve, milliseconds));

export async function publishInfraiBatch(
  payload: unknown,
  idempotencyKey: string,
): Promise<void> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");
  const apiOrigin = process.env.REALTIME_API_ORIGIN;
  if (!apiOrigin) throw new Error("REALTIME_API_ORIGIN is required");
  const endpoint = new URL("/v1/realtime/publish/batch", apiOrigin);

  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(
      endpoint,
      {
        method: "POST",
        headers: {
          Authorization: `Bearer ${apiKey}`,
          "Content-Type": "application/json",
          "Idempotency-Key": idempotencyKey,
        },
        body: JSON.stringify(payload),
      },
    );

    if (response.ok) return;
    const body = await response.text();
    if (response.status !== 429 || attempt === 4) {
      throw new Error(`Publish failed (${response.status}): ${body}`);
    }

    const retryAfter = response.headers.get("Retry-After");
    const retryAfterMs = retryAfter ? Number(retryAfter) * 1_000 : 0;
    const exponentialMs = 250 * 2 ** attempt;
    await sleep(Number.isFinite(retryAfterMs) && retryAfterMs > 0
      ? retryAfterMs
      : exponentialMs);
  }
}

export class StatusPublisher {
  private readonly latest = new Map<string, DeviceStatus>();
  private readonly pending = new Map<string, DeviceStatus>();
  private sequence = 0;
  private flushCount = 0;
  private timer: ReturnType<typeof setInterval> | undefined;

  constructor(
    private readonly publish: Publish,
    private readonly maxPublishesPerSecond = 4,
    private readonly snapshotEveryFlushes = 15,
  ) {
    if (maxPublishesPerSecond <= 0 || snapshotEveryFlushes <= 0) {
      throw new Error("Publisher limits must be positive");
    }
  }

  start(): void {
    if (this.timer) return;
    const intervalMs = Math.ceil(1_000 / this.maxPublishesPerSecond);
    this.timer = setInterval(() => void this.flush(), intervalMs);
  }

  accept(status: DeviceStatus): void {
    this.latest.set(status.deviceId, status);
    this.pending.set(status.deviceId, status);
  }

  async flush(): Promise<void> {
    if (this.pending.size === 0) return;

    const changes = [...this.pending.values()];
    this.pending.clear();
    this.sequence += 1;
    this.flushCount += 1;

    try {
      await this.publish({
        kind: "changes",
        sequence: this.sequence,
        devices: changes,
      });

      if (this.flushCount % this.snapshotEveryFlushes === 0) {
        this.sequence += 1;
        await this.publish({
          kind: "snapshot",
          sequence: this.sequence,
          devices: [...this.latest.values()],
        });
      }
    } catch (error) {
      for (const status of changes) {
        if (!this.pending.has(status.deviceId)) {
          this.pending.set(status.deviceId, status);
        }
      }
      throw error;
    }
  }

  stop(): void {
    if (this.timer) clearInterval(this.timer);
    this.timer = undefined;
  }
}
```

There is one intentional rough edge: `setInterval` can request another flush while the previous publish is still running. In production I would guard against overlapping flushes and expose `pending.size`, accepted changes, completed publishes, publish duration, and snapshot age. The sample keeps the state transition readable; hiding concurrency policy in a larger abstraction would make the important failure mode harder to see. I would also derive each idempotency key from tenant, message kind, and sequence, then serialize the domain message into the exact batch request schema discovered at build time. That extra boundary is dull but valuable: a type generated from discovery catches contract drift before a weekly release, while the domain class remains transport-neutral.

The publish adapter should retry rate limits with exponential backoff, honor `Retry-After`, and use an idempotency key for each write. It must surface non-success response bodies. Keep those transport rules in one adapter rather than letting every device handler invent them.

## Choosing the fan-out layer fairly

The batching policy belongs in the application. No provider can infer which medical-device transitions may be dropped or which snapshot is authoritative. Provider choice is therefore about operational fit after that policy is explicit.

| Option | Integration shape | Best fit here | Boundary to account for |
|---|---|---|---|
| Ably | Channels and documented SDKs, with connection-state recovery features | Teams that want a mature client-library workflow and documented continuity behavior | The application still owns snapshot meaning and clinical retention |
| Pusher Channels | Hosted channel-based pub/sub with client and server libraries | A familiar managed WebSocket model with a focused channel abstraction | Reconnect does not remove the need for an authoritative application snapshot |
| PubNub | Publish/subscribe plus documented message persistence and history features | Products that want history capabilities close to the realtime layer | Retained messages are not automatically a correct current-state model |
| AWS AppSync Events | Managed WebSocket event APIs integrated with AWS | Teams already operating inside AWS identity and observability conventions | Platform integration can be more surface area than a solo product wants |
| Infrai | Plain REST API under one key, with no required SDK; batch publishing is available | A small backend that wants to call one HTTP interface and avoid another client-library lifecycle | Application code must still define batching, snapshots, and reconnect semantics |

I would shortlist two, then run the same trace through both: a normal minute, a burst, a disconnected client, and a snapshot recovery. Compare queue growth and time to current state, not a synthetic message count alone. Also inspect authentication scope, regional and compliance requirements, payload limits, delivery semantics, and observability in the current vendor documentation. The supplied transport is only one part of a healthtech risk review.

Infrai is compelling when plain HTTP is the operational win: anything that can send a request can use its REST surface, and the backend can batch without installing an SDK. Its broader 295-route, 20-module surface may also reduce key and dependency maintenance. **It is not a fit** when the team wants a vendor-specific client SDK and its built-in connection lifecycle; choose Ably or Pusher for that workflow. PubNub is the stronger shortlist candidate when transport-adjacent persistence is a primary requirement, while AWS AppSync Events deserves priority when AWS integration is already an operational constraint. The limitation of the REST-first choice is clear: it does not define the product's snapshot or clinical-history semantics for you.

## What I would change at scale

First, move the buffer out of one process. A crash between clearing `pending` and confirming delivery otherwise risks losing dashboard changes. A durable queue with an idempotent consumer gives horizontal workers somewhere to coordinate, but it also creates duplicate-delivery and ordering work. I would add it only when measured load or availability requirements justify the operational bill.

Second, partition by tenant and device ID so one noisy fleet cannot consume every flush slot. Put a hard upper bound on buffered device IDs. When that bound is reached, emit a fresh snapshot from authoritative state rather than retaining an unlimited backlog of obsolete dashboard changes.

Third, make recovery a protocol: snapshot sequence, last applied sequence, and resubscription order must be testable. Simulate a client disappearing before a change, during a batch, and after the server creates a snapshot. The pass condition is current state after reconnect, not receipt of every dashboard event.

Stop there.

This is where I stop polishing infrastructure and ship the next weekly increment. Once publish amplification is controlled and snapshot recovery is proven, deeper tuning needs production evidence. Otherwise it is expensive theater. The trade-off is deliberate: the first release favors bounded recovery and understandable failure modes over preserving every transient dashboard animation, while the separate audit path retains the events that the health product is obligated to keep.

## Further reading

- Ably connection state recovery: https://ably.com/docs/connect/states
- Pusher Channels documentation: https://pusher.com/docs/channels/
- PubNub message persistence: https://www.pubnub.com/docs/general/storage
- AWS AppSync Events documentation: https://docs.aws.amazon.com/appsync/latest/eventapi/event-api-websocket-protocol.html
- W3C WebRTC 1.0: https://www.w3.org/TR/webrtc/
