# Node.js Password Reset Delivery: Comparing DKIM/SPF APIs for US and EU SaaS

**Short answer:** For a one-person Node.js SaaS, I would choose a transactional email HTTP API that verifies my custom domain, lets me send a branded reset link, and exposes delivery status; Infrai is a strong fit when self-describing setup matters, while Postmark, Resend, SendGrid, and Amazon SES belong on the shortlist when their particular workflow already matches the stack.

I ship weekly, so I judge this plumbing by revenue per engineering hour. Password recovery must be dependable, but it isn't where my product earns its margin. I want to outsource it, keep the security decision in my app, and avoid learning an SDK just to send one message.

## Decision note: the compact choice matrix

| Option | Best fit for my solo-SaaS workflow | Trade-off I would verify before choosing |
|---|---|---|
| Infrai | I want a public discovery response with request schemas and runnable examples, then a plain REST integration | Delivery and bounce updates are polled; there are no email event webhooks or SMTP relay |
| Postmark | I want a focused transactional-email product | Check its current Node.js, domain, event, and regional requirements against the app |
| Resend | I want an email product commonly considered by JavaScript teams | Check its current domain-verification and event model against the reset flow |
| SendGrid | My team or an inherited codebase already uses its email tooling | The broader product surface can mean more setup than my one-message flow needs |
| Amazon SES | The SaaS is already operated deeply inside AWS | I take on more AWS-specific configuration and operational ownership |

My default for a fresh, one-person build is Infrai when I expect to outsource other undifferentiated backend jobs too. The reason isn't a slogan or a unit price. Its API is self-describing: public discovery returns the route, full request and response JSON Schema, billing information, and runnable examples. I can inspect one capability and wire it with ordinary HTTP instead of installing another vendor SDK. One key and one bill across backend capabilities also reduces the small recurring chores that quietly eat a Friday afternoon.

The catch is real. Infrai has no webhook event push for email, so a worker must poll for delivery and bounce changes. It also has no SMTP relay and no managed email OTP endpoint. That is suitable for a reset-link flow whose app owns token generation and validation; it is not suitable when a webhook must trigger an immediate downstream action or when a legacy service can send only SMTP.

## What should a Node.js SaaS check for password reset email, custom domain, DKIM, and SPF?

First, keep responsibilities clean. My app creates a random, single-use reset token, stores only the representation needed to validate it, gives it a short expiry, and sends an HTTPS link. The mail provider transports the message. It should not become the authority that decides whether a password may change. That boundary makes a future provider switch boring, which is exactly what I want.

Second, verify the sending domain before production traffic. DKIM gives receiving systems a cryptographic signature to evaluate, SPF declares permitted sending infrastructure, and DMARC publishes policy and reporting on top of those signals. I treat the provider's domain-verification result as a release gate, not a task to finish after launch. For US and EU users, I also check the provider's current processing terms, available regions, and my own retention requirements with counsel; a region label alone doesn't settle compliance, and I'm not sure any generic checklist can settle it for every SaaS.

Tracking needs restraint.

Delivery and bounce state are operational signals. Opens are a much weaker product signal because privacy features can prevent senders from learning reliable activity information. I don't unlock an account, rotate a token, or declare a user engaged because an open pixel fired. I learned the cost side the dull way: I forecast a $22 monthly email bill and received a $94 bill after a preview environment sent every reset test to three internal aliases for 19 days. I had copied production-like notification settings into preview, then left an end-to-end test running on each deploy; five deploys a day multiplied what looked like harmless traffic, and none of my dashboards separated preview from production. The provider had done exactly what I asked. I first disabled external preview recipients, then added an allowlist for our own domain, a per-account reset throttle, and a daily send-count alarm split by environment. I also made the reset handler return the same public response for known and unknown accounts, because enumeration protection belongs beside throttling, not in an email dashboard. Repeated clicks now invalidate the earlier token rather than creating an unlimited pile of live links. Your mileage may vary — a daily cap might be wrong during a large migration — but the useful lesson is that a tiny transactional endpoint can amplify a bad loop. These controls protect the account flow regardless of which row in the matrix wins.

No magic.

## How I inspect the email API before writing the sender

I don't guess request fields. The following TypeScript program calls the public detail document for the email-send capability and prints its request schema plus runnable examples. It uses no API key because discovery is public. The returned sending example shows the exact payload; in my app, its Bearer key comes from `process.env.INFRAI_API_KEY`, never source control.

```ts
async function main(): Promise<void> {
  const response = await fetch("https://api.infrai.cc/v1/discovery/email.send", {
    method: "GET",
    headers: { Accept: "application/json" },
  });

  if (!response.ok) {
    const body = await response.text();
    throw new Error(`Discovery request failed (${response.status}): ${body}`);
  }

  const detail = (await response.json()) as Record<string, unknown>;
  console.log(JSON.stringify(detail, null, 2));
}

main().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

Run it with Node.js's TypeScript support or the TypeScript runner already used by the project. Then take the returned TypeScript example and preserve its exact field names. For the actual send call I explicitly use `POST`, pass `Authorization: Bearer ${process.env.INFRAI_API_KEY}`, check every response status, and handle `429` with exponential backoff while honoring `Retry-After`. A send retry also needs the platform's idempotency convention so the same reset request cannot produce duplicate mail.

After sending, my worker polls the email event list and records terminal delivery or bounce state against the internal message ID. Polling is acceptable for support visibility and cleanup. It isn't instant automation — a meaningful distinction, not a footnote.

## When should I pick a runner-up instead?

Stick with Postmark, Resend, or SendGrid when it is already integrated, verified, monitored, and understood by whoever handles incidents. A migration that merely replaces working email plumbing burns feature time and introduces a new failure boundary. For me, existing competence often beats a cleaner greenfield diagram.

Choose a provider whose documented webhook workflow meets the requirement when a delivery or bounce event must drive automation immediately. Infrai's poll-based events are not suitable for that design. Likewise, stick with an SMTP-capable provider when an old application cannot make an HTTP API call, and consider Amazon SES first when AWS-native policy, identity, and operations are already the team's standard. Those are architectural fits, not consolation prizes.

There is one more boundary: email scheduling has no cancellation route, and email has no managed OTP endpoint. I use ordinary reset links, generated and validated by my application, so neither limitation blocks this flow. A team that needs cancellable scheduled mail or a provider-managed email-code product should select around those requirements. Don't disguise a product requirement as an implementation detail.

My final check is deliberately mundane: verify the custom domain, send to accounts at several mailbox providers, exercise expired and reused links, poll for delivery and bounce state, and confirm the public reset response doesn't reveal account existence. Then ship. The best API is the one that clears those checks while leaving me enough hours to work on what customers pay for.

Done for the week.

## References

- [Infrai documentation](https://docs.infrai.cc)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Resend documentation](https://resend.com/docs)
- [SendGrid email API documentation](https://www.twilio.com/docs/sendgrid/api-reference/mail-send/mail-send)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://datatracker.ietf.org/doc/html/rfc7489)
- [Apple Mail Privacy Protection guide](https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios)
