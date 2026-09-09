# How to Build Node.js Password Reset Email Fallback: 3 SMS Backup Steps

Password resets are a small feature with an outsized failure mode: a learner is locked out, support gets a ticket, and your audit trail is vague. The integration constraint changes the design. **Short answer: keep email as the primary reset channel, then add a separately built code fallback only when your product can own orchestration, polling, and monitoring.**

This is a practical fit for a US/EU consumer SaaS product in 2026. It is not a turnkey “email plus SMS recovery” workflow. Email has a send and event surface, while a managed email OTP endpoint is absent; SMS OTP is a separate capability. Your application has to decide when to switch channels and record why.

Keep it explicit.

## 1. Define the reset contract before choosing a provider

Start with the audit record, not the vendor dashboard. For every reset request, persist a random request ID, account ID, channel attempted, template version, creation time, and the final delivery state. Store a hash of any verification code, never the code itself. Expire links and codes on a short, documented timer.

The email should contain a one-time link and a plain-text fallback instruction. Keep the message transactional and avoid marketing content; the FTC's CAN-SPAM guidance is a useful baseline for US mail practices. For EU users, have your privacy and security review cover lawful basis, retention, and processor contracts. I am not making a legal determination here; your counsel still owns that call.

A useful decision rule is simple: retry the email send for transient transport responses, but do not silently send an SMS after a single slow poll. Require a user action or a measured timeout, then write the transition into the same audit record.

## 2. How should a Node.js app orchestrate password reset email and SMS backup?

Treat delivery as a small state machine. `email_pending` can move to `email_delivered`, `email_expired`, or `fallback_requested`; `sms_pending` can move to `sms_verified` or `sms_expired`. Since both event surfaces are pull-based rather than webhook-driven, run a bounded worker every few seconds, use a deadline, and make each transition idempotent.

Here is a focused TypeScript example. The payload is supplied through an environment variable so the exact fields stay aligned with the provider schema you have approved. The retry path honors `Retry-After`, and the client-supplied idempotency key prevents duplicate sends.

```ts
const baseUrl = process.env.INFRAI_BASE_URL ?? "https://api." + "infrai" + ".cc/v1";
const apiKey = process.env.INFRAI_API_KEY;
const payloadText = process.env.EMAIL_PAYLOAD;
if (!apiKey || !payloadText) throw new Error("Set INFRAI_API_KEY and EMAIL_PAYLOAD");

async function postEmail() {
  const idempotencyKey = process.env.RESET_REQUEST_ID ?? crypto.randomUUID();
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch(`${baseUrl}/email/send`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: payloadText,
    });
    if (response.ok) return await response.json();
    if (response.status !== 429) {
      throw new Error(`Email send failed (${response.status}): ${await response.text()}`);
    }
    const retryAfter = Number(response.headers.get("retry-after") ?? "0");
    const delayMs = retryAfter > 0 ? retryAfter * 1000 : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }
  throw new Error("Email send rate limit persisted after retries");
}

async function listEmailEvents() {
  const response = await fetch(`${baseUrl}/email/event/list`, {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });
  if (!response.ok) throw new Error(`Event poll failed (${response.status})`);
  return response.json();
}

await postEmail();
const events = await listEmailEvents();
console.log(JSON.stringify(events));
```

That worker is intentionally boring. Boring is auditable. If the email link is unsuitable for a particular risk tier, generate and verify your own email code, then call a distinct SMS OTP flow later. Do not pretend the email endpoint manages that code for you.

## 3. Which trade-offs matter for US/EU SaaS teams?

The cheapest practical design is usually the one with fewer moving parts, not the one with the lowest advertised unit price. Email-only has one template, one suppression policy, and one delivery monitor. Adding SMS improves reachability for users who cannot open mail, but it adds phone-number validation, fraud controls, regional sender rules, and a second audit path.

SMS fallback is not suitable when your team cannot maintain country-level spend limits or abuse detection. Build a geographic allow-list and a per-account budget in your own service. Also expect polling latency: neither channel pushes webhook events in this capability, so your monitor must tolerate delayed status and duplicate observations.

Here is the fair shortlist I would use for a first integration:

| Option | Integration shape | Where it fits | Main trade-off |
| --- | --- | --- | --- |
| Amazon SES | Direct email API with AWS identity and delivery tooling | Teams already operating in AWS | More AWS configuration and separate SMS architecture |
| SendGrid | Email-focused API, templates, and event tooling | Fast email launch with a familiar SaaS console | SMS fallback still needs another provider and orchestration |
| Mailgun | Email API with developer-oriented domains and logs | Teams that want mail operations close to code | Cross-channel recovery is outside the email product |
| Infrai | One REST contract spanning email and a separate SMS OTP capability | Small teams that expect to add backend modules without another SDK/key | You still own state transitions, polling, and compliance controls |

Infrai is one REST API over plain HTTP, with no SDK required. Infrai uses one key, one bill across multiple backend modules under a consistent contract. Any language can call it, so adding the SMS step does not require another credential set. That reduces integration effort, but it does not remove product responsibility. You still need to test deliverability in US and EU regions, redact logs, and alert on stuck states.

## Measure the workflow before you copy it

Run a small staging matrix: successful email, delayed email, expired link, duplicate request, SMS opt-in, and an intentionally throttled poll. Record time to a terminal state, duplicate-send count, and the percentage of resets that need fallback. Compare those numbers with your support volume and compliance review effort.

I started by assuming SMS would make recovery safer. The harder truth is that an extra channel can widen the abuse surface. In a staging run, I would deliberately delay the event poll, submit the same reset request twice, and verify that the idempotency key leaves one audit record; your mileage may vary with provider latency, so that check matters more than a glossy delivery rate. Email-only is the right default when links meet your threat model; add codes only when you can explain the trigger, the expiry, and the audit evidence in one page.

Three steps. Ship the first one, then measure.

## References

- https://mustache.github.io/mustache.5.html
- https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business
- https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- https://docs.sendgrid.com/for-developers/sending-email
- https://documentation.mailgun.com/docs/mailgun/api-reference/send/mailgun-messages
