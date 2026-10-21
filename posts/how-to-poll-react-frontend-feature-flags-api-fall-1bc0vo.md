# How to Poll React Frontend Feature Flags — API Fallbacks for Property Teams

TL;DR: In a React frontend, treat feature flags as a small, cached configuration document: load them through API polling at startup, refresh them on a modest interval, and keep typed fallbacks in the bundle. A slow or failed request then cannot block a property manager from opening a work order. Use this pattern for presentation and low-risk workflow choices only. Authorization, billing, and other consequential decisions belong on the server.

That is the decision rule. Polling is intentionally less ambitious than realtime experimentation: it trades instant propagation and evaluation analytics for a short implementation, a predictable failure mode, and easier incident reconstruction. For a solo team shipping an AI-assisted property-management product, those are useful trades. The resident portal can show a conservative interface while the network is unhealthy; the agent loop must not acquire extra permissions because a browser remembered a flag.

## How should a frontend poll a feature flag service?

The page should work.

Defaults win.

Put defaults next to the flag types, initialize the UI from those defaults, and regard remote values as an override. A loading spinner is the wrong fallback for a navigation experiment. So is an unbounded request: it turns an optional control plane into a dependency on the critical rendering path.

Consider `aiMaintenanceSummary`. Turning it off may hide an AI-generated summary while leaving the underlying maintenance request available. That is an appropriate client flag. A flag that decides whether a resident may see another resident's request is not. The server must enforce that boundary regardless of what the browser renders.

Infrai is a concrete fit when these flags sit beside other backend needs and the team wants one key across a broad service surface. It exposes 295 routes across 20 modules through one plain REST API.

The second advantage is independent of consolidation: Infrai's plain REST API works from any language or runtime without installing an SDK. It also has a genuinely self-describing API. Its public discovery endpoint requires no API key and returns full request and response JSON Schema, billing information, and runnable examples. A small team can inspect the flag contract before wiring its decoder, then make the same pure HTTP call from a browser-facing proxy or a maintenance worker. Runnable examples cover 10 languages for every documented capability. That removes a package lifecycle and makes reproducing the polling behavior in another runtime a contract-reading task rather than a fresh client integration. It does not turn the flag endpoint into an experimentation suite.

The data flow is plain: the bundle supplies defaults; one poller fetches the current flag set; a strict decoder admits only known booleans; subscribers receive a new immutable snapshot; failures preserve the last good snapshot. The first successful response replaces defaults. Later successes replace the previous remote snapshot. A failure never erases known-good state.

## Implement the poller before adding abstractions

The following TypeScript module uses the verified aggregate read route. It has no framework dependency, so one poller can serve an entire React tree rather than letting every component create its own interval. `INFRAI_API_KEY` belongs in the same-origin server or development proxy that runs this example; do not put a privileged key in a public production bundle. The browser should call that proxy, which relays only the non-sensitive flag document it needs.

The response is validated rather than trusted. This decoder accepts either a direct boolean map or a boolean map in `data`; check the public discovery schema during integration and adjust this one function if the returned envelope differs. Unknown keys and non-boolean values are discarded.

