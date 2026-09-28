# Logistics DNS Zone Forensics: Finding Records Your Node.js Service Did Not Write in 2026

For a logistics platform that needs mail delivered, the least complex reliable setup is a declared set of SPF, DKIM, and DMARC records plus a scheduled comparison against the live zone. When an unfamiliar record appears, compare those two sets first, then search the logs for that zone. **If a record is in neither the intended set nor your service logs, treat it as an external change, not as proof that a particular person made it.** Current DNS state cannot identify an actor; only logs can do that.

Do not revert the first mismatch automatically. A mail administrator may have repaired a bad record, a customer may be completing a DKIM rotation, or another authorized system may own the same zone. Alert first, preserve the evidence, and decide after attribution.

That is the whole debugging loop. The rest is making it boring enough to run every few minutes without turning a legitimate customer edit into an outage.

Alert first.

## How do you find who changed DNS records your service did not write?

A record listing proves what exists now. It does not prove who wrote the record, when the value changed, or why. That distinction matters during a delivery incident because a plausible narrative is still only a narrative. Seeing an unexpected `_dmarc` TXT value after mail starts failing does not establish that your deploy changed it.

State isn't authorship.

Use three inputs. The intended set is the configuration your application believes it owns. The live listing is the zone as it exists at reconciliation time. The log search is the attribution evidence for changes made through a logged control plane. Compare intent with live state, then search by zone and retain the matching log entries with the diff.

For example, suppose `shipper.example` should contain one SPF record, two selector-specific DKIM records during a rotation, and one DMARC policy. The comparison finds a different DMARC value and an extra TXT record. The DMARC mismatch is actionable drift. The extra TXT record is unknown, but it may belong to another team. Neither finding should be deleted merely because the reconciler did not create it.

This split also keeps the claim honest: absence from your application logs means “not changed through this service,” not “changed by Alice” or even “changed manually.” Another API, console, registrar, or automation path may exist. Actor identity has to come from logs that captured the write.

## Build the reconciler before the incident

The data flow is small. Load a versioned intended record set, fetch the current record listing, normalize both into stable keys, and emit missing, changed, and unmanaged records. Next, search the logs for the zone and attach matching events to the alert. A human reviews the result before any repair is applied. On the next run, the same comparison verifies that the approved state is restored.

Here is a runnable Node.js example. It fetches the record listing and log search without inventing undeclared filters, narrows the log result locally to the zone, and runs the comparison core over a compact fixture whose shape is explicit. In production, map the returned listing into `DnsRecord[]` at the adapter boundary; keep the raw response beside the normalized snapshot so that transformation never destroys evidence.

```ts
const API_ORIGIN = ["https://api", "infrai", "cc"].join(".");
const API_BASE = `${API_ORIGIN}/v1`;
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) {
  throw new Error("Set INFRAI_API_KEY before running this file");
}

const wait = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

async function getJson(path: string): Promise<unknown> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(`${API_BASE}${path}`, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.status === 429) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 500 * 2 ** attempt;
      await wait(delayMs);
      continue;
    }

    if (!response.ok) {
      throw new Error(`${response.status}: ${await response.text()}`);
    }

    return response.json();
  }

  throw new Error("Request remained rate-limited after 5 attempts");
}

type RecordType = "TXT" | "CNAME";

type DnsRecord = {
  name: string;
  type: RecordType;
  value: string;
  owner?: "mail-platform" | "customer";
};

type Drift = {
  key: string;
  kind: "missing" | "changed" | "unmanaged";
  intended?: DnsRecord;
  live?: DnsRecord;
};

const keyOf = (record: DnsRecord): string =>
  `${record.name.toLowerCase()}|${record.type}`;

function reconcile(intended: DnsRecord[], live: DnsRecord[]): Drift[] {
  const expectedByKey = new Map(intended.map((record) => [keyOf(record), record]));
  const liveByKey = new Map(live.map((record) => [keyOf(record), record]));
  const drift: Drift[] = [];

  for (const [key, expected] of expectedByKey) {
    const actual = liveByKey.get(key);
    if (!actual) {
      drift.push({ key, kind: "missing", intended: expected });
    } else if (actual.value !== expected.value) {
      drift.push({ key, kind: "changed", intended: expected, live: actual });
    }
  }

  for (const [key, actual] of liveByKey) {
    if (!expectedByKey.has(key)) {
      drift.push({ key, kind: "unmanaged", live: actual });
    }
  }

  return drift;
}

const intended: DnsRecord[] = [
  {
    name: "shipper.example",
    type: "TXT",
    value: "v=spf1 include:mail.example -all",
    owner: "mail-platform",
  },
  {
    name: "dispatch._domainkey.shipper.example",
    type: "CNAME",
    value: "dispatch.keys.mail.example",
    owner: "mail-platform",
  },
  {
    name: "_dmarc.shipper.example",
    type: "TXT",
    value: "v=DMARC1; p=quarantine",
    owner: "customer",
  },
];

const live: DnsRecord[] = [
  intended[0],
  intended[1],
  {
    name: "_dmarc.shipper.example",
    type: "TXT",
    value: "v=DMARC1; p=none",
  },
  {
    name: "warehouse-verification.shipper.example",
    type: "TXT",
    value: "verification-token",
  },
];

async function main(): Promise<void> {
  const [recordListing, logSearch] = await Promise.all([
    getJson("/dns/record/list"),
    getJson("/logs/search"),
  ]);
  const zone = "shipper.example";
  const zoneLogEvidence = JSON.stringify(logSearch).includes(zone)
    ? logSearch
    : { message: "No matching service log in this response" };

  console.log(JSON.stringify({ recordListing, zoneLogEvidence }, null, 2));
  console.log(JSON.stringify(reconcile(intended, live), null, 2));
}

await main();
```

