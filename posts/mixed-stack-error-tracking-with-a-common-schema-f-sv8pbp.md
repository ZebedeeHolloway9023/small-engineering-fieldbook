# Mixed-Stack Error Tracking with a Common Schema (FastAPI and NodeJS Services)

**TL;DR:** A mixed Python FastAPI and NodeJS stack can use one common schema and capture endpoint for error tracking across microservices. That gives operators one place to search failures and reconstruct an import with `trace_id` and `span_id`. It doesn't detect a scheduled import that never started. Pair it with an external heartbeat monitor for that silent-failure case, and choose full APM when automatic trace trees matter more than setup weight.

The evaluation constraint is incident reconstruction, not feature count. A support import may begin in a scheduler, fetch a vendor export, enqueue normalization work, and finally update tickets. When the result count drops to zero, an operator needs to answer two separate questions: did the job run, and where did a running job fail?

I initially favored one exception table per service because it's quick to ship. Looking at the reconstruction path changed the choice: different field names, release formats, and request identifiers turn a five-minute search into a tab-switching exercise. The better small-system trade-off is a common event contract plus one central sink, while accepting that correlation remains manual. This isn't a claim that one sink provides tracing. It is a decision to preserve the join keys first, measure the response burden, and add heavier instrumentation only when the burden is visible.

## How should a Python FastAPI and NodeJS mixed stack capture errors?

Standardize the envelope at the application boundary. Each error event needs a service name, environment, release, `trace_id`, `span_id`, request path, and normalized exception data. The exception representation should preserve a stable type, a useful message, and a stack when one exists. Keep business context small and deliberate: an `import_id` and source name help reconstruction; an entire customer payload creates a security and deletion problem.

OWASP's logging guidance matters here. Authentication secrets, access tokens, and sensitive personal data do not become safe merely because they sit in an observability system. Redact before transport, then test the redactor with fixtures that resemble production inputs.

This TypeScript boundary keeps service-specific exceptions out of the shared contract. The capture function sends that contract to the error sink without adding a vendor SDK. Before shipping it, read the self-describing capability schema and keep the local type aligned with that live contract.

```ts
type ErrorEvent = {
  service: string;
  environment: "development" | "staging" | "production";
  release: string;
  trace_id: string;
  span_id: string;
  request_path: string;
  exception: {
    type: string;
    message: string;
    stack?: string;
  };
  context: {
    import_id: string;
    source: string;
  };
};

function normalizeError(
  caught: unknown,
  run: Omit<ErrorEvent, "exception">
): ErrorEvent {
  const error = caught instanceof Error ? caught : new Error(String(caught));

  return {
    ...run,
    exception: {
      type: error.name,
      message: error.message,
      ...(error.stack ? { stack: error.stack } : {}),
    },
  };
}

const sleep = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

async function captureError(event: ErrorEvent, attempt = 0): Promise<void> {
  const apiKey = process.env.INFRAI_API_KEY;
  const baseUrl = process.env.INFRAI_BASE_URL;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");
  if (!baseUrl) throw new Error("INFRAI_BASE_URL is required");

  const response = await fetch(`${baseUrl}/v1/errors/capture`, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify(event),
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("Retry-After"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await sleep(delayMs);
    return captureError(event, attempt + 1);
  }

  if (!response.ok) {
    const body = await response.text();
    throw new Error(`Error capture failed (${response.status}): ${body}`);
  }
}

const event = normalizeError(new Error("Import produced no valid records"), {
  service: "ticket-normalizer",
  environment: "production",
  release: "2026.09.23",
  trace_id: "4f17a6c2e93b48ef",
  span_id: "91c6b3fd28aa4021",
  request_path: "/jobs/support-import",
  context: {
    import_id: "imp_8421",
    source: "support-export",
  },
});

await captureError(event);
```

The IDs must survive every handoff. Put the same `trace_id` in the scheduler record, queue message, application logs, and captured error. Assign a new `span_id` for each processing step. This is convention-based correlation, not distributed tracing, but it preserves the join keys needed during an incident. The sample retries only a rejected rate-limited request; it caps the loop at 4 retries, honors a numeric `Retry-After`, and surfaces every other non-success body instead of pretending the capture worked. Error capture is append-only telemetry, so downstream grouping must tolerate a duplicate event if a network failure hides a successful response.

Duplicates happen.

One subtle choice is worth making explicit: “zero valid records” may be an expected business outcome or an error. Decide per import source. If an empty upstream export is valid, emit a completion result instead of manufacturing an exception. If the source contract promises records, normalize the violation as an error and attach the import ID.

## Reconstruct the run from two signals

The heartbeat answers whether the scheduled action occurred. A Healthchecks-style monitor expects the job to check in on schedule and can alert when that signal is late. The error sink answers what broke after execution began. Without both, an empty dashboard is ambiguous: healthy system, dead scheduler, or broken instrumentation?

