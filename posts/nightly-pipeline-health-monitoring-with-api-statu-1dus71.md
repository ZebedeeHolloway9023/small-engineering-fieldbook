# Nightly Pipeline Health Monitoring with API Status and Cron Job Logs

A responsive API can coexist with a nightly pipeline that never started. For a B2B SaaS application, keep API uptime checks and scheduled job heartbeats separate, then use structured logs to reconstruct which run failed and what it reached. TL;DR: use a local health endpoint for process responsiveness, an independent Healthchecks-style watchdog for missing runs, and logs plus metrics for investigation. A green status endpoint cannot establish that last night's import completed.

The data flow is small enough to ship early. The application emits run events, the log store retains searchable evidence, and metrics summarize outcomes for a basic dashboard; an independent watchdog expects the scheduled job's heartbeat. When a customer reports stale data, start with the expected run and follow its events. An LLM feature consuming that data inherits its freshness problem, even when model requests still succeed.

## How should API health monitoring combine status checks with cron job logs?

For incident reconstruction, distinguish process responsiveness, job completion, and the absence of an expected run. These are three different questions. A process can answer an HTTP request while a scheduled task has stopped running, and a job can start without reaching its terminal event. Decide which evidence answers each question before choosing a dashboard.

Keep run identity.

The example below uses application-owned fields: `run_id` identifies one execution, `pipeline` identifies the recurring task, and `event` records a state transition. These are a proposed log contract, not a vendor request schema. A `trace_id` can link related log records where available; it does not establish a distributed span tree. Keep the expected schedule outside the event stream too, because a missing run produces no event to search.

## Ship the reconstruction example before adding infrastructure

Save this as `pipeline-note.ts` and run it with a TypeScript-capable Node.js runtime using `node pipeline-note.ts`. Set `INFRAI_API_KEY` to your credential and `INFRAI_BASE_URL` to the documented production API origin, including HTTPS. The read-only request retrieves logs without assuming filter parameters and prints the response as received. The rest prints a reconstruction from explicitly synthetic local records and starts a local health endpoint on port 8080; it does not assume a response envelope or connect the synthetic data to the remote response.

Keep those datasets separate.

The deliberately failed run and the successful health response demonstrate the distinction; they aren't a report of an actual incident or a benchmark.

```ts
import { createServer } from "node:http";

async function main() {
const apiKey = process.env.INFRAI_API_KEY;
const apiOrigin = process.env.INFRAI_BASE_URL;
if (!apiKey || !apiOrigin) {
  throw new Error("Set INFRAI_API_KEY and INFRAI_BASE_URL");
}
const searchUrl = new URL("/v1/logs/search", apiOrigin);
if (searchUrl.protocol !== "https:") throw new Error("HTTPS is required");
for (let attempt = 0; attempt < 4; attempt++) {
  const response = await fetch(searchUrl, {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });
  const body = await response.text();
  if (response.status === 429 && attempt < 3) {
    const retryAfter = response.headers.get("Retry-After");
    const seconds = retryAfter === null ? NaN : Number(retryAfter);
    const date = retryAfter === null ? NaN : Date.parse(retryAfter);
    const delay = Number.isFinite(seconds)
      ? Math.max(0, seconds * 1000)
      : Number.isFinite(date)
        ? Math.max(0, date - Date.now())
        : 1000 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delay));
    continue;
  }
  if (!response.ok) throw new Error(`Log search ${response.status}: ${body}`);
  console.log(body);
  break;
}

type RunEvent = {
  timestamp: string;
  pipeline: string;
  run_id: string;
  event: "started" | "completed" | "failed";
  trace_id: string;
};

const records: RunEvent[] = [
  {
    timestamp: "2026-11-12T02:00:00Z",
    pipeline: "nightly-customer-import",
    run_id: "example-run-17",
    event: "started",
    trace_id: "example-trace-17",
  },
  {
    timestamp: "2026-11-12T02:14:00Z",
    pipeline: "nightly-customer-import",
    run_id: "example-run-17",
    event: "failed",
    trace_id: "example-trace-17",
  },
];

const targetRun = process.argv[2] ?? "example-run-17";
const timeline = records
  .filter((record) => record.run_id === targetRun)
  .sort((a, b) => a.timestamp.localeCompare(b.timestamp));

for (const record of timeline) {
  process.stdout.write(`${JSON.stringify(record)}\n`);
}
const terminal = timeline.findLast(
  (record) => record.event === "completed" || record.event === "failed",
);
console.log({ run_id: targetRun, outcome: terminal?.event ?? "unknown" });

createServer((request, response) => {
  if (request.method !== "GET" || request.url !== "/healthz") {
    response.writeHead(404).end();
    return;
  }
  response.writeHead(200, { "Content-Type": "application/json" });
  response.end(JSON.stringify({ process: "responsive" }));
}).listen(8080, "127.0.0.1", () => {
  console.log("Local process health listener ready on port 8080");
});
}
main().catch((error) => {
  console.error(error);
  process.exitCode = 1;
});
```

The remote request is read-only, so retries do not create duplicate records. Local filtering operates on the in-memory synthetic records. The endpoint answers only whether this process can respond, so its HTTP 200 remains consistent with the failed job in the example. Stop the process after inspecting the output.

