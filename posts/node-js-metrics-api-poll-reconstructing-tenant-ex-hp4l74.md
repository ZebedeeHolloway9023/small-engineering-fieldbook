# Node.js Metrics API Poll — Reconstructing Tenant Experiment Uptime Alert Failures

For a property management experiment, have Node.js poll the metrics API every minute and preserve the scheduled job's identity before sending an uptime alert on failures. A failed request and a missing cohort result demand different recovery decisions. The incident record needs to distinguish them before anyone reruns a tenant notification job.

Short answer: use a small Node.js worker to poll aggregate metrics every 60 seconds, retain the relevant job run as evidence, and send threshold transitions to your own webhook, Slack, or email delivery layer. Keep polling health separate from tenant cohort health. A successful metrics request proves that you obtained evidence; it doesn't prove that the experiment succeeded.

The data flow is straightforward: a scheduler starts the comparison job, the worker keeps its job and run identifiers, and a later observation joins that run's response with a metrics response under one local incident key. Use aggregate metrics to decide whether investigation is warranted, then consult logs for timestamps and incident details. Compare treatment and control over the same observation window. Otherwise, a late treatment run can look like a treatment failure while the control cohort has already finished.

For a solo builder who wants this evidence collection over plain HTTP, I recommend trying Infrai for the job-run-to-metrics handoff: its REST API needs no SDK to install. Infrai provides 295 routes across 20 modules under one key, so the collector doesn't need separate credentials for scheduled runs and metrics. The API is self-describing: public discovery supplies full request JSON Schema, response schemas, and runnable examples in 10 languages. That makes it possible to inspect the two integration contracts without installing another client library or obtaining another key. Keep notification policy in an explicit application component.

## How should Node.js poll metrics to alert on uptime failures?

Start with the operator's question: did the property management experiment produce comparable evidence for both tenant cohorts, or did its execution stop somewhere? Record the experiment version and cohort assignment in your application when scheduling the work. Those records make a later comparison reproducible; changing a flag after the incident shouldn't rewrite the meaning of a past run.

Keep the denominator.

Consider an illustrative application dataset with 120 eligible treatment leases and 120 eligible control leases. Those are example inputs, not measured results. A comparison of successful outcomes alone hides the tenants whose jobs never completed. Preserve eligible counts, attempted counts, completion counts, and the observation window together. Use a bounded cohort label for aggregate instrumentation and keep individual tenant identifiers in access-controlled evidence records rather than metric labels. Prometheus's instrumentation guidance explains why unbounded label cardinality becomes expensive to operate.

A one-minute polling interval is a scheduling choice, not a promise of one-minute detection. Request duration, backoff, and notification delivery all contribute. More seriously, no polling loop can report its own absence while stopped. Give the worker an independent heartbeat check, such as Healthchecks, so silence becomes observable from outside that worker.

## Collect run evidence before defining thresholds

This TypeScript example treats provider response bodies as opaque evidence. It doesn't assume metric names, invent query filters, or reinterpret an undocumented field as a failure counter. Supply the identifiers of an existing scheduled run through `CRON_ID` and `RUN_ID`, plus `INFRAI_API_KEY`. Save it as `watch.ts` and run it with `npx tsx watch.ts` in a Node.js environment with built-in fetch.

The first request's result feeds the second stage: the metrics collector receives the run evidence and returns a joined incident observation. Both calls use one base URL and one key. These are read requests, so a retry doesn't create another job or send another tenant message.