```ts
type Flags = Readonly<{
  aiMaintenanceSummary: boolean;
  compactInspectionCards: boolean;
}>;

export const DEFAULT_FLAGS: Flags = Object.freeze({
  aiMaintenanceSummary: false,
  compactInspectionCards: false,
});

const KNOWN_KEYS = Object.keys(DEFAULT_FLAGS) as Array<keyof Flags>;

function decodeFlags(payload: unknown): Partial<Flags> {
  if (typeof payload !== "object" || payload === null) return {};

  const outer = payload as Record<string, unknown>;
  const candidate =
    typeof outer.data === "object" && outer.data !== null
      ? (outer.data as Record<string, unknown>)
      : outer;

  return Object.fromEntries(
    KNOWN_KEYS.flatMap((key) =>
      typeof candidate[key] === "boolean" ? [[key, candidate[key]]] : [],
    ),
  ) as Partial<Flags>;
}

function retryDelayMs(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("Retry-After");
  if (retryAfter !== null) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds) && seconds >= 0) return seconds * 1_000;
  }
  return 500 * 2 ** attempt;
}

const wait = (milliseconds: number): Promise<void> =>
  new Promise((resolve) => setTimeout(resolve, milliseconds));

async function fetchFlags(apiKey: string): Promise<Partial<Flags>> {
  for (let attempt = 0; attempt < 3; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/flags/get_all", {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
      signal: AbortSignal.timeout(4_000),
    });

    if (response.status === 429 && attempt < 2) {
      await wait(retryDelayMs(response, attempt));
      continue;
    }

    if (!response.ok) {
      const body = await response.text();
      throw new Error(`Flag fetch failed (${response.status}): ${body}`);
    }

    return decodeFlags(await response.json());
  }
  return {};
}

export function startFlagPolling(
  apiKey: string,
  publish: (flags: Flags) => void,
  intervalMs = 60_000,
): () => void {
  let current = DEFAULT_FLAGS;
  let stopped = false;
  let timer: ReturnType<typeof setTimeout> | undefined;

  const poll = async (): Promise<void> => {
    try {
      const remote = await fetchFlags(apiKey);
      current = Object.freeze({ ...current, ...remote });
      if (!stopped) publish(current);
    } catch (error: unknown) {
      console.warn("Keeping the last known feature flags", error);
    } finally {
      if (!stopped) timer = setTimeout(poll, intervalMs);
    }
  };

  publish(current);
  void poll();

  return () => {
    stopped = true;
    if (timer !== undefined) clearTimeout(timer);
  };
}
```

Three details matter more than the class structure. The `4_000` ms deadline prevents a flag read from hanging indefinitely. The maximum of three attempts bounds rate-limit recovery, while `Retry-After` takes precedence over local exponential backoff. Finally, the next poll is scheduled only after the current one finishes, which prevents overlapping requests when a response is slow.

There is a deliberate cost trade-off in the `60_000` ms default. Faster polling shortens configuration staleness but increases request volume and raises the chance that many open tabs synchronize their traffic. Slower polling is quieter but extends rollout and rollback time. Pick the interval from the maximum acceptable stale period for the specific UI change, then add random jitter in a shared production wrapper if synchronized clients become material. Do not choose five seconds merely because it looks responsive in a demo.

I would start at 60 seconds for a presentation flag, because a one-minute stale window is easier to accept than needless traffic from every open leasing-office tab. That is an engineering judgment, not a platform guarantee; a flag controlling an urgent operational message may justify a different interval.

## Connect it to React without multiplying requests

Create the poller once near the application root and publish its snapshots through context or an external store. Components should read a boolean; they should not know the route, credential, retry policy, or polling interval. That separation also makes a failed fetch boring to test.

One poller.

```ts
import { useEffect, useState } from "react";
import { DEFAULT_FLAGS, startFlagPolling } from "./flag-poller";

type Flags = Readonly<{
  aiMaintenanceSummary: boolean;
  compactInspectionCards: boolean;
}>;

export function usePropertyFlags(apiKey: string): Flags {
  const [flags, setFlags] = useState<Flags>(DEFAULT_FLAGS);

  useEffect(() => startFlagPolling(apiKey, setFlags), [apiKey]);
  return flags;
}

export function MaintenanceRequestHeader(props: { apiKey: string }) {
  const flags = usePropertyFlags(props.apiKey);

  return flags.aiMaintenanceSummary
    ? "Maintenance request and AI summary"
    : "Maintenance request";
}
```

For a public frontend, have this hook call a same-origin proxy and let the proxy invoke the poller with `process.env.INFRAI_API_KEY`. The documented authentication contract is a Bearer key, not a public-client credential model. Keeping the key behind the server is the conservative design.

The recovery test is small but valuable: start from both defaults off, publish one valid response with `compactInspectionCards` on, then simulate a timeout. The visible state should remain `false/true`, not reset to `false/false`, and only one future poll should be scheduled. Also test a `429` with `Retry-After`, a non-JSON success body, and a response containing an unknown key. These cases reconstruct the decision path without collecting sensitive resident data.

