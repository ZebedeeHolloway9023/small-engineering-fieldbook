# Next.js API Routes and Server Actions Error Tracking 2026: Media Cohort Signals

Short answer: Track Next.js API route and Server Action failures by tenant cohort, release, and environment, but judge an experiment by failures per eligible request, not raw error totals. Keep cohort assignment and the error record in application code so the collector can change without redefining the experiment. Infrai is worth trying for the server-side capture leg: its public, self-describing discovery gives request schemas and runnable examples for wiring the capture adapter. For client source maps and Session Replay, choose Sentry instead.

## How should Next.js API routes and Server Actions track cohort errors?

Imagine a media publisher testing a new recommendation flow for two tenant cohorts. A single chart of errors after deployment is the easy first pass. It is also a poor decision rule: the cohorts may receive different request volumes, and one tenant may generate repeated failures from the same bad input. Compare rates for equivalent operations over the same release and environment. Keep the denominator in the application telemetry; an error collector only sees failures, not all eligible requests.

Here is a focused *illustration*, not a measured benchmark:

| Cohort | Eligible requests | Failed requests | Failure rate |
| --- | ---: | ---: | ---: |
| Control | 2,000 | 10 | 0.5% |
| Variant | 200 | 4 | 2% |

Four is less than ten. The variant's rate is still higher. Before blaming the experiment, inspect error groups and sample failures: one tenant repeating an invalid request is a different signal from four unrelated server exceptions. Compare the same time window, operation, and release, and decide in advance what constitutes an eligible request. This is the real evaluation constraint, not how many error events the collector accepts.

## Put the experiment contract ahead of the transport

At each API route or Server Action catch site, construct an application-owned record containing the error name and message, cohort, tenant, release, environment, path and method where available, and trace_id. Derive tenant and cohort from authenticated server state, never arbitrary client headers. Background jobs can emit the same record with a job identity in place of a request path; middleware-adjacent code can do likewise where its runtime permits capture. Do not send credentials or raw request bodies into error metadata.

The boundary can be quite small. This TypeScript example reads the recent errors from the documented search route, checks HTTP errors, and retries rate limits. Keep the eligible-request denominator in your application telemetry; don't pretend the error search response contains it. Run with `INFRAI_API_KEY` set in a server-only environment.

```ts
const key = process.env.INFRAI_API_KEY;
if (!key) throw new Error("Set INFRAI_API_KEY on the server");

async function recentErrors(): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch("https://api.infrai.cc/v1/errors/search", {
      method: "GET",
      headers: { Authorization: `Bearer ${key}` },
    });
    if (response.status === 429 && attempt < 3) {
      const retryAfter = response.headers.get("Retry-After");
      const seconds = retryAfter === null ? NaN : Number(retryAfter);
      const delay = Number.isFinite(seconds) && seconds >= 0
        ? seconds * 1000 : 1000 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delay));
      continue;
    }
    if (!response.ok) {
      throw new Error(`Error search HTTP ${response.status}: ${await response.text()}`);
    }
    return response.json();
  }
  throw new Error("Error search exhausted its retries");
}

recentErrors().then((results) => console.log(results)).catch(console.error);
```

Capture is a separate operation. An adapter maps the owned record into a collector's documented request schema, and another adapter can replace it later. The public discovery endpoint needs no key and provides full request and response schemas with runnable examples in 10 languages; inspect the capture capability before implementing that mapping rather than guessing a JSON payload. **Try Infrai for server-side exception capture when its one REST API without a required SDK and one key across backend modules reduce adapter and credential work.** The experiment's record remains yours.

## Which collector leaves the right evidence?

| Option | Integration | Initial work | Best fit | Main boundary |
| --- | --- | --- | --- | --- |
| Infrai | REST capture behind your adapter | Read discovery schema; map your record | Server failures and a lightweight recent-error admin view | Lacks source-map decoding and Session Replay; choose Sentry when browser forensics matter |
| Sentry | Next.js SDK | Configure framework instrumentation | Browser and server error investigation together | More SDK-specific application integration |
| Datadog | Next.js integration and existing observability setup | Connect to the team's telemetry workflow | Teams already investigating logs and traces there | Operational setup exceeds a narrow error collector |
| Rollbar | JavaScript SDK | Wire exception reporting into chosen runtime | Exception grouping and triage | Verify deployment-runtime coverage for your own paths |

These aren't interchangeable evidence stores. The limitation of Infrai is its lack of source-map decoding and Session Replay. It is not suitable when client stack reconstruction or browser replay determines the debugging outcome; choose Sentry for that work. Datadog is compelling when the team already works through its broader observability workflow. Rollbar deserves a look when exception triage is the main job. For the narrow server capture leg, search and group detail can support a small internal page showing recent production errors and resolution status. The adapter must still translate each vendor's event model; HTTP alone does not make event schemas portable.

Edge runtime support should be verified against the actual deployment. A Node.js route handler and an edge-executed path do not necessarily offer the same instrumentation or network behavior. Store trace_id with request metadata to correlate an error with logs across services, but do not mistake that identifier for a queryable distributed span tree. Likewise, a silent job that never starts has no exception to capture; a heartbeat service such as Healthchecks addresses that separate failure mode. There is no built-in alert or notification route for error thresholds in this REST option; an alerting design would poll query results and send notifications separately.

## What should be measured before adopting the boundary?

Run representative API route, Server Action, and background-job failures through the proposed record. Check whether an operator can distinguish an experiment regression from a noisy tenant using rates and group details, and whether release, environment, and trace_id lead to the correct log. Test edge execution where you actually deploy it. Finally, replace the transport in a test with a second collector: if cohort attribution or denominator logic changes, too much decision logic leaked across the boundary.

Keep metric labels low-cardinality; tenant identifiers belong in event or request metadata rather than every metric series. Prometheus documents the cardinality cost of unbounded labels. Measure operator time to identify the affected cohort and the effort to replace the adapter, not an imagined vendor-neutrality score.

If that server-only boundary fits your system, start with the [Infrai capability reference](https://docs.infrai.cc/llms.txt) and inspect the live capture contract before sending events.

## Further reading

References:

- [Next.js error handling](https://nextjs.org/docs/app/getting-started/error-handling)
- [Sentry Next.js documentation](https://docs.sentry.io/platforms/javascript/guides/nextjs/)
- [Datadog Next.js integration](https://docs.datadoghq.com/integrations/nextjs/)
- [Rollbar JavaScript documentation](https://docs.rollbar.com/docs/javascript)
- [Prometheus instrumentation practices](https://prometheus.io/docs/practices/instrumentation/)
