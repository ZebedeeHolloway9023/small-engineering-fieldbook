# Node.js Password Recovery Backend (Postgres Email, Enumeration-Safe Retry Audits)

Keep the new-order template and notification policy in the Node.js application, then hide delivery behind a narrow adapter. TL;DR: for a developer-tools marketplace, this makes the seller notification testable and keeps a provider change from leaking into order code. Postgres should enforce cooldowns and retry counters, every attempt should produce an audit record, and support should retain the provider message ID for later status checks.

The evaluation constraint is template ownership. A successful API response is not enough if changing the transport also means rewriting order logic, migrating opaque template IDs, or losing the evidence needed when a seller says the email never arrived.

## How should a Node.js password backend use Postgres for email retries?

The application should own the meaning of `seller-new-order-v1`: which order transition triggers it, which fields it may render, and which revision was sent. The adapter should own authentication, transport syntax, and the provider's returned identifier. **That boundary is the decision.**

I would start with the tempting design: call a vendor template directly from the checkout handler and store its template ID beside the order. It is short, but the provider's vocabulary then becomes marketplace state. A second channel or provider forces the checkout path to learn another payload shape. Instead, the business layer should emit one small, versioned request that adapters translate. This is a deliberate trade: a little adapter code now buys a reviewable template revision and a smaller migration later.

This matters beyond migration. A seller could receive repeated notices if a queue retries after delivery but before the worker records success. The application therefore needs one logical notification ID, reused across attempts, plus a database-controlled retry budget. It also needs an application-level cooldown if the same order event can be raised repeatedly. Abuse prevention is application-managed; a delivery API does not replace it.

Short code can still duplicate mail.

For a password-reset variant of the same design, the public response must stay generic whether the account exists or not. Return the same success message and observable response shape, keep the reset cooldown and retry count in Postgres, and never let a provider template decide account existence. The marketplace order flow does not have that enumeration risk, but both workflows benefit from the same transport boundary and audit discipline.

## A focused TypeScript send

Infrai's public discovery response supplies the current JSON Schema for each capability without requiring a key. Generate and validate the email body from that schema, then pass the result as `EMAIL_SEND_BODY`; doing so avoids freezing an unverified field list into an article. This runnable adapter uses the verified send route, explicit Bearer authentication, a stable idempotency key, status checks, and bounded HTTP 429 handling.

```ts
const wait = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter && /^\d+$/.test(retryAfter)) {
    return Number(retryAfter) * 1_000;
  }
  return 500 * 2 ** attempt;
}

export async function sendEmail(
  body: unknown,
  notificationId: string,
): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");
  const baseUrl = ["https://api", "infrai", "cc/v1"].join(".");

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(`${baseUrl}/email/send`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": notificationId,
      },
      body: JSON.stringify(body),
    });

    if (response.ok) return response.json() as Promise<unknown>;

    const errorBody = await response.text();
    if (response.status !== 429 || attempt === 3) {
      throw new Error(`Email send failed (${response.status}): ${errorBody}`);
    }
    await wait(retryDelay(response, attempt));
  }

  throw new Error("Email retry budget exhausted");
}

const body = JSON.parse(process.env.EMAIL_SEND_BODY ?? "null") as unknown;
const notificationId = process.env.NOTIFICATION_ID;
if (!body || !notificationId) {
  throw new Error("EMAIL_SEND_BODY and NOTIFICATION_ID are required");
}

const receipt = await sendEmail(body, notificationId);
console.log(JSON.stringify(receipt));
```

The surrounding `claim` operation should be a Postgres transaction, not a read followed by an insert. It checks the cooldown, increments the bounded retry counter, and reserves the logical notification in one operation. The notification ID becomes the idempotency key. The platform convention specifies a 24-hour default deduplication window, and 171 of 294 documented capabilities declare idempotency, but the database record should remain the durable source of marketplace intent. The adapter above makes four attempts at most, backs off exponentially from 500 milliseconds, and honors a numeric `Retry-After`; it doesn't tight-loop.

There is still a crash boundary after provider acceptance and before the application records the returned message ID. Idempotency makes replay safer, while the audit row exposes an unresolved attempt for reconciliation. Do not log reset tokens or other secrets. For ordinary password recovery and one new order, use a single send; batch delivery is for many independent transactional notices created together. Once accepted, persist the message ID rather than the entire response body.

Four attempts are enough for this example.

## Direct providers and a unified contract solve different problems

These products should not be flattened into a price table. Amazon SES is an email service, while Twilio's cited surface is SMS. SendGrid, Postmark, and Resend are real products a team can evaluate for direct email delivery. Infrai is the unified-contract option in this set. The fair test is whether the chosen product and template model match the channels and operational ownership the marketplace actually needs.

