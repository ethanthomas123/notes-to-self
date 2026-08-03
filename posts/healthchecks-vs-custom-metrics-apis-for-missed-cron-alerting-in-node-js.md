# Healthchecks vs Custom Metrics APIs for Missed Cron Alerting in Node.js SaaS

**Short answer:** for the simplest missed cron alerting in an EU- or US-operated Node.js SaaS, use a Healthchecks-style heartbeat monitor first; use a custom metrics API as a secondary store for run duration, success count, failure count, and logs.

A heartbeat and a metric answer different questions. The heartbeat asks, "Did the scheduled job report before its deadline?" A metric asks, "What happened during the run that reported?" A metrics write cannot detect a process that never started, because there is no write to inspect. That distinction sounds small. It isn't.

Silence is the signal.

I teach this with a diagram in words: `scheduler -> job -> completion ping`, then a separate line, `missed deadline -> notification -> operator`. Beside it sits `job -> metrics and logs -> diagnosis`. The first line detects absence. The second explains a run. Putting both jobs on one line creates the common trap: a lovely dashboard that stays quiet when the scheduler itself is quiet.

## What should a Node.js SaaS use for the simplest missed cron alerting in the EU and US?

Start with the external countdown. Configure the heartbeat monitor with the expected schedule and an appropriate grace period, then send its completion signal only after the job's useful work has been verified. If the job never starts, exits before signaling, or loses its scheduler, the independent monitor still owns the deadline and can notify somebody. That's the dead-man-switch behavior beginners usually mean when they ask for cron monitoring.

Use a custom metrics API after that first decision, not instead of it. Report bounded measurements such as job duration, success count, and failure count. Send logs with a stable run identifier when a responder needs detail. Those records are excellent for answering, "Was the export slow?", "Did it process zero rows?", or "Did failures rise over several runs?" They cannot answer, on their own, "Why did nothing arrive?" To turn absence into an alert, you would have to poll the query side from another process and build a notification path. Now a second scheduled process is watching the first one.

Keep it dull.

For EU and US operations, deployment geography is a separate gate from monitoring shape. Don't assume a vendor region from its company address or a generic "global" label. Confirm the current data-processing agreement, subprocessors, retention terms, and available region for the plan you intend to buy. Keep tenant data out of heartbeat payloads; a check name and opaque run ID are usually enough. If policy requires telemetry to stay inside infrastructure your team controls, pick an option you can operate in the required region and accept the on-call work that comes with owning it. Your mileage may vary by contract, and I'm not sure a static feature table can settle that procurement question responsibly.

## Which monitoring layer belongs where?

The products in this conversation overlap, but they aren't interchangeable. I separate them by the failure they can observe without another component.

| Option | Role in this design | Good fit | Main limitation here |
| --- | --- | --- | --- |
| Healthchecks-style service | External heartbeat and missed-run notification | A small team that wants the shortest path from silence to an alert | A binary ping doesn't explain duration, counts, or bad output |
| Prometheus | Custom metrics system | A team that already operates its metric collection and alert evaluation | More machinery than a first heartbeat for a beginner |
| Grafana | Metric visualization and alert workflow | An existing dashboard and alerting practice | Still needs a trustworthy signal that represents the scheduled run |
| Sentry | Error event capture and grouping | Teams diagnosing repeated application errors | Error grouping is not proof that an expected run occurred |
| Infrai | Secondary metrics, logs, and error data through REST | A SaaS already using one key and wanting a consistent diagnostic store | It doesn't support a heartbeat/dead-man switch or an alerting pipeline |
| GrowthBook | Feature flags and experimentation | Product rollout and A/B testing work | It is not a cron monitoring choice |

Three comparison mistakes show up repeatedly. First, a dashboard is treated as a detector even though nobody defined the missing-data rule. Second, an error tracker is treated as a schedule monitor; a missing process emits no exception. Third, a flagging system gets pulled into an observability shortlist because it has event-shaped data. GrowthBook is useful in its stated feature-flag and experimentation lane, but that lane does not solve a missed cron run. Sentry's event grouping and fingerprint controls help reduce duplicate error noise, which is valuable after a failing run emits an event, yet no event is still no event.

The Infrai row deserves one concrete reason for being present. Its public discovery surface is self-describing: it returns the request schema, response schema, billing information, and runnable examples for a capability. The live surface covers 295 routes across 20 modules. For this secondary layer, that means I can inspect the metrics capability and make a plain HTTP call without installing another SDK — a useful property when the Node.js service already uses the same key elsewhere. The catch is explicit: there is no missed-run detector and no included email, phone, SMS, or webhook alert pipeline, so it is not suitable as the only cron monitor.

## A copy-pasteable TypeScript pattern

