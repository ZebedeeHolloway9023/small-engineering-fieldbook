# Hard API Spend Caps vs Application Rate Limiting: SaaS Key-Rotation Audit

When a production API key must rotate without downtime, the hard part is proving what happened afterward. A spend ceiling can stop an account from crossing a budget, while application rate limiting can slow or reject requests. Neither one, by itself, proves which tenant used the key, which requests were admitted during the handoff, or why a bill changed.

Short answer: use both controls, but treat them as separate evidence streams. Put a hard spend cap at the account or provider boundary, enforce per-tenant request limits in the application, and write an append-only rotation record that joins key versions, request decisions, and usage estimates.

That distinction matters for a developer-tools SaaS where one leaked credential can fan out across build jobs. I care about the audit trail because a green dashboard is not evidence. The useful question is whether an engineer can reconstruct the key transition from logs without asking the billing system to guess.

## What can a hard spend cap stop, and what can it never explain?

A hard cap is a budget boundary enforced outside the request handler. Depending on the upstream service, crossing it may reject new work, suspend a project, or require an operator to raise the limit. It is a valuable blast-radius brake: a runaway retry loop cannot spend past the configured ceiling once the boundary is active.

The cap does not understand your tenant, feature, or release. If ten customers share one account, the cap sees aggregate usage. It also cannot tell you whether a rejected call was a legitimate deployment, a replayed webhook, or a compromised key. Billing windows add another wrinkle: usage can be reported later than the request, so an apparent zero balance is not a proof that no work is in flight.

The audit implication is easy to miss. A cap event is an account-level fact, not a request-level decision. Store the cap identifier, effective time, and observed provider response alongside your own request ledger. Do not turn a provider's eventual invoice into the sole source of truth for access review.

## How should API spend caps and application rate limiting work together for SaaS?

Rate limiting lives closer to the caller. A token bucket keyed by tenant and route can enforce a burst size and a refill rate before the request reaches an expensive API. It can preserve fairness, protect latency, and attach a reason code to every refusal. Those are things a hard spend cap cannot do.

Rate limiting has its own blind spots. Tokens measure requests or estimated units, not necessarily the provider's billable amount. A single request can contain a large payload, and a low-volume batch can still consume a large budget. Distributed workers also need a shared counter; a process-local limiter quietly multiplies the allowance when a new replica starts.

I once started with a single `X-RateLimit-Remaining` counter in an API gateway. It looked tidy until a blue-green deploy split traffic across two stores. During the 14-minute overlap, both fleets admitted their full allowance. The incident was not a mysterious provider charge; it was a missing ownership field in the decision log. We traced the requests by deployment id, compared the two counter streams, and found that each fleet believed it was the only writer. The fix was larger than changing a number: the decision key now includes tenant, credential version, route class, and a monotonic window id, while the record also names the limiter store and deployment that made the decision. That extra context let us explain the overage to a reviewer without guessing from aggregate billing data.

Evidence beats a green dashboard.

The practical division is:

| Control | It is good at | It cannot establish |
| --- | --- | --- |
| Hard spend cap | Bounding aggregate account exposure | Which tenant or request caused the boundary |
| Application rate limit | Fairness, bursts, and immediate refusal reasons | Exact provider billing or delayed usage |
| Usage ledger | Attribution and reconciliation | Preventing a request unless paired with enforcement |
| Key-rotation log | Who changed access and when | Detecting spend by itself |

The table is not a hierarchy. These controls should disagree sometimes, and the disagreement should be observable rather than silently resolved.

## A rotation protocol that leaves reviewable evidence

Key rotation is a short distributed transaction. Create the new credential, publish it to workers, observe successful use, then revoke the old one. Keeping the old key alive for a bounded overlap avoids a service-down cutover, but the overlap must be an explicit state with an expiry.

Here is the shape I use in a Node.js service. The storage calls are deliberately generic: the important contract is the event written before and after each state transition.

```ts
type KeyVersion = "old" | "new";

type RotationEvent = {
  id: string;
  tenantId: string;
  keyVersion: KeyVersion;
  action: "created" | "published" | "observed" | "revoked";
  actor: string;
  at: string;
  requestId?: string;
};

async function rotateProductionKey(tenantId: string, actor: string) {
  const rotationId = crypto.randomUUID();
  await audit.append({
    id: rotationId,
    tenantId,
    keyVersion: "new",
    action: "created",
    actor,
    at: new Date().toISOString(),
  });

  const secret = await secrets.create({ tenantId, purpose: "production-api" });
  await workers.publishSecret(tenantId, secret, { rotationId });
  await audit.append({
    id: crypto.randomUUID(),
    tenantId,
    keyVersion: "new",
    action: "published",
    actor,
    at: new Date().toISOString(),
  });

  await readiness.waitForVersion(tenantId, secret.version);
  await audit.append({
    id: crypto.randomUUID(),
    tenantId,
    keyVersion: "new",
    action: "observed",
    actor: "rotation-controller",
    at: new Date().toISOString(),
  });

  await secrets.revokePrevious({ tenantId, purpose: "production-api" });
  await audit.append({
    id: crypto.randomUUID(),
    tenantId,
    keyVersion: "old",
    action: "revoked",
    actor,
    at: new Date().toISOString(),
  });
}
```

The subtle rule is ordering. A `revoked` event without a preceding `observed` event should page an operator, not be repaired by editing history. Make audit records append-only, give each one a request or rotation id, and keep timestamps in UTC. OWASP's secrets guidance also recommends limiting secret exposure, rotating credentials, and monitoring their use; those practices make the log useful during an access review rather than merely decorative.

## What each control cannot do during a failure?

Imagine a worker retries a failed build five times while the old key is still valid. The rate limiter may correctly allow the retry burst because the tenant is below its request quota. The spend cap may later stop the account, but neither signal says which retry consumed the disputed units. Your ledger needs an estimated cost per admission, the provider's usage event when available, and a link to the key version used.

The inverse failure is just as common: a strict request limiter rejects a release because a tenant has a temporary burst, even though the account has ample budget. That is an availability decision, not a cost decision. Return a stable reason such as `tenant_rate_limit`, include a retry time, and keep the refusal in the same audit stream as successful calls.

Do not hide uncertainty. Provider usage can arrive late, and your estimate can differ from the final charge. Mark the record as `estimated` or `reconciled`; never overwrite the original decision. I'm not sure any single dashboard can show this faithfully across providers, so I prefer a small event schema that can be queried and replayed.

## A decision rule for a solo team

Start with the failure you are trying to contain. If the unacceptable outcome is an unexpectedly large invoice, configure a hard cap and alert on the remaining headroom. If it is one tenant starving everyone else, enforce an application limiter backed by shared state. If the requirement is an access review, neither control is sufficient without key-versioned audit events.

Test the handoff under load: two key versions, two worker pools, a delayed usage report, and a simulated limiter-store failover. Measure admitted requests, estimated units, reconciliation lag, and the time needed to answer “who used this credential?” Keep the overlap window short enough to reduce exposure, but long enough for workers to confirm the new version.

The catch is operational weight. A shared limiter, append-only storage, reconciliation job, and rotation controller are not suitable for a tiny internal script. In that case, stick with a provider cap and a simple single-tenant key until the audit requirement justifies the machinery. For a multi-tenant developer SaaS, accepting that complexity is usually cheaper than reconstructing access from an invoice after the fact.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
- https://www.rfc-editor.org/rfc/rfc6585
- https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/429
- https://www.rfc-editor.org/rfc/rfc9110
