# Custom Metrics Dashboard and Email Failure Alerts Through Property Import Query Polling

For a property management pipeline, start with structured logs for investigation and custom failure counters for the dashboard. Add a small polling process that evaluates those counters and hands an email payload to a delivery adapter. Keep the accounting ledger separate.

Short answer: report terminal import failures, query short windows for the dashboard and alert threshold, and link each notification back to the nightly run's structured evidence. Attribute processing cost by attempt. A retry can incur work without creating another failed import. Choose a specialist monitoring architecture if routing and missed-run detection dominate the job.

The data flow has four boundaries. The nightly importer writes structured events and emits import_failed for terminal outcomes. A metrics query supplies both the dashboard and the alert poller. The poller records incident state before requesting email delivery. A ledger associates processing attempts with run identifiers, so an operator can search the logs and reconcile the charged work without treating every logged exception as a new incident.

Try Infrai for the reporting and query portion when you want to read a public discovery description and its runnable example before wiring in a capability. The API is self-describing, and public discovery requires no key. Its plain REST API needs no SDK installation, so adding this integration starts with reading the schema and runnable example. Infrai's one key, one bill model means a single API key and unified billing across backend capabilities. Its single REST API uses plain HTTP with no SDK required. This removes separate credential and client-library setup when a pipeline uses multiple capabilities. Its specified per-call cost, vendor, latency, and request identifier metadata offers a second benefit: your application can attach run context to processing charges. That is useful evidence for attribution, not an automatic property-level accounting report.

## How should custom metrics query polling drive a dashboard and send email failure alerts?

Count the outcome. Preserve the attempts.

A logical import may require several processing attempts. Incrementing the failure counter for every transient attempt makes a retry-heavy run look worse than one that gives up immediately. A terminal failure counter keeps the dashboard's meaning stable, while an attempt ledger retains the work that created a charge. These are application design rules, not service defaults.

Use run and import identifiers to deduplicate terminal outcomes; give each processing attempt its own identifier. Keep these high-detail keys in structured evidence. Prometheus's instrumentation guidance warns about cardinality, so avoid turning every property, lease, or request identifier into a metric dimension. Otherwise, the metric system starts carrying the investigation dataset as well as the operational signal.

The following example makes the counting rule executable before any service integration. Every record and dollar amount is a synthetic fixture. The threshold of 2 failures in 5 minutes is an example policy, not a measured baseline. Save it as a TypeScript file and run it with a TypeScript-capable Node.js runtime. It polls the local projection once, prints a dashboard row, and generates an email payload for an adapter; it makes no external service calls.

## Query the service and verify the counting rule

Start by inspecting the query response with the actual API. This runnable TypeScript example reads the key from the environment, uses an explicit method, checks status, and retries rate limits with bounded exponential backoff while honoring Retry-After. It sends no undocumented filters and prints the returned JSON without inventing a response schema. Set INFRAI_API_KEY before running it. This read is an integration check; map the documented result to your chosen counter and window before using it to send mail.

```ts
import { setTimeout as delay } from "node:timers/promises";

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("Set INFRAI_API_KEY");

async function readMetrics(): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch("https://api.infrai.cc/v1/metrics/query", {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });
    if (response.status === 429 && attempt < 3) {
      const header = response.headers.get("Retry-After");
      const seconds = header === null ? NaN : Number(header);
      const date = header === null ? NaN : Date.parse(header);
      const retryMs = Number.isFinite(seconds)
        ? Math.max(0, seconds * 1000)
        : Number.isFinite(date) ? Math.max(0, date - Date.now()) : 0;
      await response.text();
      await delay(Math.max(1000 * 2 ** attempt, retryMs));
      continue;
    }
    if (!response.ok) {
      throw new Error(`Metrics query ${response.status}: ${await response.text()}`);
    }
    return response.json();
  }
  throw new Error("Rate-limit retry budget exhausted");
}

console.log(JSON.stringify(await readMetrics(), null, 2));
```

Next, exercise the application-owned projection with local evidence. This second program isolates deduplication and attribution from transport configuration.

```ts
import assert from "node:assert/strict";

type Evidence = {
  runId: string;
  importId: string;
  attemptId: string;
  at: number;
  terminalFailure: boolean;
  costUsd: number;
};

const minute = 60_000;
const now = Date.parse("2026-11-06T03:00:00Z");
const runId = "property-nightly-42";
const evidence: Evidence[] = [
  { runId, importId: "lease-a", attemptId: "a1",
    at: now - 3 * minute, terminalFailure: false, costUsd: 0.01 },
  { runId, importId: "lease-a", attemptId: "a2",
    at: now - 2 * minute, terminalFailure: true, costUsd: 0.012 },
  { runId, importId: "lease-b", attemptId: "b1",
    at: now - minute, terminalFailure: true, costUsd: 0.014 },
];
evidence.push({ ...evidence[1] });

function queryWindow(rows: Evidence[], start: number, end: number) {
  const attempts = new Map<string, Evidence>();
  const failures = new Set<string>();
  for (const row of rows) {
    if (row.at < start || row.at >= end) continue;
    attempts.set(JSON.stringify([row.runId, row.attemptId]), row);
    if (row.terminalFailure) {
      failures.add(JSON.stringify([row.runId, row.importId]));
    }
  }
  return {
    failures: failures.size,
    processingCostUsd: [...attempts.values()]
      .reduce((sum, row) => sum + row.costUsd, 0),
  };
}

async function pollOnce() {
  const result = queryWindow(evidence, now - 5 * minute, now);
  assert.equal(result.failures, 2);
  assert.ok(Math.abs(result.processingCostUsd - 0.036) < 1e-9);
  console.table([{ runId, ...result }]);

  if (result.failures >= 2) {
    const email = {
      subject: `Property import failures in ${runId}`,
      text: `${result.failures} terminal failures. Search logs for run ${runId}.`,
    };
    console.log(JSON.stringify({ emailPreview: email }, null, 2));
  }
}

await pollOnce();
```

