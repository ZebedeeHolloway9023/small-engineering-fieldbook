# Email Fallback Workflow: 3 Evidence Gates for Multi-Channel SMS Status Polling

Use a persisted reset deadline and three evidence gates in a status polling workflow to control email-to-SMS fallback for healthtech password resets. The deciding constraint is whether a reviewer can reconstruct why a second channel was used while the reset was still valid.

Short answer: in a Node.js workflow without webhooks, poll email status from a scheduled worker, preserve each observation, and authorize SMS only after a defined email failure, a fresh expiry check, and a durable send reservation. Treat an unknown outcome as unresolved. This note proposes an implementation and a test protocol; it reports no production measurements or compliance certification.

## How should status polling authorize multi-channel email fallback to SMS?

The simplest candidate is a loop: send email, wait, ask for status, then send SMS if the answer is not delivered. Reject that rule for this experiment. A pending observation doesn't establish failure under the example contract below, and a worker that waits long enough may send a reset message after its useful lifetime. The chosen design separates the reset's validity from the transport's state. A status response supplies evidence about delivery; the application owns the deadline and the authorization to use another channel.

Keep those clocks separate.

For this proposed fixture, a reset expires 300 seconds after creation. Reserve the final 60 seconds for the recipient to act, so the worker stops authorizing new sends before that interval begins. These are example policy values, not standards or measured delivery guarantees. At the cutoff, an unresolved email produces an unresolved audit outcome, not automatic SMS. That choice sacrifices some opportunities to reach a recipient in exchange for a narrower, explainable fallback rule. A different policy can allow timeout-based fallback, but its evidence should explicitly say that a timeout authorized the send; it should never record a timeout as a confirmed email failure.

The three gates are concrete: an adapter reports a terminal failure under its documented mapping; the reset remains active with enough time left; and a transaction reserves exactly one fallback attempt. Record the policy version alongside the observation. For a healthtech application, the proposed audit payload uses an opaque reset reference and a channel identifier, with no patient details, destination address, or reset credential. Decide retention and access controls with the people responsible for the application's obligations. A delivery history alone cannot establish legal compliance.

## A focused worker example

The following TypeScript is an application-owned contract, not a provider SDK. The adapter must define what its normalized states mean. In particular, map a remote status to `failed` only when its documented semantics justify treating it as terminal; keep unrecognized or intermediate responses `pending`. The worker performs one observation per invocation. A durable scheduler supplies later invocations, which avoids keeping a server process asleep while the deadline advances.

The fixture is reset `reset-demo-17`, policy `reset-v1`, with a 300-second expiry and a 60-second action reserve. These identifiers describe synthetic data. The store and gateway interfaces expose the transactional and reconciliation responsibilities instead of hiding them in a timer.

```ts
type EmailState = "pending" | "delivered" | "failed";
type Outcome = "reserved" | "accepted" | "unknown";

interface ResetJob {
  id: string;
  emailRef: string;
  expiresAtMs: number;
  active: boolean;
  fallbackOutcome?: Outcome;
  policy: "reset-v1";
}

interface Store {
  load(id: string): Promise<ResetJob | null>;
  observe(
    id: string, state: EmailState, observedAtMs: number
  ): Promise<void>;
  // Atomically recheck activity, expiry, and absence of a reservation.
  // Persist the reservation and its stable key before returning true.
  reserveSms(
    id: string, key: string, nowMs: number, reserveMs: number
  ): Promise<boolean>;
  recordSms(
    id: string, outcome: "accepted" | "unknown", atMs: number
  ): Promise<void>;
}

interface Delivery {
  emailStatus(ref: string): Promise<EmailState>;
  // Resolve the destination and reset credential outside the audit log.
  // Reconcile ambiguous submissions by key before any repeat attempt.
  submitSms(resetId: string, key: string): Promise<"accepted">;
}

async function pollReset(
  id: string, store: Store, delivery: Delivery,
  now: () => number = Date.now
): Promise<void> {
  const actionReserveMs = 60_000;
  const job = await store.load(id);
  if (!job || !job.active || job.fallbackOutcome) return;
  if (now() >= job.expiresAtMs - actionReserveMs) return;

  const state = await delivery.emailStatus(job.emailRef);
  await store.observe(job.id, state, now());
  if (state !== "failed") return;

  const key = `${job.id}:sms:${job.policy}`;
  const reserved = await store.reserveSms(
    job.id, key, now(), actionReserveMs
  );
  if (!reserved) return;

  try {
    await delivery.submitSms(job.id, key);
  } catch {
    await store.recordSms(job.id, "unknown", now());
    return;
  }
  await store.recordSms(job.id, "accepted", now());
}
```

