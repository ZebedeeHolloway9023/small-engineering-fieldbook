# Transactional Email Deliverability 2026: Auditable Domain Setup and Polling-Based Evidence

Use a verified custom domain, treat SPF, DKIM, and DMARC as deployable configuration, and keep a durable application-side record of every delivery event. For a US/EU compliance notice, the deciding constraint is evidence: the system must show what it attempted, what the provider later reported, and whether an address was suppressed before another send.

**TL;DR:** A direct email API with domain verification and event polling is enough for a basic compliance-notice workflow. It is a poor fit when the application requires SMTP relay or immediate webhook automation. Polling is not merely a slower webhook; it changes cursor storage, reconciliation, alerting, and the meaning of “delivered.”

The simple approach is to authenticate a domain, send a message, and save the API's initial success response. That proves too little. An accepted request is not final delivery, and a later bounce or complaint must affect the next send. The better design is a small state machine backed by an append-only evidence log.

## How should a NodeJS custom domain email deliverability setup work?

Start by separating facts that are often collapsed into one “sent” flag. A useful record can prove that the application selected an approved template revision, addressed a particular recipient, submitted the request at a known time, and received a provider outcome later. It can also prove that the application checked suppression state before a future attempt.

It cannot prove that a human read the notice. Open tracking is weak evidence for that claim, especially because privacy features can load remote content without the recipient deliberately opening the message. Apple documents that Mail Privacy Protection prevents senders from learning Mail activity accurately. For compliance work, retain provider delivery outcomes, bounce and complaint history, and the exact notice content; do not promote an open pixel into proof of readership.

I would model submission and delivery as separate transitions:

| State | Evidence retained | Operational consequence |
|---|---|---|
| `prepared` | template revision, recipient, domain, content hash | no provider claim yet |
| `accepted` | provider message ID, request time, idempotency key | await a later outcome |
| `delivered` | raw delivery event and ingestion time | stop delivery polling for this message |
| `bounced` or `complained` | raw event, reason if supplied, ingestion time | suppress before another send |
| `unknown` | last successful poll time and cursor | reconcile; never relabel as delivered |

Keep both.

The raw event belongs beside the normalized state because provider schemas change, and normalization code can discard the one field an auditor later asks about. Storage is cheap compared with reconstructing an incomplete trail. The practical test is uncomfortable but useful: if the normalized row were challenged six months later, could an engineer reproduce it from an immutable source record, identify the parser version, and explain every transition without consulting an old dashboard? If not, the record is an operational convenience rather than audit evidence.

## Domain authentication is a release, not a checkbox

SPF authorizes sending infrastructure. DKIM attaches a cryptographic signature associated with the sending domain. DMARC tells receivers how to evaluate alignment and where to send aggregate or forensic reports. RFC 7489 is the primary reference for DMARC semantics; dashboard labels are not a substitute.

Publish the records supplied by the selected provider, verify the domain through its control plane, and wait for the domain status to become verified before permitting production sends. Then monitor that status. DKIM keys also need rotation, so ownership of rotation and its rollback path belongs in the runbook rather than in somebody's calendar memory.

Roll out DMARC deliberately. A reporting policy can reveal legitimate senders that were missed during inventory, while an enforcement policy can reject mail that fails alignment. The correct policy depends on the domain's actual mail streams. No generic article can choose it safely for you.

Use a dedicated transactional subdomain when organizational policy permits it. That makes ownership and DNS changes easier to reason about, but it does not excuse reputation management. Domain verification confirms control and configuration; it does not guarantee inbox placement.

This is also where a broad API surface can reduce integration work. Infrai puts 295 capabilities across 20 modules behind one REST API and one key, so adding domain controls and suppression does not add another SDK or credential. Its public discovery is self-describing and returns request and response schemas without authentication. The trade-off is structural: email events are pull-based, and there is no SMTP relay, so the application owns the poller and cannot drop the service behind an existing SMTP transport.

## The polling loop is part of the compliance boundary

A webhook usually moves retry responsibility toward the receiver endpoint. Polling moves scheduling and cursor correctness into your application. Run one logical poller per account or partition, persist its cursor transactionally, and make event ingestion idempotent. A restart must replay safely.

