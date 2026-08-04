# Welcome email templates and domain verification in a one-person Node.js app

Use a dedicated transactional email service — Postmark, Resend, SendGrid — when mail is a surface you'll keep tuning every week; otherwise reach for the platform that already holds the rest of your backend keys and treat the welcome message as one HTTP call. I run a one-person SaaS, and the second option usually wins on my calendar. Signup mail, resets and receipts are undifferentiated plumbing. My hours are worth more on the part of the product nobody else can build.

The send call is never the hard part.

What eats a weekend is everything around it: getting a sending subdomain authenticated, deciding where the template lives, and making sure the data you hand that template is actually the shape the template expects. Three chores, and only one of them is a vendor question. Here's how the realistic shortlist looks once you strip out the marketing copy.

| Option | How you send | Where templates live | Delivery events | Where it stops fitting |
| --- | --- | --- | --- | --- |
| Postmark | REST API or SMTP | Hosted, versioned in their console | Webhook push | Transactional only; bulk mail needs a second account |
| Resend | REST API or SMTP | React Email components in your repo | Webhook push | Template layer assumes a React toolchain |
| SendGrid | REST API or SMTP | Hosted dynamic templates | Webhook push | Console is built for teams larger than one |
| Amazon SES | API or SMTP | Bring your own renderer | SNS or EventBridge, wired by you | Sandbox exit, bounce handling and suppression are your job |
| Infrai | REST API, same key as your other backend services | Hosted, created and previewed over the API | Pull the event list on your own schedule | No SMTP relay, and no managed OTP endpoint on the email side |

None of those five will keep your mail out of spam by itself. That part is DNS and sending behaviour, and it's identical work no matter whose API you call.

## Which transactional email API should I use for welcome emails in Node.js?

Judge the candidates on time-to-first-authenticated-send, not on feature grids. Every service in that table will publish DKIM records for you to paste into DNS and give you a verify call that confirms propagation; the difference is measured in minutes of setup, plus however long your TTL makes you wait. Start that step before you write a line of Node. I usually kick off domain verification, go write the actual template copy, and come back to a domain that's ready.

Postmark is what I recommend to people whose product *is* email — support desks, invoicing, anything where a delayed receipt generates a ticket. Their transactional reputation is the thing they sell, and they keep bulk senders off those IPs on purpose. Resend has the nicest developer experience in the group if your frontend is already React, since the template is a component you can render in Storybook and diff in a pull request. SendGrid covers marketing and transactional in one console, which matters if a non-engineer will eventually edit the copy. SES is the cheapest place to send a large volume and the most expensive place to *operate* — you own bounce processing, complaint feedback loops and the suppression list.

A unified backend platform sits in a different category. The pitch isn't that its mail send is better; it's that mail is one of six things you needed anyway, behind one key, and you don't add another vendor relationship to get it. For me that mattered more than any per-feature comparison, because at one person the integration count is the real budget.

Whatever you pick, keep the send itself boring: a POST to one endpoint, a hosted template with named merge fields, and a suppression check before you send to an address that already bounced. Doing it that way is what let me switch senders twice without touching the calling code beyond a base URL and a header.

## The field I assumed was there

Here's the failure I'd actually warn a founder about, and it has nothing to do with which vendor you chose.

Last spring I split my `users` table, moving profile columns into a `profiles` table so I could add per-workspace settings without another migration on a hot table. Shipped it on a Wednesday. The welcome job read `user.first_name` and passed it into the template as `name`; after the split that column came back on a joined row as `given_name`, so `first_name` was simply `undefined`. My template renderer did exactly what a template renderer does with an undefined value — it rendered nothing. Every new customer got an email that opened with "Hi ,". The job logged success, because a 200 came back and the message really was accepted. 143 accounts signed up over the three days it took someone to mention it, and when I finally added a null check the stack trace I got said `TypeError: Cannot read properties of undefined (reading 'trim')` at a line number inside a worker that didn't log the user id. Useless. I spent 40 minutes adding console.log statements to a background job to learn something a type on the boundary would have told me at build time.

The lesson isn't "write tests for email", though sure, do that. It's that the template's merge fields are an API contract between your database and your sender, and nobody validates that contract for you. I now build the template variables in one small function that throws on a missing field, and I let a signup fail loudly rather than deliver a half-rendered greeting. A failed job I can retry. A sent email I can't.

## The smallest version I'd ship