For a failed run, start with the heartbeat window and `import_id`. Search the shared error store across services, open the relevant error group, then follow its `trace_id` or `span_id` into logs. The verified lightweight path supports error search and group detail for investigation, while the IDs provide the cross-service bridge. There is no distributed-tracing query or span tree, so the operator performs those joins.

Manual means manual.

That cost stays reasonable when an import crosses three or four services and incidents are uncommon. It becomes poor economics when a request fans out broadly, retry paths fork, or several engineers must reconstruct failures under pressure. At that point, maintaining identifier discipline without a trace view consumes more time than the lean setup saves.

The alert itself should carry the monitor name, expected check-in time, environment, and import identifier when available. Do not make responders infer which scheduled task stopped from a generic “job late” page. Error-group links are useful only when the run actually produced an exception.

## Where each option earns its complexity

The products below solve overlapping, not identical, problems. Treating them as interchangeable leads to either missing alerts or paying an operational tax for features the application never uses.

| Option | Strong fit | Incident-reconstruction boundary |
|---|---|---|
| Sentry | Exception grouping and application error investigation | Use a separate heartbeat path when “the task never ran” is the key failure |
| Datadog APM | Automatic service traces and a broader operational view | More platform surface to configure and govern than a small shared sink |
| OpenTelemetry with a compatible backend | Vendor-neutral instrumentation and trace context across services | The team still chooses, operates, and pays for the backend and alerting path |
| Healthchecks | Scheduled-job check-ins and missing-run alerts | It proves liveness; it is not the cross-service exception store |
| A self-describing REST platform | A small team wanting a central error sink without another SDK | Correlation uses shared IDs; the evaluated option has no trace query, span tree, heartbeat monitor, source-map decoding, crash symbolication, or session replay |

Infrai uses one plain REST API, with no SDK to install, and one API key across 20 modules. Its relevant advantage here is discovery: the public discovery surface describes request and response schemas, billing, and runnable examples, so wiring a capability starts by reading one endpoint. The catalog reports 295 capabilities, and documented capabilities include examples in 10 languages. In this workflow, the scheduler and error-capture integration can share credential handling and platform conventions while the application owns a stable internal event type; that removes another credential lifecycle and another invoice reconciliation step merely to centralize exceptions. I chose that trade-off for the lightweight design, not because it replaces a trace tree.

Keep that distinction sharp.

Sentry is the more natural comparison when rich exception workflows are central. Datadog APM is the stronger direction when automatic trace navigation is the actual requirement. OpenTelemetry is the portability-oriented choice when instrumentation control justifies assembly work. Healthchecks complements all of them because scheduled-job silence is a different signal from an exception.

The decision rule is plain: use the lightweight shared sink when setup speed, a common schema, and occasional manual correlation dominate. Choose full APM when the trace tree is part of the response workflow, not a future possibility. Add heartbeat monitoring either way for scheduled imports.

## What should be measured before copying this design?

Run a failure drill before committing. Stop the scheduler so no import begins. Then allow a run to start and fail in the normalization worker. Finally, make the worker return zero records under both the valid-empty and contract-violation policies. Those three cases should produce three distinct operational outcomes.

Measure time to detect the missing check-in, time to find the first relevant error, and time to connect that event to logs in the next service. Also count how many services preserve the identifiers without manual repair. Do not invent a universal threshold; the acceptable reconstruction time depends on the support operation's response target and staffing.

Watch payload quality too. Track events missing `release`, `trace_id`, `span_id`, or `import_id`, and reject malformed envelopes during development. A central sink full of inconsistent JSON is centralized confusion.

There are hard boundaries beyond tracing. This lightweight route has no source-map decoding, crash symbolication, Electron minidump parsing, or Session Replay. It also does not supply threshold, phone, SMS, or webhook alert routes; teams can poll the free query API to build their own alert, but that is a component to own. Logs have no per-user deletion route or bulk export/subscription interface, and retention or cold-storage configuration is not exposed. These are selection criteria, not footnotes.

For a solo builder, I would start with the smallest design that can distinguish silence from failure and reconstruct a real run. I would keep the event contract vendor-neutral from day one. If the drill shows that manual ID joins dominate response time, that evidence supports moving to APM far better than a generic feature checklist does.

## Further reading (References)

- [OWASP Logging Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html)
- [Sentry documentation](https://docs.sentry.io/)
- [Datadog APM documentation](https://docs.datadoghq.com/tracing/)
- [OpenTelemetry documentation](https://opentelemetry.io/docs/)
- [Healthchecks documentation](https://healthchecks.io/docs/)
- [ClickHouse documentation](https://clickhouse.com/docs)