The reservation method is the concurrency boundary. Its implementation must use a transaction or equivalent conditional write; a preceding read followed by an unconditional update doesn't meet the contract. The gateway must also check current reset validity before submission, because the reset can be consumed or expire after reservation. The reset verification handler must enforce expiry and consumption independently of notification delivery. This sample leaves credential generation, destination eligibility, authentication, and storage implementation outside its scope. It is a worker skeleton, not a complete password-reset endpoint.

One reservation doesn't prove one external send. If the process dies after submission but before recording acceptance, the durable record still says `reserved`. Route that record to reconciliation. The example deliberately doesn't issue another submission on the next poll, because a stable key is useful only if the receiving boundary actually deduplicates it or can reconcile it. A failed local audit write after acceptance requires the same care. Keep the submission and audit-write error paths separate so that a database error is not mislabeled as a delivery failure.

## Template and sender evidence belong in the same record

An audit trail needs to identify what the application intended to send. Preserve a template revision, policy revision, channel, observation time, and adapter mapping revision with each decision. Store a restricted reference to the reset operation rather than copying its secret into the event log. That is a proposed data-minimization rule for this design, not a claim that a particular record layout satisfies a regulation.

The Mustache manual provides a useful, specific boundary: ordinary variable tags HTML-escape their values, while triple mustaches and ampersand tags insert unescaped values. It also documents that missing variables normally render as empty strings. Validate the required reset-link field before rendering; otherwise the template can produce a message without the intended link while the transport still accepts the resulting body. HTML escaping also doesn't validate where a link points. Construct and validate the reset destination in application code before passing it to the template. Keep SMS text rendering separate from HTML rendering so escaping decisions follow the output context.

Sender configuration deserves its own evidence. Yahoo's sender requirements describe authentication and distinguish requirements that apply to all senders from additional bulk-sender requirements. Use those published requirements to inform a deployment check for the relevant mail stream, and preserve the configuration revision used in a test. Don't turn a mailbox-specific requirement into a universal transport guarantee. Authentication configuration and a recorded transport acceptance answer different questions; neither establishes that the recipient completed the reset.

## Test the gaps between observations

Before deployment, run the same synthetic reset through a short trace matrix. Include a pending email that later reports delivery, a confirmed failure before cutoff, a failure observed after cutoff, and a consumed reset. Then inject overlapping worker invocations, a status lookup exception, and a crash immediately after SMS submission. Expected outcomes should be explicit: only the eligible confirmed failure reserves SMS, concurrent invocations share one reservation, and ambiguous submission requires reconciliation. A lookup error must not silently become a terminal delivery failure.

Use a controllable clock.

The most revealing expiry test starts the status lookup before cutoff and completes it afterward. The transaction must refuse the reservation even though the worker's initial check passed. Also test a crash after reservation but before submission: the recovery path needs to distinguish an unsent reservation from an externally accepted message, using whatever evidence the gateway can actually supply. If that evidence is unavailable, retain the uncertainty and expose it operationally. Don't manufacture a successful outcome to make the trace look complete.

For a small application, begin with one scheduled worker and a store that already supports the required conditional update. Keep normalization at the adapter boundary so changing a transport doesn't rewrite the reset policy. Deploy adapter mappings with recorded revisions, and run the trace matrix when a mapping or policy changes. Avoid putting an LLM in the fallback decision path: this proposed rule needs explicit states and timestamps, so model calls would add token usage and another dependency without supplying the missing delivery evidence.

## Measure before copying the policy

Measure queue delay, status-observation age, time remaining at fallback authorization, unresolved reservation count, and reset completion before expiry. Track status reads per reset and the number of second-channel submissions too. Those counts let a solo builder assess operational cost without relying on a volatile unit price. Faster polling creates more reads; choose its cadence from observed status timing and the action window rather than assuming every second improves recipient outcomes.

Keep the decision criterion strict: adopt this policy only if the trace can explain each fallback and the measured remaining window supports a useful reset attempt. If too many requests stay unresolved until cutoff, revisit the deadline, channel policy, or available evidence. A clean audit trail should show that trade-off plainly, including the requests for which no second message was authorized.

## References

- Mustache template syntax manual: https://mustache.github.io/mustache.5.html
- Yahoo sender best practices and requirements: https://senders.yahooinc.com/best-practices/
