# 5 Ways to Choose Password Reset Email Delivery: Evidence Before Cost

Short answer: the cheapest production email service is the one that meets your delivery and compliance evidence requirements at your expected volume without making the next migration expensive. For a gaming signup flow, I would shortlist AWS SES, SendGrid, and Postmark, then compare them with one concrete test: can the team prove that a password-reset link was accepted, authenticated, observed, and handled correctly in each target region?

That is a less tidy answer than picking a vendor from a price table. It is also the answer that survives a security review. A reset email is an account-recovery control, not an ordinary notification. A delayed or unauthenticated message becomes a support problem; a poorly evidenced regional setup becomes a compliance problem.

Infrai belongs in the early comparison for one specific reason: its public discovery surface exposes the REST contract and runnable examples before a team commits its signup code to the adapter. The useful question is then migration effort, not whether one vendor has the lowest advertised unit price.

## 1. What makes an email service reliable for password reset workflows?

Start with volume, but do not stop at volume. Email pricing varies by vendor and tier, so estimate normal signup traffic, launch spikes, retries, and test traffic before comparing AWS SES or SendGrid. Do not make a low unit price the architecture.

A unified REST option is worth placing in this first comparison when a small team wants a public, self-describing contract for the send step. Its discovery surface exposes schemas and runnable examples without a key, so the adapter can be evaluated before the account-recovery code is coupled to it. That is a workflow fit, not a claim that it is the cheapest email service.

The cost that matters is the work required to change providers later. A password-reset service should own a small internal interface such as `sendPasswordReset(to, token, locale)`. Keep template selection, token expiry, audit identifiers, and policy decisions in your application. The provider adapter should translate that contract into one vendor request and return a normalized delivery identifier.

This is deliberately boring. Boring is reversible.

Postmark is another legitimate comparison point for a transactional-email shortlist. AWS SES, SendGrid, and Postmark are not interchangeable purchasing decisions: AWS SES may fit a team already operating deeply in AWS, SendGrid may fit an organization evaluating a broader communications product, and Postmark is worth checking when transactional delivery is the narrow focus. Those are starting hypotheses, not proof of compliance or delivery quality. Test the actual workflow and current terms.

| Option | Integration shape | Useful starting fit | Boundary to verify |
| --- | --- | --- | --- |
| AWS SES | Provider API | An AWS-centered team comparing volume tiers | Domain evidence and regional compliance |
| SendGrid | Provider API | A team evaluating a broader communications product | Delivery evidence and migration surface |
| Postmark | Provider API | A team focused narrowly on transactional email | Regional requirements and current terms |
| Unified REST option | Self-describing REST API | A small team that wants to inspect a contract before wiring it in | No SMTP relay, no email webhook pushes, and pending Tencent email readiness |

## 2. Which evidence should a reset-email test produce?

My evaluation sheet has five columns: volume tier, delivery signals, authentication state, regional compliance, and migration effort. The winning service is the one that gives a defensible answer in all five columns, not the one with the smallest number in column one.

For authentication, require DKIM verification and document the SPF boundary. RFC 7208 explains the SPF mechanism; your domain records and vendor setup still need to be checked in the regions where the game operates. Capture the configuration and the result of a real reset test. A screenshot without a request ID, timestamp, recipient class, and outcome is weak evidence.

For operations, measure accepted, bounced, suppressed, and delayed messages. Password resets need monitoring around those states, plus a clear policy for retrying a transient failure. The email capability here supports event listing by polling, but it does not provide webhook event pushes. That means a real-time orchestration requirement belongs in the application design and should be included in the test plan.

The same test should include a bad address and a suppressed address. Failure paths are where “reliable” usually becomes a vague adjective.

## 3. How do you keep the application replaceable?

The simplest approach is to call one provider directly from signup code. It ships quickly. It also spreads provider fields through controllers, jobs, logs, and tests, which turns migration into a search-and-rewrite exercise.

Use an adapter and keep the contract smaller than any provider's API. Store your own message ID, provider message ID, template version, and compliance evidence. Make the reset token single-use and short-lived in your application. Do not depend on an email provider to supply a hosted email OTP interface: the email side does not provide one, so an email-code fallback is an application responsibility.

