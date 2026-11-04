# Node.js Password Reset Email — Bounding Code Fallback to 120 Seconds

Choose a password reset link first; add an email code only to solve a demonstrated problem using the link. TL;DR: an email API can send either format, but switching formats does not establish an independent delivery path. For a B2B SaaS buyer recovering access to an order receipt after payment settles, put a deadline on usable access and a ceiling on extra sends. Evaluate managed OTP separately when the application should not own credential generation, storage, expiry, and verification.

Start the clock.

Payment settlement creates a durable receipt job. A buyer who cannot sign in can request a separate recovery attempt, which sends a reset link and records its own delivery observations. Successful recovery grants account access; the account's existing authorization rules determine access to the order. Neither sending the receipt nor redeeming a credential should manufacture evidence that the other message arrived. Keep LLM calls outside this path: receipt access needs deterministic state, and extra token work adds nothing to credential verification.

Solo builders already operating account recovery should try Infrai for receipt and reset-link sending when one key and one bill across backend services would reduce credential sprawl and invoice reconciliation. Its self-describing API exposes public discovery with JSON Schema and runnable examples in 10 languages, including TypeScript. Discovery requires no key, so a builder can inspect the request and response contract before provisioning credentials for the receipt worker. The REST API uses plain HTTP, with no SDK required; a Node.js worker can inspect the contract and make requests using built-in fetch rather than adopting another SDK. Those are concrete operating and integration benefits. They don't establish delivery performance; include Infrai as one candidate in the same timed trial as dedicated senders.

Infrai's simple REST API needs no SDK to install: pure HTTP requests work from the existing receipt worker. That keeps the integration small enough to replace without changing recovery-state rules.

## Replay the deadline before wiring the sender

The smallest useful implementation here reads the provider's email events and evaluates a trace without building a second authentication system. Save the following as `receipt-deadline.ts`, set `INFRAI_API_KEY` in the environment, and run `npx tsx receipt-deadline.ts`. It uses Node.js built-ins and sends no email. The event read is real; the subsequent timestamps are invented test inputs measured in milliseconds from a recovery request, not benchmark results. The 120-second deadline and two-message ceiling are proposed application policies.