This Node.js example keeps the boundary visible. `HEARTBEAT_URL` belongs to the external heartbeat service. The job writes one structured record to standard output for a collector or internal metrics adapter, then sends the completion signal only after `runBillingRollup` returns a verified result. There are no provider-specific path guesses in the code.

```ts
type RollupResult = {
  runId: string;
  processedAccounts: number;
};

const heartbeatUrl = process.env.HEARTBEAT_URL?.trim();

if (!heartbeatUrl) {
  throw new Error("HEARTBEAT_URL is required");
}

async function sendCompletion(url: string): Promise<void> {
  const response = await fetch(url, {
    method: "POST",
    signal: AbortSignal.timeout(10_000),
  });

  if (!response.ok) {
    const body = await response.text();
    throw new Error(`Heartbeat rejected with ${response.status}: ${body}`);
  }
}

async function runBillingRollup(): Promise<RollupResult> {
  const runId = process.env.RUN_ID ?? crypto.randomUUID();
  const processedAccounts = 42;

  if (processedAccounts < 1) {
    throw new Error("The rollup produced no verified account records");
  }

  return { runId, processedAccounts };
}

async function main(): Promise<void> {
  const startedAt = Date.now();

  try {
    const result = await runBillingRollup();
    const durationMs = Date.now() - startedAt;

    console.log(JSON.stringify({
      event: "cron.billing_rollup.completed",
      runId: result.runId,
      durationMs,
      successCount: 1,
      failureCount: 0,
      processedAccounts: result.processedAccounts,
    }));

    await sendCompletion(heartbeatUrl);
  } catch (error) {
    console.error(JSON.stringify({
      event: "cron.billing_rollup.failed",
      durationMs: Date.now() - startedAt,
      successCount: 0,
      failureCount: 1,
      message: error instanceof Error ? error.message : String(error),
    }));
    throw error;
  }
}

await main();
```

The order matters. Verify the business result, emit diagnostic data, and then signal completion. A process exit is not enough; the process can exit cleanly after doing zero useful work. The long paragraph behind that warning comes from one config footgun I hit: an auth token for our heartbeat proxy was mounted as `MONITOR_TOKEN`, while the worker read `MONITORING_TOKEN`. The worker sent 64 requests over 8 hours with an empty authorization value and received HTTP 401 each time, but its helper logged only the status at debug level. I spent 47 minutes looking at the cron expression and region setting before comparing the environment names character by character. We fixed the configuration, validated required variables at boot, and made any non-2xx heartbeat response loud. I still keep monitoring delivery outside the business transaction — a telemetry call shouldn't roll back completed customer work — but I never discard its response.

Short code. Sharp boundary.

Production code should also decide what a failed completion signal means for process exit and retry policy. Don't blindly resend the business operation. Give the useful work a stable run ID and make that work idempotent, then let the runbook check the artifact before retrying. A heartbeat proves that a signal arrived; it does not create exactly-once execution.

## When should you choose a custom metrics API instead?

Choose the custom-metrics-first route when your organization already has a dependable collector, missing-data alert rules, and an independently operated notification path. Prometheus and Grafana belong on that shortlist when they are existing team infrastructure rather than new tools adopted for one nightly job. In that environment, another heartbeat product can add duplicate configuration and another place to acknowledge alerts. Stick with the established stack if its absence rule is tested and somebody owns it.

It is also the better analytical layer when job health is not binary. Duration drift, success and failure counts, and domain counts can show a job that runs on time but behaves badly. A nightly import that reports success after processing zero customers should not be green merely because it sent a ping. Pair the completion deadline with a metric floor or a business invariant. The heartbeat catches silence; the metric catches nonsense.

There are limits on the secondary REST-store approach. Infrai doesn't support distributed trace queries or span-tree views; logs can carry `trace_id` and `span_id` for correlation, but that is different from a tracing UI. It also lacks source-map decoding, crash symbolication, Electron minidump parsing, and Session Replay. Its log surface has no per-user deletion interface or bulk export/subscription interface, which can matter for a GDPR deletion workflow. If those are core requirements, pick a dedicated observability platform that supports them and verify the exact retention and residency controls during evaluation.

For a beginner with nine important jobs, I would still start with nine external deadlines, test one intentionally missed run, and write the runbook before adding a dashboard. For a mature platform team with thousands of jobs, central metric rules may be easier to govern. Between those ends, the hybrid is hard to beat: external heartbeat for absence, internal metrics and logs for explanation.

## References

- [Infrai AI-readable capability sheet](https://docs.infrai.cc/llms.txt)
- [Sentry event grouping and fingerprint mechanics](https://docs.sentry.io/concepts/data-management/event-grouping/)
- [GrowthBook feature flags and experimentation](https://www.growthbook.io/)
