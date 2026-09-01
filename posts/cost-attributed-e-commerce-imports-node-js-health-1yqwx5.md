# Cost-Attributed E-commerce Imports: Node.js Health Endpoints, Metrics, and Logging

Short answer: treat a scheduled import's last successful result as durable state, then let `/ready` report whether that result is still fresh, keep `/live` limited to process responsiveness, and use `/health` as a small operator-facing summary. Emit one structured completion log and two bounded metrics per import so an alert can detect silence and finance can attribute work to a stable cost center.

The key change is small. Before: "the cron process is running." After: "the catalog import for cost center `merchandising-eu` produced 18,420 results within its agreed window." The second statement is measurable. It also catches the quiet failure where no exception exists because no run began.

Don't make a probe your event store.

## Why process health misses a silent scheduled import

An e-commerce import can stop producing results while its Node.js process remains perfectly responsive. A scheduler expression may be absent from a deployment, a queue may stop delivering work, or a run may start without ever recording completion. In all three cases, `/live` can honestly answer 200. Restarting that responsive process may add churn without proving that the next import will finish.

Use three separate questions. `/live` asks, "Can this process execute an HTTP handler?" `/ready` asks, "Should this import-service instance accept new work?" `/health` gives a compact aggregate status for operators and an uptime monitor. For a service whose main responsibility is scheduled imports, stale result data can make readiness fail. For a storefront API that merely triggers an auxiliary import, it usually should not: taking checkout traffic away because a catalog feed is late couples unrelated failure domains.

The missing-run detector therefore lives outside the request itself. Each successful import writes `lastResultAt`, its bounded source identifier, and its cost center to durable storage. Probe handlers read the latest snapshot; they don't call every upstream supplier on each request. An independent evaluator compares the current time with the expected interval plus a grace period. This is a dead-man rule: if a 15-minute feed has a 5-minute grace period, an age above 20 minutes is stale. Those numbers are an example policy, not a universal best practice.

Walk one feed through the rule. A catalog import owned by `merchandising-eu` is due at 09:00 and normally commits its result at 09:04. At that moment the worker writes the completion timestamp and result count in the same operational flow as the committed output, then emits the completion log; the age gauge falls toward zero, the run counter increases by one, and the service stays ready. At 09:15 the next trigger should start, but imagine that it never arrives. Nothing throws. There is no failed promise to catch and no error log to search. The stored result simply gets older. The evaluator still allows the expected 15-minute interval and the configured 5-minute grace period, so a check at 09:23 does not page. Once the age crosses 20 minutes, `/ready` returns 503 for this import-focused service, the transition produces one warning log, and the external rule observes a stale age on repeated evaluations before notifying the cost center owner. If a late run commits at 09:27, the durable timestamp advances, readiness returns to 200, and the recovery transition is visible without manufacturing thousands of successful probe logs. This sequence is the useful before-and-after: the old design could only say that Express answered throughout the incident; the new design identifies missing business output, assigns it to a bounded owner, and recovers from evidence rather than from a process restart.

That distinction matters during deployments. A new process can be live before it is ready, restore the most recent durable snapshot, and then make the same freshness decision as the instance it replaced. A purely in-memory timestamp would reset on every restart and create false confidence or false pages, depending on its default value. Durable state makes the decision survive the exact event that liveness systems are designed to cause.

## What should a Node.js production health endpoint log when imports stop?

Log state transitions and completed business work, not every poll. A monitor requesting `/ready` every five seconds can generate 17,280 checks per day per instance; logging each successful request would hide the one event that matters. Metrics can represent the repeated observations. A warning log should appear when the freshness decision changes from ready to stale, with fields such as `feed`, `cost_center`, `last_result_at`, `age_seconds`, and `reason`.

Keep the public response narrower. Return 200 for a healthy decision and 503 for a degraded one, but don't expose supplier URLs, credentials, raw exceptions, order data, or internal topology. A body such as `{ "status": "degraded", "reason": "import_result_stale" }` is enough. RFC 5424 defines warning as a severity for warning conditions; it is a reasonable semantic match for a late feed that needs attention but has not established a wider outage.

For metrics, a counter records completed results and a gauge records the age of the latest result. OpenTelemetry's metrics concepts distinguish additive measurements from current-value measurements, which maps cleanly to those two questions. Keep labels bounded. `cost_center="merchandising-eu"` and `feed="catalog"` are workable when they come from a controlled list; `order_id`, a URL, or an exception message creates unbounded series and destroys useful cost attribution.

Cost attribution is then direct: aggregate completed records, runs, or processing duration by the stable cost center that owns the feed. It isn't a dollar estimate unless a documented pricing model converts those units. The operational alert and the finance report can share dimensions without pretending they answer the same question.

## A copyable Express and TypeScript example

This example keeps the decision path explicit. `loadImportState` and `saveImportResult` use an injected durable store, so the same handler works with a database or key-value implementation. There is no network call inside a probe. The metric serializer is deliberately small and uses fixed label values supplied by configuration.

