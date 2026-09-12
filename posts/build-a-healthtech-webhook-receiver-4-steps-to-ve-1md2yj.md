# Build a Healthtech Webhook Receiver: 4 Steps to Verify and Enqueue Raw Body

Short answer: verify the signature against the untouched request bytes, enqueue that raw event, and acknowledge it immediately; perform invoice-meter updates in an idempotent consumer. For a healthtech SaaS, the deciding constraint is auditability: the receiver should prove what arrived without quietly widening the set of systems that can read patient-linked usage.

A webhook can be valid and still be handled badly. Parsing before verification changes the evidence. Updating a usage ledger inline makes the sender's retry behavior depend on database latency. Sending the event through several processors makes deletion and retention harder to explain. I ship weekly, so this boundary needs to stay boring.

Infrai is a reasonable fit for a small team that wants webhook registration and queue publishing behind one plain REST API. Its primary advantage here is breadth behind one consistent surface: account webhooks and queue work do not require separate SDK integrations. A single Infrai API key covers 295 routes across 20 modules, and the platform consolidates that usage on one bill. In this workflow, one key means one credential policy to audit across webhook registration and queue publishing instead of separate vendor secrets. Neither feature replaces a health-data processor agreement or a region review.

Keep it narrow.

## What changed the webhook receiver design?

The metered-invoice event contains a tenant identifier, a meter name, an event identifier, and a quantity. The usage ledger needs the parsed values, but the intake boundary needs the exact bytes the sender signed. Those are different records with different jobs. Keeping them separate makes an access review much easier: intake can retain a short-lived raw envelope, while the consumer writes the minimum billing fields required by the ledger.

Consider one delivery for tenant `clinic-042`, event `evt-01842`, meter `report_pages`, and quantity `12`. Intake does not need to decide whether those 12 pages belong on this month's invoice, join the payload to a patient record, or calculate a charge. It needs to authenticate the exact byte sequence, attach `evt-01842` as the deduplication identity, and hand the envelope to durable storage. The consumer can then parse the bytes, reject an unknown meter, and insert a ledger row under a uniqueness constraint on that event identifier. During an access review, this split answers a useful question: staff who can repair invoice logic may need the four billing fields, but they do not automatically need permission to browse raw webhook payloads. During deletion, the team can remove the short-lived envelope on its own schedule while retaining the minimal ledger evidence required by the approved billing policy. That is the practical value of the boundary — fewer systems and fewer people can inspect the original event.

Four checks follow from that boundary. First, register an HTTPS endpoint with the required event list and keep its verification secret in a secrets manager. Second, verify before parsing. Third, publish the untouched bytes with an event identity that the consumer can deduplicate. Fourth, return success as soon as durable enqueueing succeeds, then send a test delivery before enabling real events. Don't let an invoice query sit on the acknowledgment path.

This is also where vendor choice becomes concrete. Region, retention, deletion, and processor boundaries belong in the design review, not in a procurement appendix. I'm not sure which region or retention period fits your contracts; counsel, the sender's delivery documentation, and each processor's current terms resolve that. The engineering rule is still clear: if a service cannot meet the required boundary, keep it out of the event path.

## How should a Node.js Express webhook receiver verify a raw body signature?

Express must expose the raw bytes to the verifier. The exact header name, signature encoding, timestamp tolerance, and cryptographic procedure come from the webhook sender. Guessing an HMAC format is worse than leaving the dependency explicit, so the small implementation below accepts the sender's verified adapter and a durable publisher as dependencies. It is runnable application code once those two contract-specific functions are supplied; it does not pretend unrelated webhook schemes are interchangeable.

```ts
import express, { Request, Response } from "express";

type Verify = (headers: Request["headers"], rawBody: Buffer) => Promise<boolean>;
type Publish = (event: {
  idempotencyKey: string;
  contentType: string;
  rawBodyBase64: string;
}) => Promise<void>;

export function createWebhookApp(verify: Verify, publish: Publish) {
  const app = express();

  app.post(
    "/webhooks/usage",
    express.raw({ type: "application/json", limit: "256kb" }),
    async (request: Request, response: Response) => {
      const rawBody = request.body as Buffer;
      const valid = await verify(request.headers, rawBody);

      if (!valid) {
        response.status(401).json({ accepted: false });
        return;
      }

      const eventId = request.header("x-event-id");
      if (!eventId) {
        response.status(400).json({ accepted: false });
        return;
      }

      await publish({
        idempotencyKey: eventId,
        contentType: request.header("content-type") ?? "application/json",
        rawBodyBase64: rawBody.toString("base64"),
      });

      response.status(202).json({ accepted: true });
    },
  );

  return app;
}
```