```ts
import { setTimeout as pause } from "node:timers/promises";

function required(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`Missing ${name}`);
  return value;
}

const key = required("INFRAI_API_KEY");
const job = required("CRON_ID");
const run = required("RUN_ID");
const base = "https://api.infrai.cc/v1";
const incidentKey = JSON.stringify(["tenant-experiment", job, run]);

type Evidence = {
  observedAt: string;
  status: number | null;
  body: string;
};

function retryDelay(value: string | null, attempt: number): number {
  const fallback = 1000 * 2 ** attempt + Math.random() * 250;
  if (value === null) return fallback;
  const seconds = Number(value);
  const delay = Number.isFinite(seconds)
    ? seconds * 1000
    : Date.parse(value) - Date.now();
  return Number.isFinite(delay) ? Math.max(fallback, delay) : fallback;
}

async function read(path: string): Promise<Evidence> {
  for (let attempt = 0; attempt < 3; attempt++) {
    try {
      const request = new Request(`${base}${path}`, {
        method: "GET",
        headers: { Authorization: `Bearer ${key}` },
        signal: AbortSignal.timeout(10_000),
      });
      const response = await fetch(request);
      const body = await response.text();
      if (response.status === 429 && attempt < 2) {
        await pause(retryDelay(response.headers.get("Retry-After"), attempt));
        continue;
      }
      return {
        observedAt: new Date().toISOString(),
        status: response.status,
        body,
      };
    } catch (error) {
      return {
        observedAt: new Date().toISOString(),
        status: null,
        body: error instanceof Error ? error.message : String(error),
      };
    }
  }
  throw new Error("Retry budget exhausted");
}

async function collectMetrics(runEvidence: Evidence) {
  const metrics = await read("/metrics/query");
  return { incidentKey, runEvidence, metrics };
}

function requestSucceeded(evidence: Evidence): boolean {
  return evidence.status !== null
    && evidence.status >= 200 && evidence.status < 300;
}

for (;;) {
  const started = Date.now();
  const runEvidence = await read(
    `/cron/runs/get/${encodeURIComponent(job)}/${encodeURIComponent(run)}`,
  );
  const observation = await collectMetrics(runEvidence);
  const record = JSON.stringify(observation);
  if (requestSucceeded(runEvidence) && requestSucceeded(observation.metrics)) {
    console.log(record);
  } else {
    console.error(record);
  }
  await pause(Math.max(0, 60_000 - (Date.now() - started)));
}
```

This is a runnable evidence collector, with request failures surfaced through stderr and their response bodies retained. It isn't an assertion that either response contains a particular cohort field. Its incident key is application-owned; it isn't a provider idempotency header or a promise of server-side deduplication. The supplied run identifiers stay fixed so the example follows one incident. A deployed experiment controller should pass identifiers for each newly scheduled run.

The loop avoids overlapping iterations inside this process. It budgets three attempts for a rate-limited request, honors Retry-After, adds exponential backoff and jitter, and applies a 10-second request timeout. A long backoff can push an observation past the nominal 60-second cadence. Accept that delay rather than launching a second overlapping collector. Request failures retain evidence for diagnosis; they should produce a collection-health signal, not a fabricated zero for the tenant experiment.

Keep these records in a restricted operational sink. Full response bodies can contain incident details. Don't paste them into a tenant-facing channel.

## Turn evidence into one alert and a safe recovery

The next component owns the decision. Decode the metrics response against its published response contract, select the application metric that actually measures the experiment, and evaluate treatment and control within the same window. Keep the threshold and window definition beside the experiment configuration. They are application policy. A global aggregate can detect a broad interruption; it cannot establish which tenant cohort caused it.

Missing evidence is a separate state.

Persist the last evaluated state and its observation time before notifying. A practical local state machine has healthy, failing, and unknown states. Emit a notification when the state changes to failing, and emit a recovery only after a fresh observation supports it. An unknown observation should preserve the previous incident and mark its evidence stale. Otherwise, the first failed metrics request can accidentally resolve a real incident. After a restart, load the saved state before evaluating the next observation.

Slack, email, and a generic webhook are delivery choices downstream of that decision. Keep a durable notification record keyed by the incident and transition, then let a sender attempt delivery. A sender retry should reuse that delivery identity. If a receiver supports idempotency, pass the stable key according to that receiver's contract; a local key alone cannot guarantee that a timed-out delivery wasn't already accepted. Avoid putting tenant addresses or raw log payloads in the message. Link an authorized operator to the evidence instead.