```ts
import express, { Request, Response } from "express";

type ImportState = {
  feed: string;
  costCenter: string;
  lastResultAt: string | null;
  completedRuns: number;
  completedRecords: number;
};

interface ImportStateStore {
  load(): Promise<ImportState>;
  save(state: ImportState): Promise<void>;
}

type Config = {
  expectedIntervalSeconds: number;
  graceSeconds: number;
};

export function createImportApp(store: ImportStateStore, config: Config) {
  const app = express();
  let acceptingWork = true;
  let lastReady: boolean | undefined;

  async function evaluateReadiness() {
    const state = await store.load();
    const lastResultMs = state.lastResultAt
      ? Date.parse(state.lastResultAt)
      : Number.NaN;
    const ageSeconds = Number.isFinite(lastResultMs)
      ? Math.max(0, (Date.now() - lastResultMs) / 1000)
      : Number.POSITIVE_INFINITY;
    const maximumAge = config.expectedIntervalSeconds + config.graceSeconds;
    const ready = acceptingWork && ageSeconds <= maximumAge;
    const reason = !acceptingWork
      ? "draining"
      : ready
        ? "import_result_fresh"
        : "import_result_stale";

    if (lastReady !== undefined && lastReady !== ready) {
      console.warn(JSON.stringify({
        severity: "warning",
        event: "import_readiness_changed",
        ready,
        reason,
        feed: state.feed,
        cost_center: state.costCenter,
        last_result_at: state.lastResultAt,
        age_seconds: Number.isFinite(ageSeconds) ? Math.round(ageSeconds) : null,
      }));
    }
    lastReady = ready;

    return { ready, reason, ageSeconds, state };
  }

  app.get("/live", (_request: Request, response: Response) => {
    response.status(200).json({ status: "healthy" });
  });

  app.get("/ready", async (_request: Request, response: Response) => {
    const result = await evaluateReadiness();
    response.status(result.ready ? 200 : 503).json({
      status: result.ready ? "healthy" : "degraded",
      reason: result.reason,
    });
  });

  app.get("/health", async (_request: Request, response: Response) => {
    const result = await evaluateReadiness();
    response.status(result.ready ? 200 : 503).json({
      status: result.ready ? "healthy" : "degraded",
      checks: { import_results: result.reason },
    });
  });

  app.get("/metrics", async (_request: Request, response: Response) => {
    const { ageSeconds, state } = await evaluateReadiness();
    const labels = `feed="${state.feed}",cost_center="${state.costCenter}"`;
    const finiteAge = Number.isFinite(ageSeconds) ? ageSeconds : -1;
    const lines = [
      "# HELP import_completed_runs_total Completed scheduled import runs.",
      "# TYPE import_completed_runs_total counter",
      `import_completed_runs_total{${labels}} ${state.completedRuns}`,
      "# HELP import_completed_records_total Records produced by scheduled imports.",
      "# TYPE import_completed_records_total counter",
      `import_completed_records_total{${labels}} ${state.completedRecords}`,
      "# HELP import_last_result_age_seconds Age of the latest import result, or -1 before one exists.",
      "# TYPE import_last_result_age_seconds gauge",
      `import_last_result_age_seconds{${labels}} ${finiteAge}`,
    ];
    response.type("text/plain; version=0.0.4").send(`${lines.join("\n")}\n`);
  });

  async function recordImportResult(records: number): Promise<void> {
    const state = await store.load();
    const next = {
      ...state,
      lastResultAt: new Date().toISOString(),
      completedRuns: state.completedRuns + 1,
      completedRecords: state.completedRecords + records,
    };
    await store.save(next);
    console.info(JSON.stringify({
      severity: "informational",
      event: "import_completed",
      feed: next.feed,
      cost_center: next.costCenter,
      records,
      completed_at: next.lastResultAt,
    }));
  }

  function beginShutdown(): void {
    acceptingWork = false;
  }

  return { app, recordImportResult, beginShutdown };
}
```

Call `recordImportResult(records)` only after the import has committed the result that downstream systems can consume. Calling it when a run starts would recreate the original blind spot. On SIGTERM, call `beginShutdown()` before closing the HTTP server so readiness can turn degraded while liveness remains responsive during the drain.

One caveat is visible in the serializer: label values must be escaped before use if configuration is not already restricted to a safe controlled vocabulary. In a real service, validate `feed` and `costCenter` at configuration load time. I'm not sure what cardinality ceiling fits your telemetry backend; retention, aggregation, and billing rules resolve that. Start with the smallest stable set and measure series growth.

## How should the alert and deployment behave?

The metric does not page anyone by itself. Run the evaluator in a different failure domain from the importer, query or scrape the age gauge, and require repeated stale observations before notifying the on-call engineer. An external request to `/health` adds another view: it can detect DNS, TLS, routing, or process reachability problems that an internal completion metric cannot establish. The two checks overlap on purpose, but they answer different questions.

Test the boundary with a clock you can control. At exactly `expectedIntervalSeconds + graceSeconds`, define whether the state remains ready; the example uses `<=`, so it does. One second later it returns 503. Also test a missing timestamp, a restored timestamp after restart, a completed run that reports zero valid records, shutdown ordering, and two concurrent completions. The last case requires the durable store to update counters atomically; the interface above does not promise that behavior on its own.

The catch is that freshness-based readiness is not suitable when removing the instance cannot improve import execution. If every replica reads the same delayed supplier feed, marking all replicas unready only removes the control surface. Keep `/ready` tied to local admission in that architecture and put staleness solely in the external alert. Likewise, stick with a dedicated heartbeat from the scheduler when the scheduler runs outside the Node.js service, because the service cannot report a trigger it never owned.

Deploy the policy in observation mode first: graph age, inspect completion logs, and compare the proposed threshold with actual schedules. Holidays, supplier time zones, daylight-saving changes, and intentionally paused feeds can all look like silence. Encode those calendars in the evaluator rather than stretching one global grace period until it stops being useful. Then enable paging, document the owner by cost center, and give the alert a concrete first action: check the latest completion event and the scheduler's durable run record.

Small rule. Big payoff.

## References

- RFC 5424, *The Syslog Protocol*: https://datatracker.ietf.org/doc/html/rfc5424

## Further reading

- OpenTelemetry, *Metrics signal concepts*: https://opentelemetry.io/docs/concepts/signals/metrics/