Do not advance the cursor before the corresponding raw events are durable. If the provider response and your database write cannot share a transaction, save the page first, normalize second, and advance only after both steps succeed. Overlap the query window when the API allows it and deduplicate on a stable provider event identifier. Where the provider does not expose such an identifier, hash a canonical representation of the raw record and document the weaker guarantee.

No cursor, no progress.

The focused code below polls the documented event-list route and records the raw response rather than guessing undocumented event fields. Set `INFRAI_BASE_URL` to the service API base and `INFRAI_API_KEY` in the runtime secret store. A `429` honors `Retry-After` when present and otherwise uses exponential backoff; all other non-success responses surface their actual body.

```ts
import { createHash } from "node:crypto";
import { appendFile } from "node:fs/promises";

type EvidenceRow = {
  observedAt: string;
  digest: string;
  raw: unknown;
};

const evidenceFile = "./email-event-evidence.jsonl";

function canonicalJson(value: unknown): string {
  if (Array.isArray(value)) {
    return `[${value.map(canonicalJson).join(",")}]`;
  }
  if (value !== null && typeof value === "object") {
    const entries = Object.entries(value as Record<string, unknown>)
      .sort(([a], [b]) => a.localeCompare(b));
    return `{${entries.map(([key, item]) =>
      `${JSON.stringify(key)}:${canonicalJson(item)}`).join(",")}}`;
  }
  return JSON.stringify(value);
}

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter && /^\d+$/.test(retryAfter)) return Number(retryAfter) * 1_000;
  return Math.min(1_000 * 2 ** attempt, 30_000);
}

async function pollEvents(): Promise<void> {
  const baseUrl = process.env.INFRAI_BASE_URL;
  const apiKey = process.env.INFRAI_API_KEY;
  if (!baseUrl || !apiKey) throw new Error("Missing API configuration");

  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(new URL("/v1/email/event/list", baseUrl), {
      method: "GET",
      headers: {
        Authorization: `Bearer ${apiKey}`,
      },
    });

    if (response.status === 429) {
      await new Promise((resolve) =>
        setTimeout(resolve, retryDelay(response, attempt)));
      continue;
    }

    const body = await response.text();
    if (!response.ok) {
      throw new Error(`Event poll failed (${response.status}): ${body}`);
    }

    const raw: unknown = JSON.parse(body);
    const row: EvidenceRow = {
      observedAt: new Date().toISOString(),
      digest: createHash("sha256").update(canonicalJson(raw)).digest("hex"),
      raw,
    };
    await appendFile(evidenceFile, `${JSON.stringify(row)}\n`, "utf8");
    return;
  }
  throw new Error("Event poll remained rate-limited after five attempts");
}

await pollEvents();
```

This example is deliberately local and small. It writes one raw poll response per line; it does not pretend the digest alone is a complete deduplication scheme. A production adapter should parse only fields present in the documented response schema, enforce uniqueness with a provider event ID when one exists, encrypt sensitive records, restrict access, and apply a retention schedule approved for the notice type. If pagination or a cursor is present in that schema, commit it only after the corresponding raw page is durable. That ordering is the part worth copying.

Polling interval is a business decision. As a concrete design exercise, a system with a 15-minute evidence SLA could begin by polling every 60 seconds, then validate that choice against provider limits and measured lag before launch; those are proposed control values, not benchmark results. Measure the age of the oldest unprocessed event, not just whether the job ran. Alert on cursor stagnation, repeated poll failures, and a growing gap between submission and terminal outcome.

## Suppression must happen before submission

A suppression list is executable compliance state. Check it before each send, and add hard bounces and complaints after event ingestion. Do both in the same application workflow so a marketing retry, a scheduled job, or a second service cannot bypass the decision. My ship-first rule is to block the second sender until it uses the same suppression check; duplicating the policy in two codebases creates a quiet disagreement exactly where the evidence needs one answer.

No exceptions by default.

Be precise about scope. A global complaint suppression may be appropriate, while a temporary mailbox failure may call for a time-bounded rule. Legal notices can have delivery obligations that differ from promotional mail, so product policy and counsel need to define the permitted fallback. The mail API cannot make that judgment.