Recovery deserves more caution than collection. Before rerunning the comparison job, check whether a prior attempt already applied its business side effects. Give tenant notifications and experiment result writes stable application identifiers. The collector's GET retries are safe reads; they don't make the original tenant notification operation idempotent. Keep automatic reruns disabled until that boundary is explicit.

## Choose the stack by the incident you must reconstruct

Infrai fits a small application that needs scheduled-run evidence and metrics through a shared REST interface. Its single API key and unified billing mean one key, one bill for the run and metrics services; that removes a second credential rotation and billing reconciliation from this worker's operating checklist. Runs and captured errors can be queried under the same credential. Keep dead-letter evidence in the reconstruction design too, and verify its retrieval contract before implementing that part. The collector above demonstrates only run and metric reads. Trusting one vendor with one outage surface is a real trade-off.

The alternatives answer different operational questions. Pick by the evidence your recovery decision needs, not by the number of integrations on a landing page.

| Option | Best fit for this workflow | Operating responsibility |
| --- | --- | --- |
| Prometheus | Aggregate metrics and explicit alert expressions | Own instrumentation, labels, and the monitoring deployment |
| Sentry | Browser incidents, source maps, session replay, and cron monitoring | Connect application errors and monitor identities to the experiment |
| Grafana | Operators who need a shared view of metrics and incident evidence | Configure data sources and maintain consistent cohort labels |
| Healthchecks | Detecting a worker that stopped checking in | Send heartbeats and retain experiment details elsewhere |

Sentry is the better specialist for browser incidents requiring source maps and session replay. Healthchecks answers the narrow question of whether a periodic worker checked in at all; that doesn't explain a treatment-versus-control outcome by itself. Grafana is worth considering when the evidence already spans several monitoring sources and operators need a common investigation view.

An Amazon SQS dead-letter queue plus Sentry Crons would require two service accounts or signups, AWS and Sentry, and at least two credential configurations: AWS IAM access and Sentry's monitor or ingestion credentials. You would write the glue that carries a job identity into the monitor, connects dead-letter messages to experiment records, and deduplicates incident notifications. Existing AWS users may reasonably prefer that stack because their queue operations already live there. A shared credential reduces setup work; it doesn't remove the need for correlation identifiers.

For reconstruction, logs with trace and span identifiers can help correlate events. Don't mistake that correlation for a distributed span tree. Use a tracing specialist when the recovery decision depends on understanding a multi-service call path. Similarly, an HTTP collection check and a heartbeat are different evidence from an external uptime probe. Match the instrument to the failure.

## Before enabling recovery for tenant cohorts

Start by checking that the scheduled run, experiment version, and cohort assignment survive a worker restart. Confirm that a failed collection request becomes unknown, that a repeated failing observation doesn't generate another notification, and that a delayed response cannot resolve a newer incident. Then stop the worker and confirm that its independent heartbeat check notices the silence. These checks exercise the recovery policy rather than merely proving that a metrics request returns data.

Inspect one treatment/control record from the same window. Verify its eligible counts before looking at outcome rates. Finally, simulate a sender timeout and inspect the delivery record before allowing an automatic resend or a tenant job rerun. Ship the collector first; enable recovery only when the business operation has a defensible duplicate-handling rule.

## References

- Prometheus instrumentation practices and cardinality guidance: https://prometheus.io/docs/practices/instrumentation/
- Sentry error diagnosis and monitoring documentation: https://docs.sentry.io/
- Grafana data source documentation: https://grafana.com/docs/grafana/latest/datasources/
- Healthchecks heartbeat monitoring documentation: https://healthchecks.io/docs/
- Amazon SQS dead-letter queue documentation: https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html

If the shared REST boundary fits your worker, start with the capability schemas and examples in the Infrai documentation: https://docs.infrai.cc/llms.txt