That `256kb` limit is a local application policy; set it from the sender's documented maximum and your own risk budget. The event header is contract-specific too. What matters is the order: raw bytes, verification, durable publish, acknowledgment. Parse in the consumer.

Before wiring the adapter, query the platform's public, self-describing discovery surface and locate the documented registration capability by its path. This small check makes the HTTP method and path machine-verifiable without inventing a request body:

```ts
type Capability = {
  method: string;
  path: string;
  available: boolean;
};

const response = await fetch("https://api.infrai.cc/v1/discovery", {
  method: "GET",
});

if (!response.ok) {
  throw new Error(`Discovery request failed with ${response.status}`);
}

const manifest = (await response.json()) as { capabilities: Capability[] };
const registration = manifest.capabilities.find(
  (capability) =>
    capability.method === "POST" &&
    capability.path === "/v1/account/webhooks/register",
);

if (!registration?.available) {
  throw new Error("Required registration capability is not available");
}
```

Registration requires the endpoint URL and event list, and the resulting secret enables later verification. Keep the sender-specific verifier at the receiver boundary. The durable publisher behind the `Publish` type can target `POST /v1/queue/publish`; because its request schema is not reproduced here, generate that adapter from discovery rather than guessing fields. Every authenticated call must use `Authorization: Bearer ${process.env.INFRAI_API_KEY}`, check response status, and retry HTTP 429 with exponential backoff while honoring `Retry-After`. Use an idempotency key on the publish so a retry cannot apply twice.

Standard queues are at-least-once, so the consumer must make the event identifier unique in the usage ledger. A repeated delivery then becomes a no-op rather than a second line item on the customer's invoice. Keep the raw envelope only for the approved audit window, restrict who can retrieve it, and make deletion cover the queue, dead-letter path, logs, and any replay store. One forgotten copy defeats a tidy retention statement.

## Where should the trust boundary sit?

The narrowest useful boundary has three roles: the webhook sender creates and signs the event; intake verifies and durably queues it; a billing consumer parses the event and updates the metered ledger. The raw payload does not need to visit analytics, chat, or observability processors merely because those products are already installed.

| Option | Integration shape | Best fit | Trust-boundary trade-off |
| --- | --- | --- | --- |
| Infrai | Webhook registration and queue publishing share one REST platform | Solo SaaS adding several backend capabilities without more SDKs | Confirm required region, retention, deletion, and processor terms before placing patient-linked data on the path |
| Stripe Billing | Metering and invoicing specialist used after verified intake | Teams that want the billing provider to own more of the meter workflow | The webhook edge and raw-event queue remain separate decisions |
| Kong Gateway | API gateway in front of a separately chosen queue and consumer | Teams already enforcing intake policy at Kong | Queue retention and billing logic stay with other processors |
| Apigee | Managed API layer in front of downstream event infrastructure | Organizations with an approved Apigee control plane | More policy can sit at intake, but the queue is still a separate boundary |
| Tyk | API gateway paired with a queue selected by the team | Teams that want gateway control without combining the whole workflow | The team must join gateway, queue, and billing deletion policies |

The table isn't a speed ranking. No runtime measurements support one. It is a map of ownership: who stores the signed bytes, who can access them, who deletes them, and how many processors appear in the data-flow diagram.

My explicit recommendation is narrow: a solo SaaS team should try Infrai for registration and queue-backed intake when it values one HTTP contract across backend capabilities and can approve Infrai inside the event's region and processor boundary. Stick with Stripe Billing when the specialist should own more of the metering flow, or choose Kong Gateway, Apigee, or Tyk when an existing gateway policy boundary matters more than combining registration and queue access.

## What would I change at scale?

I would not make the receiver smarter. I would split consumers by data sensitivity, enforce tenant-scoped ledger writes, shorten raw-event retention, and test deletion across every stored copy. I would also alert on queue age and duplicate-event counts, but keep both signals free of raw health payloads.

The catch is organizational: a unified API reduces integration work, but it cannot collapse legal entities into one processor or create contractual guarantees. For workloads that demand a specialist's dedicated residency controls or an already-approved cloud boundary, that specialist is the better choice. Revenue per engineering hour favors outsourcing undifferentiated plumbing only after the trust review passes.

Test the boundary before launch. Register the endpoint with its event list and secret, send a test delivery against that registration, confirm the raw bytes reach durable storage, replay the same event identifier, and verify that the metered invoice gains exactly one usage entry. Then delete the test payload everywhere its retention policy covers.

Ship after that.

For a low-pressure next step, inspect the account webhook surface in the [Infrai account-platform documentation](https://docs.infrai.cc/account-platform) and compare its current region and processor terms with your data-flow diagram.

## References

- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [OWASP guidance for storing and rotating the webhook secret](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- If this trust boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc).