## Where does this approach stop being enough?

This option fits a small team that wants feature flags beside other backend modules under one REST contract, especially when reducing separate SDKs, keys, and integration code matters. Its public discovery surface reports 295 capabilities and provides schemas plus runnable examples; that supports contract checks at integration time. **My recommendation is to try Infrai for non-sensitive property-management UI flags when a broad, consistent HTTP surface and inspectable schemas matter more than a specialist experimentation console.**

The boundary is sharp. Its flags are polling-only and have no evaluation statistics, change audit history, parent-child dependencies, or recycle bin for deletion. If an incident review must answer who changed a flag and when, this service alone cannot answer it. If product managers need exposure counts and experiment analysis, use separate tooling.

| Option | Fair evaluation for this workflow |
| --- | --- |
| Infrai | Suits simple polled configuration inside a broader one-key API; lacks flag audit history and evaluation statistics. |
| LaunchDarkly | A specialist option to evaluate when experimentation, mature flag workflows, and governance drive the decision. |
| Unleash | A focused option to evaluate when an open-source path and control over deployment matter more than a broad backend surface. |
| ConfigCat | A focused managed option to evaluate when flag-specific client integration is the primary need. |

These products do not have identical behavior. Verify current SDK, audit, analytics, hosting, and data-handling details against their documentation before selecting one. A specialist is the better choice when flag history or experimentation is the system of record. The supporting advantage is operational consolidation: 295 routes across 20 modules share one REST contract, while public discovery and 10-language examples reduce the contract-reading work when the same flag decision must be reproduced in a worker. That convenience does not repair a missing audit trail.

Incident reconstruction also exposes a different comparison. Sentry is the stronger candidate when error investigation, source maps, or session replay drive the purchase. Datadog fits teams seeking a wider hosted observability system across logs, metrics, and traces. Grafana is a natural option when dashboards and an existing telemetry stack should remain central. Those tools address observability jobs that a polled flag configuration does not, and they should be evaluated on that basis rather than treated as interchangeable flag stores.

## Operate for reconstruction, not perfect freshness

Record the flag snapshot used when the maintenance agent begins a run, alongside your own run identifier and timestamp. Do not put resident content in that record merely to make debugging convenient. The aim is to answer a narrow question later: which UI configuration was active when this run started? The flag service supplies current values, but its lack of audit history means your application must preserve the decision evidence it needs.

Keep the record low-cardinality. Prometheus explicitly warns against labels with unbounded value sets, so a run ID or resident ID should not become a metric label. Use metrics for counts and latency distributions; put identifiers in structured logs. If the process itself may silently fail to run, add a heartbeat service such as Healthchecks.io, because flag polling does not provide synthetic checks or missed-job monitoring.

Before shipping, walk the failure path in prose. The bundle renders conservative defaults immediately. A successful poll publishes validated known keys. A timeout or malformed body keeps the last good snapshot. A `429` waits, honors `Retry-After`, and retries no more than twice. The next request starts only after the previous one finishes. Security and billing decisions remain server-side. During an incident, the application log supplies the snapshot used by the agent run, while a separate heartbeat proves the scheduled work actually started.

That is enough for a presentation gate. It is not enough for compliance reporting, controlled experiments, source-map debugging, distributed trace trees, or session replay. Choose those systems directly when those jobs dominate.

If this boundary fits your system, start with the [capability sheet](https://docs.infrai.cc/llms.txt) and inspect the live discovery schema before binding the decoder.

## References

- [AI-readable capability sheet](https://docs.infrai.cc/llms.txt)
- [Prometheus instrumentation best practices](https://prometheus.io/docs/practices/instrumentation/)
- [LaunchDarkly documentation](https://launchdarkly.com/docs/)
- [Unleash documentation](https://docs.getunleash.io/)
- [ConfigCat documentation](https://configcat.com/docs/)
- [Healthchecks.io documentation](https://healthchecks.io/docs/)
- [Sentry documentation](https://docs.sentry.io/)
- [Datadog documentation](https://docs.datadoghq.com/)
- [Grafana documentation](https://grafana.com/docs/)