There is another boundary worth making explicit: this capability has no hosted email OTP operation, and scheduled email sends do not expose a cancellation operation. If a workflow needs an email verification code, the code lifecycle belongs to the application. If a notice must remain cancellable until a deadline, queue it in your own scheduler and submit only when the deadline passes.

Short section, important rule: never retry a write blindly. Give each notice a stable client-generated identity, use the provider's documented idempotency mechanism where available, and preserve that identity across timeouts.

## Choosing among direct APIs, SMTP, and push events

The shortlist should follow the application's existing control plane. Amazon SES, Twilio SendGrid, Postmark, and Mailgun are real alternatives, but their most useful distinction here is integration shape rather than a temporary price table.

| Option | Documented integration shape | Best fit for this notice system | Boundary to test |
|---|---|---|---|
| Amazon SES | API and SMTP sending; event publishing through AWS destinations | teams already operating IAM, AWS monitoring, and event infrastructure | configuration spans several AWS services |
| Twilio SendGrid | Web API or SMTP; Event Webhook | applications that need pushed delivery events or an SMTP migration path | verify webhook signatures and define replay handling |
| Postmark | API or SMTP; delivery and bounce webhooks | transactional-only programs favoring a focused mail product | confirm retention and event export meet audit policy |
| Mailgun | API or SMTP; webhooks and event retrieval | teams wanting both push events and queryable event history | define region and retention requirements before launch |
| Pull-based unified API | direct API, domain controls, suppression operations, event polling | small teams adding mail beside other backend capabilities under one contract | application owns polling; no SMTP drop-in path |

These are not quality rankings. SES can be a natural choice inside an AWS estate and needless operational surface for a tiny independent product. SendGrid, Postmark, or Mailgun can remove the polling delay through webhooks, but then signature validation, endpoint availability, duplicate delivery, and replay are your problems. The pull-based option is attractive when one key and a consistent contract across many backend modules reduce maintenance more than immediate events would.

For domestic China delivery, do not infer readiness from US/EU support. The available domestic email vendor is still pending in the capability described here, so it cannot serve as evidence for a China compliance decision. Voice, WhatsApp, and RCS are outside this surface too; a multichannel escalation plan needs separate providers or a different platform.

## What to measure before copying this design

Ship the smallest evidence-complete path first: one verified domain, one approved template, one notice class, one event poller, and one suppression policy. Then test it with controlled addresses that can produce delivery and bounce outcomes. Do not manufacture a complaint against an uninvolved mailbox.

Track domain verification state, submission acceptance, time to observed terminal outcome, poll lag, duplicate-event rate, unknown outcomes past the notice deadline, suppression-check failures, and reconciliation mismatches. Record these by provider and region when those dimensions are available. A tag-based cost aggregation endpoint is not available in the unified surface, so allocate cost in your own ledger if notices need per-case accounting.

The decision rule is plain. Choose polling when the permitted evidence delay exceeds measured poll lag and your application can own reconciliation. Choose a webhook-capable provider when response time is tighter than a responsible polling interval. Choose SMTP only when compatibility with an existing mail transport is a real requirement, not because SMTP is familiar.

One last limit matters more than any dashboard: “delivered” is the provider's transport outcome, not proof that the recipient understood or even saw the notice. Name the claim correctly in schemas, reports, and audit exports. That restraint keeps the engineering evidence useful.

## References

- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance (DMARC): https://datatracker.ietf.org/doc/html/rfc7489
- Apple, Use Mail Privacy Protection: https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios
- Amazon SES, email sending methods: https://docs.aws.amazon.com/ses/latest/dg/send-email.html
- Amazon SES, event publishing: https://docs.aws.amazon.com/ses/latest/dg/monitor-sending-activity-using-notifications.html
- Twilio SendGrid, Event Webhook: https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event
- Twilio SendGrid, SMTP integration: https://www.twilio.com/docs/sendgrid/for-developers/sending-email/integrating-with-the-smtp-api
- Postmark, Webhooks overview: https://postmarkapp.com/developer/webhooks/webhooks-overview
- Postmark, SMTP API: https://postmarkapp.com/developer/api/smtp-api
- Mailgun, Webhooks: https://documentation.mailgun.com/docs/mailgun/user-manual/events/webhooks
- Mailgun, Events API: https://documentation.mailgun.com/docs/mailgun/api-reference/send/mailgun/events