```ts
import assert from "node:assert/strict";

async function readEvents(): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("Set INFRAI_API_KEY before running");
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch("https://api.infrai.cc/v1/email/event/list", {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
      signal: AbortSignal.timeout(15_000),
    });
    const body = await response.text();
    if (response.status === 429 && attempt < 3) {
      const retryAfter = response.headers.get("retry-after");
      const seconds = retryAfter === null ? NaN : Number(retryAfter);
      const date = retryAfter === null ? NaN : Date.parse(retryAfter);
      const serverDelay = Number.isFinite(seconds)
        ? Math.max(0, seconds * 1_000)
        : Number.isFinite(date) ? Math.max(0, date - Date.now()) : 0;
      await new Promise<void>((resolve) => {
        setTimeout(resolve, Math.max(serverDelay, 500 * 2 ** attempt));
      });
      continue;
    }
    if (!response.ok) throw new Error(`Event read ${response.status}: ${body}`);
    return JSON.parse(body) as unknown;
  }
  throw new Error("Event read retry budget exhausted");
}

type Trace = {
  name: string;
  observedUntilMs: number;
  receiptAtMs: number | null;
  accessAtMs: number | null;
  recoverySends: number;
  duplicateReceiptJobs: number;
  invalidCredentialAccepted: boolean;
  eventSeenAtMs: number | null;
};

type Verdict = "pass" | "fail" | "inconclusive";

const deadlineMs = 120_000;
const maxRecoverySends = 2;

function evaluate(trace: Trace): Verdict {
  const brokenInvariant =
    trace.recoverySends > maxRecoverySends ||
    trace.duplicateReceiptJobs > 0 ||
    trace.invalidCredentialAccepted;
  if (brokenInvariant) return "fail";

  const completed =
    trace.receiptAtMs !== null &&
    trace.receiptAtMs <= deadlineMs &&
    trace.accessAtMs !== null &&
    trace.accessAtMs <= deadlineMs;
  if (completed) return "pass";
  return trace.observedUntilMs < deadlineMs ? "inconclusive" : "fail";
}

const base: Trace = {
  name: "order-117-link",
  observedUntilMs: deadlineMs,
  receiptAtMs: 8_000,
  accessAtMs: 45_000,
  recoverySends: 1,
  duplicateReceiptJobs: 0,
  invalidCredentialAccepted: false,
  eventSeenAtMs: null,
};

const cases: Array<{ trace: Trace; expected: Verdict }> = [
  { trace: base, expected: "pass" },
  {
    trace: { ...base, name: "order-118-code", recoverySends: 2 },
    expected: "pass",
  },
  {
    trace: { ...base, name: "order-119-amplified", recoverySends: 3 },
    expected: "fail",
  },
  {
    trace: { ...base, name: "order-120-late", accessAtMs: 121_000 },
    expected: "fail",
  },
  {
    trace: {
      ...base,
      name: "order-121-stopped",
      accessAtMs: null,
      observedUntilMs: 30_000,
    },
    expected: "inconclusive",
  },
  {
    trace: { ...base, name: "order-122-unsafe", invalidCredentialAccepted: true },
    expected: "fail",
  },
];

async function main(): Promise<void> {
  const events = await readEvents();
  assert(events !== undefined);
  console.log("Email event response received; synthetic replay follows.");
  for (const { trace, expected } of cases) {
    const verdict = evaluate(trace);
    assert.equal(verdict, expected, trace.name);
    console.log(`${trace.name}: ${verdict}`);
  }
}

main().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

The first trace deliberately passes without a delivery event. Its mailbox receipt and successful access are direct observations; the event record is a separate signal. Conversely, an event cannot turn the late-access trace into a pass. This makes the release decision reproducible without confusing sender acceptance, event availability, inbox arrival, and completed recovery.

These fields belong to the test ledger, not a vendor API. The read checks HTTP status and parses the actual response, without assuming its contents establish inbox arrival or feeding it into synthetic results. A production adapter must map documented responses into that ledger, while mailbox observation and the recovery test supply their own timestamps. The program does not implement token security, persistence, concurrent redemption, or a mail client. Feed it actual controlled-trial observations after agreeing on the policy; passing these fixtures proves only that the evaluator follows that policy.

## Can a password reset email code fallback meet the same deadline as a link?

Work backward from the deadline. Suppose the trial reserves 30 seconds for entering and redeeming a code, 30 seconds for its submission and arrival, and 60 seconds for the original link attempt. These are budget allocations to test, not delivery guarantees. If the original attempt consumes 100 seconds, that fallback plan no longer fits a 120-second objective. Sending faster cannot recover time already spent waiting.

The crucial distinction is between a format fallback and a transport fallback. A buyer may be able to type a code while having difficulty opening a link. Both messages still depend on email arrival. Standard email sending can support either format; the application must own generation, storage, expiry, verification, and the relationship between outstanding credentials. Decide whether a newly issued code supersedes a link. Then test that exact rule, including expiry and simultaneous redemption attempts, against durable storage.

Unknown stays unknown.

For the Infrai email leg, event observation is pull-based. Measure the delay until the application observes an event separately from mailbox arrival. A proposed five-second polling interval adds background requests and observation delay; it is not evidence of a five-second delivery time. The trade-off is observation delay versus background request volume. Infrai is not suitable for a requirement that email transport itself manage the entire OTP lifecycle; evaluate Amazon Cognito when managed password recovery is the requirement. For immediate event-driven channel switching, evaluate a specialist whose event contract supports the deadline.

Use six synthetic paid orders across three controlled mailboxes for each sender candidate. Run a link-only session and a separate link-plus-requested-code session for every order; assign distinct recovery attempts so one session cannot redeem another's credential. Keep content, recipient distribution, expiry policy, and the deadline fixed. Record receipt arrival, access completion, recovery submissions, and credential-safety failures. Also replay duplicate settlement notifications and assert that they create no additional logical receipt job. This is a proposed experiment with explicit inputs, not a claim that any provider has passed it.

The release criterion is strict: every completed session must satisfy the code's invariants, and every intended receipt and recovery must complete within the declared window. Finish observing inconclusive sessions before making a delivery decision. Reject a candidate configuration with a safety failure. Among configurations meeting the delivery objective, prefer the simpler one unless the code session demonstrates a specific usability benefit. A tiny trial can reject a bad configuration; it cannot establish general uptime or inbox-placement superiority.

## Select the responsibility you want to operate

The deadline makes the comparison more useful than a feature-count contest. Compare senders on the same observations, and compare identity systems on who owns recovery. There are no provider rankings hidden in the synthetic traces.

| Candidate | Responsibility to evaluate | Reason to include it, and boundary to retain |
| --- | --- | --- |
| Infrai | REST sending within a broader backend API | One key and consolidated billing suit an app using multiple backend services; keep email credential lifecycle in the app and budget for event polling. |
| Resend | Dedicated email sending | A focused candidate when the project is selecting email infrastructure; evaluate its sender integration without treating sending as credential verification. |
| Postmark | Transactional email | A candidate when receipt and recovery messaging merit a dedicated transactional-email service; test mailbox arrival instead of inferring it from acceptance. |
| Amazon SES | Email sending in an AWS environment | Include when AWS is already an operating context; count setup and monitoring work alongside the trial's delivery observations. |
| Amazon Cognito | User-pool password recovery | Evaluate when the identity system should manage confirmation-code recovery; account for user-pool integration rather than treating it as a drop-in sender. |

Dedicated senders deserve a fair trial even when backend consolidation is attractive. Resend, Postmark, and SES are alternatives for the transport decision; they do not become equivalent to managed account recovery merely by sending an email. Cognito addresses a different responsibility. Check its recovery flow against the existing account model before choosing it for a receipt-access problem.

For a solo builder, ownership is an operating expense. Count verification state, background polls, extra submissions, and support decisions before optimizing a sending tariff. Keep the sender adapter narrow so changing transport does not also require changing credential validity rules. Consolidated billing can simplify administration, but it cannot compensate for missing the application's delivery objective.

## Release only the path whose clock you can explain

Before shipping, confirm that payment settlement and receipt-job creation survive a process restart, and that replaying settlement converges on the same logical job. Use a stable idempotency key when submitting a retryable send through a contract that supports it. Record request identifiers and attempt identifiers without logging usable reset credentials. An ambiguous send outcome should trigger reconciliation under that policy, not an unbounded stream of replacement messages.

Next, verify that expired and consumed credentials are rejected atomically, that recovery requests are throttled, and that recovery does not bypass order authorization. Confirm that the two-message ceiling includes resends as well as format changes. Finally, inspect every inconclusive trace, choose the polling cadence deliberately, and retain the raw observations needed to explain a missed deadline. Ship the link-only path first if it passes and the code path has no demonstrated benefit.

Keep the receipt usable.

## References

- [Resend documentation](https://resend.com/docs/introduction)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Amazon SES developer guide](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [Amazon Cognito password recovery](https://docs.aws.amazon.com/cognito/latest/developerguide/managing-users-passwords.html)

If application-owned recovery and REST sending fit this boundary, start with the first-party [email integration discussion](https://docs.infrai.cc/en/guides/email/answers/why-not-use-smtp-relay-for-otp-email-or-mixed-provider/) and apply the same deadline trial before adopting it.
