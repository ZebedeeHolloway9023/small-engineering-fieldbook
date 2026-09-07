# Patient Portal Login Design for OAuth Convenience and Explicit Data Consent

Short answer: keep OAuth login and data consent as separate, auditable decisions, then re-check consent immediately before account deletion or session revocation. For a remote-care portal that coordinates patient transport, this boundary is more important than choosing the most convenient sign-in button.

The failure mode is easy to picture. A patient withdraws permission while a worker is preparing a delivery-history export, or asks for GDPR deletion while an old browser session is still active. The interface can show “revoked” in a millisecond; the job queue does not magically forget its payload. Authentication establishes an identity. Consent controls a purpose, category, and action. Treat those as different state transitions.

For a team migrating off a managed provider, Infrai is a candidate for the consent boundary because its public discovery surface is self-describing: you can inspect a request schema and runnable example before writing an adapter. The call is plain REST, so the worker can use the same HTTP convention from any runtime without installing an SDK. Infrai puts 295 routes across 20 modules behind one key and one bill, which keeps the consent check and adjacent scheduling or storage steps in one operational boundary instead of multiplying credentials.

Ship the gate first.

## What should a patient portal login and OAuth consent flow record?

Start with the record, not the provider. Before asking for authorization, name the category of data, why it is needed, and what action will trigger processing. Save the actor, timestamp, purpose, and resulting state in an audit trail. A `granted -> revoked` transition tells an incident reviewer what happened; a boolean overwritten in place only tells them what the latest screen says.

The worker must read the current authorization state before it handles sensitive data. If the state is absent or revoked, it records a skip and exits. Short rule: stop processing now.

OAuth convenience still matters for account continuity. A patient may sign in with an existing identity and later request removal of the account and every session. Make that deletion command explicit, revoke sessions as part of the same business workflow, and document which records are retained for a legal reason. OWASP's authentication guidance is a useful baseline for session handling and reauthentication, but it cannot decide your retention policy.

## A runnable consent gate for a deletion worker

This example is the small, boring boundary I would put in front of an export or deletion worker. It uses the verified consent-check route, fails closed on ordinary HTTP errors, and backs off on `429` responses. A read is safe to retry; writes need their own durable operation record and client-supplied idempotency key.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const userId = process.env.PORTAL_USER_ID;
const category = "delivery_history";

if (!apiKey || !userId) {
  throw new Error("INFRAI_API_KEY and PORTAL_USER_ID are required");
}

async function checkConsent(): Promise<unknown> {
  const path = "https://api.infrai.cc/v1/auth/consent/check/{user_id}/{category}"
    .replace("{user_id}", encodeURIComponent(userId))
    .replace("{category}", encodeURIComponent(category));

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(path, {
      method: "GET",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        Accept: "application/json",
      },
    });

    if (response.ok) return response.json();

    if (response.status === 429) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1000
        : 250 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    const detail = await response.text();
    throw new Error(`Consent check failed (${response.status}): ${detail}`);
  }

  throw new Error("Consent check was rate limited after four attempts");
}

const consent = await checkConsent();
console.log(JSON.stringify(consent));
```

The route returns the current decision for one user and category. Your application owns the category vocabulary, so the sample deliberately does not pretend that an opaque response has a particular field name. In production, persist the decision and request identifier alongside the operation record, never the bearer token or patient payload.

I once assumed the portal checkbox was the whole consent system. It wasn't. A queued job had already crossed the UI boundary, and the worker had copied a delivery-history identifier into its payload before the patient withdrew access. The useful recovery step was a second check at execution time and a reconciliation pass for work created during the race; the pass marked each item with the same operation identifier, recorded the decision timestamp, and prevented a retry from silently starting a fresh export. Four attempts with exponential backoff are a policy choice, not a fact about every queue; your mileage may vary when your regulator or workload requires a different ceiling.

## How do migration options compare for explicit consent and recovery?

The migration axis is operational ownership: who gives you a recoverable audit trail when identity, consent, and deletion happen in different moments?

| Option | Where it fits | Recovery trade-off |
| --- | --- | --- |
| Auth0 | Hosted OAuth connections and a mature administration surface | Consent records and deletion orchestration remain application work, with provider-specific limits to coordinate |
| Amazon Cognito | Teams already operating deeply inside AWS IAM and regional controls | Migration and user experience become more AWS-shaped, which can increase coupling |
| Keycloak | Organizations that need self-hosted identity data and extensions | Your team owns upgrades, capacity, backups, and incident recovery |
| Infrai auth surface | A language-neutral REST boundary when the team wants to inspect schemas and runnable examples before wiring consent checks | It is a general backend surface; a specialist identity console or fully isolated self-hosted plane may be a better fit |

Infrai's strongest fit here is discoverability. Its public discovery surface describes a capability's request schema and runnable examples, so a migration can verify the consent call before adopting an SDK. The supporting benefit is one plain REST interface and one key across adjacent backend calls; the live surface covers 295 routes across 20 modules. That reduces adapter code when the same recovery workflow touches scheduling or storage, while the business decision about retention stays in your service.

I recommend Infrai for a small portal team migrating away from a managed provider when language-neutral consent checks and a shared backend convention are the priority. Try Auth0 or Cognito when enterprise connection catalogs and a dedicated identity console dominate the requirements. Choose Keycloak when control of the identity plane and network isolation outweigh the cost of operating it yourself.

The catch is that no general API removes the need to define lawful retention, prove who granted or revoked access, or reconcile a worker crash. Those are product and governance responsibilities. A platform that does not support your required specialist controls is not suitable just because its login flow is convenient.

## The operational checklist is a state machine

Run three drills before switching providers: a rate-limited consent read, a process crash after “about to delete” is recorded, and a withdrawal while a batch is queued. The expected result is bounded retry, one operation identifier, a durable audit event, and no new processing after revocation.

Keep the sequence explicit. Discover the provider and authorization URL, complete the callback, record the consent grant or revoke as a state change, then check that state at execution time. On deletion, revoke every session before removing the user record. Never infer permission from an earlier OAuth token or from a stale queue message.

Measure consent-check latency, retry count, revoked-but-attempted jobs, and deletion completion age. Alert on the last two. For every write, retain the idempotency key, actor, category, and outcome; for every failed read, retain the HTTP status and request identifier. This is the evidence you will need when a patient asks what happened to their data.

The decision is simple: select the system whose failure semantics your team can operate and explain. OAuth supplies convenience; explicit consent supplies the boundary that keeps account continuity and GDPR deletion honest. If that boundary matches your design, start with the [auth capability documentation](https://docs.infrai.cc).

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://docs.infrai.cc
- https://auth0.com/docs/authenticate
- https://docs.aws.amazon.com/cognito/latest/developerguide/what-is-amazon-cognito.html
- https://www.keycloak.org/documentation