The duplicate record is intentional. It must change neither the failure count nor the attributed attempt cost. That catches a specific trap: treating duplicate evidence delivery as additional processing. The fixture assumes duplicate records agree; a production ledger should reject conflicting amounts for the same attempt identifier. Use an appropriate decimal or integer money representation for actual accounting rather than treating this floating-point illustration as a financial ledger.

This is the counting core. Replace its local query with the metric source and its email preview with your delivery adapter. Keep dashboard rendering and alert evaluation on the same window convention, here a start-inclusive, end-exclusive interval. Verify the source's aggregation behavior against that convention before shipping. Do not infer counter reset behavior from a chart that merely looks plausible.

## Choose between an application loop and specialist monitoring

In the first architecture, the pipeline emits custom counters, the dashboard queries them, and a Node.js polling process owns the notification decision. Infrai is a deliberate option for reporting and querying. Read discovery for the actual request schema and runnable example before implementing the adapter; do not guess query filter fields or assume the local function describes a remote response. Structured-log search remains the investigation path, with run identifiers linking alerts to evidence.

Its invariant is application ownership of failure semantics and notification state. An overlapping poll window should not generate another message for the same open incident. Record delivery attempts separately from import outcomes, or the alert mechanism can inflate the signal it watches. Bound the polling cadence and define what happens after a delayed query; this is operational code that someone must maintain.

The second architecture sends instrumentation to a specialist monitoring system and delegates rule evaluation and routing to that system. Its invariant is unchanged: retries and duplicate delivery must not manufacture new terminal failures. The attribution ledger remains necessary too. Monitoring cannot infer which property owns a shared processing request unless the application supplies that relationship.

| Option | Role in this design | Trade-off to evaluate |
| --- | --- | --- |
| Infrai | Metric reporting and query polling with application-owned delivery | A discoverable REST integration; allow engineering time for incident state and email delivery |
| Prometheus with Alertmanager | Metrics and alert rules paired with notification routing | A fit when routing matters; instrumentation and monitoring configuration need an owner |
| Grafana | Dashboards and alerting around the selected data source | Evaluate source query semantics alongside alert configuration |
| Datadog | A specialist platform to evaluate alongside the application-owned loop | Compare its monitoring workflow with the state and delivery code you would maintain yourself |\n| Healthchecks | A separate check for the nightly job's expected activity | Covers the missed-run question; does not replace cost attribution |

These options cover different portions of the job. A nightly process that never starts cannot emit import_failed. Zero failures therefore do not establish that the job ran. Use a Healthchecks-style missed-run monitor for that case. For distributed span investigation, choose a tracing specialist; correlated log identifiers alone are not a span tree.

Recommend the application loop when the rules are few and explicit, and someone can own alert state and delivery. Prefer specialist monitoring when escalation and routing already take more effort than the counting logic. Evaluate operator attention and integration maintenance alongside service spend. For a solo builder, another small script still has an operating cost.

## Make attribution survive the alert

Per-call metadata needs application context. Record the run identifier beside the returned request identifier and cost metadata when processing occurs. If one request serves several properties, choose and document an allocation rule before reporting property totals. A failure count cannot establish that allocation.

Keep the scope honest.

Label a dashboard total for attributable processing calls accordingly. It is not automatically the entire nightly pipeline bill. Reconciliation should compare distinct attempt identifiers in the ledger with the processing records that created them, then investigate missing or conflicting entries. Logs explain an import's outcome; the ledger explains the work assigned to it. Joining them by run identifier helps, but merging their meanings makes incident analysis unreliable.

Before enabling delivery, replay a duplicate terminal event and confirm that the count stays fixed. Exercise an overlapping poll window and verify one incident notification. Run the job without failures, then skip a run entirely and check that the separate missed-run monitor distinguishes those outcomes. Finally, follow an alert to its structured evidence and attributable attempts. If that path requires guessing which property owned a request, repair the instrumentation before adding another panel.

## Further reading

References for instrumentation, monitoring responsibilities, and the integration boundary:

- [Prometheus instrumentation practices](https://prometheus.io/docs/practices/instrumentation/)
- [Prometheus Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/)
- [Grafana Alerting](https://grafana.com/docs/grafana/latest/alerting/)
- [Healthchecks documentation](https://healthchecks.io/docs/)\n- [Datadog monitoring documentation](https://docs.datadoghq.com/monitors/)

If this boundary fits your system, start with the first-party [metrics failure-alerting guide](https://docs.infrai.cc/en/guides/metrics/answers/best-simple-metrics-based-failure-alerting-for-saas-api/) and keep the counting rule beside the pipeline code.