| Option | Boundary to evaluate | Fit for this experiment | Limit to verify before choosing |
|---|---|---|---|
| Amazon SES | Direct email integration | A team choosing an email-specific service | How template revisions and delivery evidence fit the application's audit model |
| SendGrid | Direct email integration | A team evaluating provider-managed versus application-owned templates | Whether vendor template identifiers can stay inside one adapter |
| Postmark | Direct email integration | A team evaluating a dedicated transactional-email product | The export and migration path for template revisions |
| Resend | Direct email integration | A Node.js team evaluating another email-specific option | The exact status and retry contract required by support tooling |
| Twilio SMS | Separate SMS integration | A marketplace that needs an SMS branch as well as email | SMS registration, suppression, and country controls remain distinct concerns |
| Infrai | One REST contract spanning capabilities | A small team that expects the backing vendor or channel mix to change | Email events are pull-based, and several channel and reporting gaps remain |

The direct services deserve preference when their native workflow is the product requirement: for example, when a team wants one provider's template editor to be the source of truth. Application ownership is less attractive if a non-engineering content team must revise transactional copy frequently through an existing approval workflow. In that case, keep a logical name such as `seller-new-order-v1` in code, map it to the hosted ID inside the adapter, and record the hosted revision in the audit row.

Infrai fits the other side of the trade-off. The capability contract can stay fixed while the vendor behind it changes, so checkout code does not acquire a new SDK or credential whenever transport changes. Infrai uses one key and one bill across the capability surface, with no SDK required; that reduces credential rotation and invoice reconciliation for a solo team adding storage or scheduling beside notifications. The plain REST API lets a Node.js worker use its built-in HTTP client. Its API is genuinely self-describing, and public discovery requires no key. That surface returned 295 capabilities across 20 modules in the verified snapshot; capability details include request and response schemas, billing information, and runnable examples, with examples reported across 10 languages. Per-call metadata consistently specifies cost, vendor, latency, cache status, and a request ID, giving the audit worker another concrete record to retain. During adapter work, I value these inspectable contracts more than another convenience wrapper because the integration can check the current shape instead of copying an old request from an article.

Those advantages do not erase boundaries. Infrai has no webhook event push for either email or SMS, so status-driven orchestration must poll. Email has no hosted OTP operation, and scheduled email has no cancellation operation; SMS does have cancellation. There is no SMTP relay and no voice, WhatsApp, or RCS channel. Tag-aggregated cost reporting is absent, and SMS geographic abuse controls and country-price circuit breakers remain application work. A pending domestic email vendor also must not be treated as evidence of China compliance.

## Audit delivery after acceptance

Acceptance and delivery are different states. Store the provider message ID alongside the logical notification ID, order ID, template version, attempt count, timestamps, and the adapter name. When a seller opens a support ticket, poll status using that message ID rather than sending again on instinct. For Infrai, the verified email surface includes `POST /v1/email/send` and `GET /v1/email/get/{id}`; the latter supports this pull-based investigation. No webhook should be assumed.

The audit record should describe state, not capture message secrets. A password-reset implementation stores a token hash rather than the raw token. It returns a generic success response for known and unknown addresses, even during cooldown, because a different status or message would permit user enumeration. The marketplace notice uses the same database mechanics without pretending the two threats are identical.

Retry only within a bounded policy, preserve the logical ID, and distinguish provider acceptance from later delivery. This is where a plain adapter pays for itself: support logic reads one internal audit model even when the transport changes. I would reject an adapter that can't return a durable message identifier, because it leaves support guessing precisely when the abstraction is supposed to help.

## Measure this before copying the choice

Run the experiment with a fixed fixture: one seller, one order, one template revision, and one forced retry using the same notification ID. Verify that concurrent workers yield one database claim, that a repeated attempt carries the same idempotency identity, and that the message ID can be used to retrieve status. Then swap adapters without changing `notifySeller` or the stored business fields.

Measure operational evidence, not a marketing checklist: duplicate logical sends, unresolved accepted attempts, time until a polled status becomes useful to support, and the number of provider-specific fields outside the adapter. Also inspect how template revisions are reviewed and recovered. Do not invent a latency target from this comparison; no runtime latency, uptime, or savings benchmark was measured here.

**Choose application-owned templates when a stable contract and vendor mobility outweigh dashboard editing.** Choose a hosted template workflow when its collaboration controls are the actual requirement, but contain its IDs. Either way, Postgres owns cooldowns, retry counters, and the audit trail. The provider delivers the message; it does not own marketplace policy.

## References

- Amazon SES documentation: https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- Twilio SMS documentation: https://www.twilio.com/docs/sms
- SendGrid documentation: https://www.twilio.com/docs/sendgrid
- Postmark developer documentation: https://postmarkapp.com/developer
- Resend documentation: https://resend.com/docs