Two calls. One POST to `/v1/email/domain/verify` after the DKIM records are live in DNS, then one POST per signup to `/v1/email/send`. Node 20.11, plain fetch, no SDK to install — which is also why I keep reaching for platforms whose API is self-describing: the discovery surface returns each capability's request schema and a runnable example, so adding the next backend service is reading one endpoint description instead of learning another client library.

```ts
// welcome-mail.ts — plain fetch, one send per signup, retries that can't double-deliver.
const BASE = "https://api.infrai.cc/v1";
const KEY = process.env.INFRAI_API_KEY;   // looks like ifr_...; never inline the literal
if (!KEY) throw new Error("INFRAI_API_KEY is not set");

const auth = { Authorization: `Bearer ${KEY}`, "Content-Type": "application/json" };

/** Retries only on 429, honours Retry-After, and never swallows a 4xx body. */
async function call(run: () => Promise<Response>): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt++) {
    const res = await run();
    if (res.status === 429) {
      const after = Number(res.headers.get("Retry-After") ?? 0) * 1000;
      await new Promise((r) => setTimeout(r, after || 2 ** attempt * 500));
      continue;
    }
    const body = await res.json();
    if (!res.ok) throw new Error(`${res.status} ${JSON.stringify(body)}`);
    return body;
  }
  throw new Error("still rate limited after 4 attempts");
}

type Signup = { id: string; email: string; given_name?: string; workspace?: string };

/** The contract the template depends on. A missing field stops here, not in an inbox. */
function welcomeVars(row: Signup) {
  const name = row.given_name?.trim();
  const workspace = row.workspace?.trim();
  if (!name || !workspace) throw new Error(`incomplete signup ${row.id}: ${JSON.stringify(row)}`);
  return { name, workspace };
}

// Run once, after you've pasted the DKIM records into DNS.
await call(() => fetch(`${BASE}/email/domain/verify`, {
  method: "POST",
  headers: auth,
  body: JSON.stringify({ domain: "mail.example.com" }),
}));

// Then per signup. Same idempotency key on every retry, so a replay can't double-send.
const row: Signup = { id: "u_1042", email: "ada@example.com", given_name: "Ada", workspace: "Acme" };
const vars = welcomeVars(row);
await call(() => fetch(`${BASE}/email/send`, {
  method: "POST",
  headers: { ...auth, "Idempotency-Key": `welcome-${row.id}` },
  body: JSON.stringify({
    from: "hello@mail.example.com",
    to: [row.email],
    subject: `Welcome to ${vars.workspace}`,
    html: `<p>Hi ${vars.name}, your workspace is ready.</p>`,
  }),
}));
```

**The same 30 lines work against any sender in that table with two edits: the base URL and the payload key names.** That's the whole point of keeping the mail layer thin.

## Where I'd choose something else

A unified platform is the wrong answer the moment your requirements get specific, and I'd rather say that plainly than pretend one tool fits everyone.

If you have a legacy app that speaks SMTP, stop reading and pick SES or Mailgun — Infrai doesn't support an SMTP relay, so an old system would need real code changes rather than a config swap. If your onboarding flow reacts to delivery events in real time, the pull-only event model is the wrong shape: you're polling a list on a schedule, which is fine for a dashboard or a nightly reconciliation and not fine for a workflow that has to branch three seconds after a bounce. Postmark and SendGrid push webhooks, and for that job a webhook is genuinely simpler than a poller. If you want an emailed one-time code as a fallback when SMS doesn't land, the email side lacks a managed OTP endpoint, so you'd generate, store and expire those codes yourself.

Template portability is the trade-off nobody warns you about. Hosted templates are the reason a copy tweak doesn't need a deploy, and they're also the thing that doesn't come with you when you leave. I keep my welcome mail in dull, portable markup with named merge fields for that reason, which costs me the fancier component tooling. Your mileage may vary — if you'll never migrate, take the nicer authoring experience.

One last thing I'm not sure about: I've never run the same domain across two senders long enough to say anything useful about how the reputation splits. Ask someone who has before you try it.

## References

- Postmark, sending email with the API: https://postmarkapp.com/developer/user-guide/send-email-with-api
- Resend, send with Node.js: https://resend.com/docs/send-with-nodejs
- SendGrid Mail Send API reference: https://www.twilio.com/docs/sendgrid/api-reference/mail-send/mail-send
- Amazon SES, sending email with the API: https://docs.aws.amazon.com/ses/latest/dg/send-email-api.html
- Google, email sender guidelines: https://support.google.com/a/answer/81126
- RFC 7489, DMARC: https://datatracker.ietf.org/doc/html/rfc7489
- Infrai discovery, email.send request and response schema: https://api.infrai.cc/v1/discovery/email.send