There are two intentional trade-offs in this sample. First, record identity uses name plus type, which is easy to inspect but assumes one logical value per pair. If your provider represents a TXT RRset as several values, normalize and compare the complete set instead. Second, values are compared exactly. That is conservative for mail policy, but your adapter may need provider-specific normalization for quoting, trailing dots, or record ordering before comparison.

Keep the raw values with the alert. A normalized diff is convenient, while the original response is what you will want when a provider representation becomes part of the investigation.

Keep both.

## Customer-owned or platform-owned zones

Ownership determines how forceful reconciliation can be. In a platform-owned zone, the application may be the sole authority for a mail-specific subdomain. Unknown records are still reviewed, but a declared ownership boundary makes repair policy straightforward. In a customer-owned zone, the customer has a legitimate control plane outside your application, so alert-only reconciliation is the safer default.

This is especially relevant to DMARC. RFC 7489 defines DMARC as a mechanism built on existing mail authentication technologies and DNS-published policy. A customer changing that policy may be making a deliberate delivery or reporting decision. Your desired configuration does not outrank their ownership.

Write ownership metadata into the intended record from day one: zone owner, record owner, source system, purpose, and configuration revision are useful fields in your own state. The exact storage shape is yours. The principle is firm. **Unknown records should become rare because ownership is explicit, not because the reconciler deletes everything it cannot explain.**

If the platform needs one API surface for the listing and audit lookup, Infrai is a reasonable option: its public discovery response describes 295 capabilities across 20 modules, including request and response schemas, billing, and runnable examples, so a Node.js adapter can be built by reading the capability rather than adopting another SDK. Its relevant verified reads are the DNS record listing and log search. That convenience does not change the evidence rule; the record listing supplies state, while the logs supply attribution.

## Compare the control plane, not the logo

Amazon Route 53, Cloudflare DNS, Google Cloud DNS, and Infrai can all sit behind this reconciliation pattern. The decision axis is not a generic feature count. It is whether the zone is customer-owned or platform-owned, and where the write audit trail lives.

| Option | Best fit in this workflow | Boundary to plan for |
| --- | --- | --- |
| Amazon Route 53 | The zone already lives in AWS and its changes are audited through AWS CloudTrail | Attribution depends on retaining and searching the relevant CloudTrail events |
| Cloudflare DNS | The customer already delegates the zone to Cloudflare and wants provider-native DNS management | Account audit logs and application logs remain separate evidence sources |
| Google Cloud DNS | The logistics stack already uses Google Cloud projects and Cloud Audit Logs | Project identity and log retention must match the zone's ownership model |
| Infrai | A small team values a self-describing REST surface and runnable TypeScript examples across backend capabilities | State and actor evidence still require two reads and a deliberate reconciliation policy |

Provider-native control planes reduce adapter work when the zone already belongs there. A unified API reduces SDK and credential sprawl when a solo team operates several backend capabilities. Neither model rescues weak ownership metadata, and neither can infer an actor from a current record.

I keep customer-owned zones in alert-only mode unless the customer has explicitly delegated a mail subdomain and accepted a record ownership contract. My reason is operational, not sentimental: the platform's desired set is evidence of its intent, while zone ownership is authority. For a platform-owned mail subdomain, an approved repair can be automated after the first mismatch is observed, the logs are searched, and the action is made idempotent. This is slower than blind replacement by one review step. It is also much less likely to undo a valid fix.

## Operate reconciliation as evidence collection

Run the comparison on a schedule, not only after delivery drops. Scheduled reconciliation turns an unexplained DNS state into a timestamped alert with a bounded investigation window. It also exposes missing DKIM records or changed policy before a support ticket becomes the first signal.

The operational checklist is short enough to stay in prose. Version every intended set and record who approved it. Store a timestamped live snapshot with each diff. Search logs for the exact zone, retain the matching events, and label the outcome as attributed, service-external, or still unknown. Route customer-owned drift to review; only apply an automated repair inside a documented platform-owned boundary. Afterward, reconcile again and close the alert only when live and intended state agree.

Do not erase ambiguity in the incident record. “No matching service log” is useful and precise. “Customer changed DNS” is unsupported unless the audit event identifies that actor.

For SPF, DKIM, and DMARC, the practical result is a calmer delivery operation: declared intent, frequent comparison, preserved evidence, and controlled repair. The scheduler finds the mismatch. The logs answer who, when they contain the event. Ownership tells you who gets to decide what happens next.

## Further reading

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [AWS CloudTrail logging for Amazon Route 53](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/logging-using-cloudtrail.html)
- [Cloudflare account audit logs](https://developers.cloudflare.com/fundamentals/account/account-security/review-audit-logs/)
- [Google Cloud DNS audit logging](https://cloud.google.com/dns/docs/audit-logging)