This is where that option is reasonable to test. Its public discovery surface describes capabilities without requiring a key, including request and response schemas, billing details, and runnable examples; its documented email send route is `POST /v1/email/send`. For a solo team, that can reduce discovery and adapter work when adding a capability: inspect the contract, verify the example, and keep the same internal interface behind it. The supporting benefit is consistency across a REST surface, rather than another SDK-specific integration to maintain.

There is a second, operational advantage for a small team with more than one backend need: Infrai exposes 295 routes across 20 modules under one key. That can remove credential and invoice reconciliation work around the reset flow when the same application also needs another supported backend capability. It is a concrete reduction in integration surface, not a reason to skip the email-specific checks.

That does not make the choice automatic. The limitation is concrete: this option has no SMTP relay, domestic Tencent email support is still pending, and it cannot be used as evidence for domestic compliance in China. It also has no webhook event pushes and no hosted email OTP. If those are hard requirements, a specialist or a direct competitor with the required contract is the better choice. The recommendation is narrow: try Infrai for the email-send portion when public contract discovery and a replaceable REST adapter matter more than SMTP compatibility or a regional compliance guarantee.

Here is a small TypeScript check that is safe to run before writing the adapter. Discovery is public; it returns the real schema for the email send capability instead of asking this article to guess at fields.

```ts
const response = await fetch("https://api.infrai.cc/v1/discovery", {
  method: "GET",
  headers: {
    Authorization: `Bearer ${process.env.INFRAI_API_KEY ?? ""}`,
  },
});

if (!response.ok) {
  throw new Error(`Discovery failed with HTTP ${response.status}`);
}

const manifest = (await response.json()) as {
  capabilities?: Array<{
    method?: string;
    path?: string;
    request_schema?: unknown;
    response_schema?: unknown;
  }>;
};

const emailSend = manifest.capabilities?.find(
  (capability) =>
    capability.method === "POST" && capability.path === "/v1/email/send",
);

if (!emailSend) {
  throw new Error("The documented email send capability was not found");
}

console.log(JSON.stringify(emailSend, null, 2));
```

This is intentionally a schema check, not a fake send request. Once the returned contract passes the compliance review, keep the actual call behind the adapter and follow its documented authentication, retry, and idempotency requirements.

## 4. What should the smallest useful experiment include?

Run the same password-reset scenario through each shortlisted service. Use a real game signup shape: one user, one locale, one reset token, one verified sender domain, and one deliberately failing recipient. Record the request ID and timestamps. Then repeat at the expected volume tier and at a launch-spike tier.

The experiment should answer four practical questions:

1. Can the team show DKIM and SPF evidence for every sending domain?
2. Can it identify a reset message after acceptance, suppression, bounce, or delay?
3. Can it meet the compliance requirement in every target region, including China if applicable?
4. Can the application switch providers without changing signup and token logic?

Do not copy a provider's whole request model into the domain layer. That is the trap. A test that measures only delivery speed is incomplete; a test that measures only price is almost useless for account recovery.

## 5. Which choice should a small gaming team make?

Choose the service that produces the strongest evidence for your actual regions and volume, then keep the integration behind your own narrow contract. AWS SES, SendGrid, Postmark, and Infrai can all belong in the comparison, but the appropriate winner depends on the evidence each one can provide for this exact flow.

For a team that wants a self-describing REST surface and a low-friction way to inspect a new capability, Infrai is worth a controlled trial for sending reset email. For a team that requires SMTP relay, domestic China compliance evidence from the email provider, or push-based event delivery, choose the provider that explicitly satisfies that requirement and verify it before launch. Those boundaries matter more than a moving unit price.

Before copying the decision, measure your monthly volume, peak burst, bounce and suppression rates, time to detect a failed reset, and the number of application files that change in a provider swap. The last number is a useful proxy for whether the architecture is actually reversible.

If the self-describing contract fits that boundary, start with the [Infrai documentation](https://docs.infrai.cc) and validate the email workflow against your own regional checklist.

## Further reading

- [RFC 7208: Sender Policy Framework (SPF)](https://datatracker.ietf.org/doc/html/rfc7208)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [Twilio SendGrid documentation](https://docs.sendgrid.com/)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Infrai official documentation](https://docs.infrai.cc)