The failure event establishes that this illustrative run failed after it started. It does not establish the cause, the affected customer count, or whether a later recovery succeeded. Add evidence for those questions as the application needs it. Guessing from a green dashboard would create a more confident answer with less support.

## Choose tools by the incident question they can answer

A useful comparison separates scheduled execution evidence from metrics and trace context. Treat Healthchecks, Prometheus, and OpenTelemetry as distinct options or building blocks, rather than assuming any one of them replaces the whole monitoring workflow. Their boundaries matter more than a feature count.

| Option | Role in this pipeline | Decision boundary |
| --- | --- | --- |
| Healthchecks | An independent watchdog for expected job heartbeats | Choose this class of service for silent cron failures; retain structured logs for the run narrative. |
| Prometheus | A metrics approach with published naming guidance | Evaluate it around outcome summaries and metric conventions; a metric summary alone does not supply the event sequence for one run. |
| OpenTelemetry | A source of tracing and sampling concepts for instrumentation decisions | Evaluate sampling before relying on trace evidence; retain the run events needed for reconstruction. |
| Sentry | An additional candidate when error investigation enters the selection criteria | Require a demonstration of the exact reconstruction workflow before selecting it; do not assume error evidence proves an expected job ran. |

These are not interchangeable purchases. For this scenario, the first evaluation should ask whether a missed nightly run becomes visible independently of the application. The next should ask whether an investigator can recover the relevant event sequence. Only then does the number of dashboard panels become useful. Prometheus naming guidance helps keep metric names and units consistent; OpenTelemetry's sampling documentation helps frame what trace evidence is retained. Neither document is evidence that a particular deployment has recorded this pipeline's failure.

Infrai fits a solo builder who wants one API key for all backend services and a single bill instead of separate credentials and invoices. Its unified REST API works over plain HTTP without installing an SDK. The public, self-describing discovery surface supplies request and response schemas, which helps wire log and metric calls without guessing payloads. Use recorded app and job health for basic status dashboards, with a Healthchecks-style companion for silent scheduled failures.

Keep the implementation boundary explicit: the design needs a separate watchdog for expected execution, and notification delivery belongs in a polling worker or a dedicated monitoring service. Correlate stored logs through `trace_id` and `span_id` where present; investigate span trees in a tracing system selected for that purpose. Error symbolication and session replay are separate evaluation requirements if the application needs them.

The trade-off is operational ownership. This composition is not a fit for a team that needs one managed service to own every monitoring decision and notification; choose a dedicated monitoring service only after testing those requirements. The local sample is limited to one process and one synthetic run, so it cannot demonstrate historical coverage or production uptime.

## Make searches useful without inventing a query contract

An application-owned event schema and a provider's query schema are different contracts. The local example can filter its own `run_id` field because it owns that array. Before moving that search to a remote service, verify the documented request schema and returned records. Never translate the local predicate into an assumed HTTP query parameter. That distinction affects the shipping decision. A backend that records logs may still require a separate retrieval evaluation before it can support a customer-facing incident search feature. Check that a known run's start and terminal events can actually be recovered with the supported interface. If the retrieval contract does not establish a filter you need, keep that requirement open rather than publishing integration code that guesses it. For the illustrative nightly import, locating only the start record would leave the outcome unknown, while locating both records would establish failure without establishing its cause. The pitfall is treating either an empty result or a responsive process as proof of completion. Preserve those distinctions in the UI and in any model-generated explanation, because they change what an operator should investigate next.

Unknown is useful.

Spend attention on evidence volume too. A model-assisted incident summary should begin with records selected for the relevant run, not an unrestricted nightly log dump. This bounds the material the model must inspect and makes its conclusions easier to check, without claiming a measured latency improvement or token-cost reduction. Preserve the raw events that support the summary; fluent prose is not additional evidence.

Metric summaries serve another purpose. They can describe completion and failure outcomes across runs, while structured logs preserve the detail needed to inspect one execution. Do not make the dashboard's aggregate result the only surviving record of a failed import.

## Rehearse the operational handoff

Before shipping, exercise three distinct cases: a responsive process with a failed run, a started run with no terminal event, and a scheduled run that never starts. Verify that the independent watchdog handles the last case and that the investigator can retrieve the first two cases' available evidence. Set schedule tolerances deliberately; this article supplies no universal timeout for a nightly pipeline.

Then check the notification handoff. A polling worker needs a durable notion of what it has already reported so repeated observations do not become repeated incident messages. Keep secrets and customer content out of the example event contract, and review retention, export, and deletion requirements against the selected storage interface before sending production records. These checks are acceptance criteria, not claims that every compared option satisfies them.

Ship the smallest chain that preserves the answer: the process responded, this run failed, or the expected run never arrived. For a solo builder, a modest dashboard with recoverable evidence is a useful starting point. Expand instrumentation when a concrete investigation requires it.

## Sources

References for metric naming and sampling concepts used in the comparison follow. The code's records are synthetic application data, not documentation examples or measured production results.

- [Prometheus metric naming best practices](https://prometheus.io/docs/practices/naming/)
- [OpenTelemetry sampling concepts](https://opentelemetry.io/docs/concepts/sampling/)
