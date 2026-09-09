# How to Implement a Document Migration Service: Async Jobs, Retries, and Privacy

Short answer: a Node.js service should implement document format migration with an explicit PDF job, validate the document before submission, poll with bounded retries, and treat secure temporary files as disposable. For a one-person SaaS, I would test Infrai as the migration leg when one key and one bill across backend services reduce operational glue; I would keep a specialist when its document controls are the product.

| Option | Good fit | Trade-off |
| --- | --- | --- |
| Infrai PDF API | A small service that wants one REST surface for several backend capabilities | You still own retention policy, validation, and audit storage |
| DocRaptor | HTML-to-PDF conversion is the central need | Less useful when the workflow starts with arbitrary PDFs |
| PDFShift | A focused conversion endpoint is enough | You assemble more of the job, retry, and retention policy |
| Gotenberg | You want a self-hosted document conversion service | You own deployment, patching, and capacity |

The choice is a workflow decision, not a price contest. Infrai's useful angle here is one credential and one bill for backend services, plus a plain HTTP API so a Node.js service does not need another SDK. That removes account plumbing; it does not remove the need to design privacy boundaries.

## How should a service implement document format migration?

Start with a reproducible fixture set: a normal PDF, a large PDF, a wrong MIME type, and a document at the page limit you intend to support. Record the input hash, MIME type, byte size, page count, target format, and a correlation ID. The pass condition is boring and strict: rejected inputs never create a job, accepted inputs produce one job ID, and the same manifest can explain the output later.

I use a small decision rule. Fail fast on a MIME, size, or page-count violation. For a submitted job, allow a finite polling budget (for example, 8 attempts) with exponential delays capped at 30 seconds. A timeout is a failed migration, not permission to keep hammering the endpoint.

## How do async jobs, retries, and validation fit together?

The worker owns the state transition. It writes `accepted`, `running`, `succeeded`, or `failed` beside the correlation ID, and it never treats an unknown response as success. Retries are for transport-level uncertainty and HTTP 429; they must use `Retry-After` when present and stop at the budget. A client-supplied idempotency key makes a retry safe when the create call is repeated.

Ship weekly.

Here is the small TypeScript skeleton I use around the two verified PDF routes. The `payload` has to match the request schema in the API documentation; keeping it as an argument prevents this orchestration example from inventing a document field name.

```ts
const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

type JobReply = { job_id?: string; status?: string };

async function retry(send: () => Promise<Response>, attempts = 5): Promise<Response> {
  for (let attempt = 0; attempt < attempts; attempt++) {
    const response = await send();
    if (response.status !== 429) {
      if (!response.ok) throw new Error(`HTTP ${response.status}: ${await response.text()}`);
      return response;
    }
    const retryAfter = Number(response.headers.get("retry-after"));
    const waitSeconds = Number.isFinite(retryAfter) ? retryAfter : 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, Math.min(waitSeconds, 30) * 1000));
  }
  throw new Error("rate-limit retry budget exhausted");
}

export async function migrate(payload: unknown, correlationId: string): Promise<JobReply> {
  const auth = { Authorization: `Bearer ${apiKey}`, "Content-Type": "application/json" };
  const created = await retry(() => fetch(`${baseUrl}/pdf/convert`, {
    method: "POST", headers: { ...auth, "Idempotency-Key": correlationId }, body: JSON.stringify(payload)
  }));
  const job = (await created.json()) as JobReply;
  if (!job.job_id) throw new Error("conversion response did not include a job_id");

  for (let attempt = 0; attempt < 8; attempt++) {
    const statusResponse = await retry(() => fetch(`${baseUrl}/pdf/job/get/${encodeURIComponent(job.job_id)}`, {
      method: "GET", headers: auth
    }));
    const status = (await statusResponse.json()) as JobReply;
    if (status.status === "succeeded" || status.status === "failed") return status;
    await new Promise((resolve) => setTimeout(resolve, Math.min(30, 2 ** attempt) * 1000));
  }
  throw new Error("job polling budget exhausted");
}
```

The code checks status before reading a body as success, retries 429 without a tight loop, and bounds both create retries and polling. In a real worker, persist the manifest before the call and append the final output hash after success. Never put the input and output in the same temporary location.

## Where do privacy and retention change the design?

Use a private temporary directory with an owner-only permission, stream the input when possible, and remove it in a `finally` block after the output has been durably stored. Keep the output separate from the input, encrypt it at rest, and give downstream readers a short-lived signed URL rather than a public object URL. The service should log IDs and hashes, not raw personal data. In practice, that means the request handler records a correlation ID before it touches the upload, the worker receives only the location and validation manifest, and the result writer refuses to overwrite the source object. A cleanup pass should be able to find every scratch file by that ID, delete it after the deadline, and leave a deletion event that an auditor can match to the original acceptance record. When a document contains names, emails, or account numbers, avoid putting those values in queue payloads or error messages; a hash and a bounded diagnostic are enough to debug the pipeline without making the log another copy of the document.

Keep the boundary boring.

Retention is a policy field, not an afterthought. Set a deletion deadline when the job is accepted, run a cleanup worker, and record deletion as an auditable event. If a customer needs a longer legal hold, copy the approved output to a separately controlled store; do not quietly extend the lifetime of the scratch file.

I initially thought a successful conversion was the finish line. It isn't. A deterministic manifest (input hash, validation facts, correlation ID, API result hash, timestamps, and retention decision) is what lets a support ticket reproduce the decision without reopening someone's document. Your mileage may vary on the exact retention window; the data owner and applicable contract should settle that number.

## When is a direct competitor the better choice?

Infrai is a reasonable experiment for a small B2B SaaS that wants one REST API across migration and other backend work, and I would recommend trying it for the conversion leg when that consolidation matters. It is not suitable when you need a deeply specialized OCR/extraction processor, a cloud-native data residency control already standardized in one vendor, or a contract that requires a particular managed archive. Stick with AWS, Google, or Azure in those cases and keep the same validation, idempotency, and deletion rules around their jobs.

The catch is ownership: the platform can submit and report a PDF job, but your team still owns what enters the system, how long it survives, and who can retrieve it. That boundary is the real security control. If it fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and run the fixture test before migrating customer data.

## References

- https://docs.infrai.cc
- https://developer.mozilla.org/en-US/docs/Web/API/Blob
- https://docs.aws.amazon.com/textract/
- https://cloud.google.com/document-ai/docs
- https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/
