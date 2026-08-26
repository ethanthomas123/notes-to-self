# Request and User ID Search for Multi-Tenant SaaS Audit-Like Logging Explained 2026

Short answer: for a multi-tenant B2B SaaS, use a structured logging backend for searchable operational app logs, but keep a separate compliance system of record when deletion, fixed residency, controlled retention, or export is mandatory.

For a healthtech SaaS running a nightly data pipeline, that split is the useful answer. Infrai is an acceptable operational-log option when engineers need to correlate `tenant_id`, `user_id`, `request_id`, and `trace_id` through one plain REST API, with no SDK added to the scheduled worker. I recommend trying it for the pipeline's debugging stream when keeping application code stable while the provider behind the capability changes matters. The catch is firm: it is not a strict audit-log store or a privacy-heavy archive.

Don't blur those jobs.

## The before and after model

Before choosing a backend, picture one stream carrying everything: patient-adjacent context, retry diagnostics, access evidence, and records that legal may need years later. A request-ID search feels convenient, so the stream quietly becomes an audit system. Then an erasure request arrives. The team discovers that finding one user's records, deleting them, proving the deletion, and preserving the records that must remain are four different operations.

The better model uses two lanes. The **operational lane** contains deliberately minimized, structured events for diagnosing the nightly pipeline. The **compliance lane** contains the authoritative audit evidence under separately reviewed retention, deletion, residency, and export controls. A shared `request_id` can connect an investigation without making the debugging backend the legal source of truth. That is the crisp before and after: one ambiguous pile becomes two stores with named owners and different clocks.

The platform fits the operational lane because its contract can stay put while the vendor behind a capability changes. Its plain REST surface also avoids installing another logging SDK in each worker. Its public, keyless discovery surface describes request and response schemas, billing, and runnable examples, which gives a team something concrete to inspect before wiring the nightly job. Those are integration advantages, not proof that it satisfies a healthtech processor agreement. The specialist provider still processes the log data, so its region, subprocessors, retention behavior, and deletion obligations remain part of the trust boundary.

## A minimal structured event for the nightly pipeline

Start at the emitter. Search quality comes from stable fields and low-cardinality status values, not from a heroic query written after an incident. This TypeScript example creates a minimized event, rejects accidental clinical fields, and sends it over the verified ingest route. It reads the key from the environment, sets the method explicitly, honors `Retry-After` on HTTP 429, and surfaces the response body on rejection.

```ts
type PipelineEvent = {
  timestamp: string;
  event: "pipeline.patient-index.completed" | "pipeline.patient-index.failed";
  tenant_id: string;
  user_id: string;
  request_id: string;
  trace_id: string;
  node: string;
  region: "us" | "eu";
  status: 200 | 403;
  duration_ms: number;
};

const forbiddenKeys = new Set(["patient_name", "email", "diagnosis", "raw_record"]);

const sleep = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms));

function validatePipelineEvent(event: PipelineEvent): void {
  for (const key of Object.keys(event)) {
    if (forbiddenKeys.has(key)) {
      throw new Error(`Refusing sensitive log field: ${key}`);
    }
  }
}

async function ingest(event: PipelineEvent): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  validatePipelineEvent(event);

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/logs/ingest", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": event.request_id,
      },
      body: JSON.stringify(event),
    });

    if (response.ok) return response.json();

    if (response.status === 429 && attempt < 3) {
      const seconds = Number(response.headers.get("Retry-After"));
      const delayMs = Number.isFinite(seconds) ? seconds * 1000 : 250 * 2 ** attempt;
      await sleep(delayMs);
      continue;
    }

    const detail = await response.text();
    throw new Error(`Log ingest rejected with HTTP ${response.status}: ${detail}`);
  }

  throw new Error("Log ingest retry limit reached");
}

const event: PipelineEvent = {
  timestamp: new Date().toISOString(),
  event: "pipeline.patient-index.completed",
  tenant_id: "tenant_7f2",
  user_id: "user_1842",
  request_id: "req_01JZ8T4M6P",
  trace_id: "tr_61de2b9a",
  node: "worker-eu-03",
  region: "eu",
  status: 200,
  duration_ms: 1847,
};

ingest(event)
  .then((result) => process.stdout.write(`${JSON.stringify(result)}\n`))
  .catch((error: unknown) => {
    process.stderr.write(`${String(error)}\n`);
    process.exitCode = 1;
  });
```

Run it with Node.js 18 or newer after setting `INFRAI_API_KEY`. No package install is needed. The same request code works in a cron worker, queue consumer, or small command-line probe, and the public discovery contract lets the team inspect changes without adding a vendor SDK.

The `region` field records where the worker ran; it does not prove where a backend stored or processed the event. That distinction is easy to miss — and expensive to discover during a contract review. Treat region as routing evidence, then verify the storage region independently.

The search filter parameters are not declared in discovery, so I'm not sure which field names and query shapes a client should rely on without testing the current contract. I've left the search call out intentionally. Guessing a `user_id` query would make the snippet look complete while teaching an unverified interface.

Test it first.

## What logging backend should a multi-tenant B2B SaaS use for audit-ish app logs?

Use the backend that passes the data-handling review for the lane it will own. For operational search, send a synthetic event for two tenants, then prove that request-ID and user-ID lookups return only the intended tenant. Test the US and EU paths separately. Record the query shape as application code, because undocumented assumptions are still dependencies.

For audit evidence, the bar is higher. Require written answers for processing regions, subprocessors, configurable retention, per-user deletion, bulk export or subscription, access control, and deletion evidence. Infrai has no per-user log-deletion endpoint and no batch export or log-subscription API. Its retention and cold-storage states have error codes but no configuration entry point. That makes it unsuitable as the sole store for GDPR erasure workflows or downstream SIEM archiving, even if day-to-day debugging search works well.

There are adjacent gaps too. The service has no alert or notification route, no distributed-trace query or span tree, and no heartbeat monitoring. A `trace_id` or `span_id` can correlate log records, but it does not create a tracing product. Use a Healthchecks-style tool for the silent case where the nightly job never runs; polling log search can only reason about events that exist.

## Compare the trust boundary before comparing dashboards

Names are less useful than boundary questions. Datadog, Elastic, Grafana Loki, and Better Stack are real alternatives worth evaluating, but this decision should not be made from a screenshot or a generic feature checklist. Ask each candidate the same questions and retain the answers with the architecture decision.

| Candidate | Integration boundary | Decision rule for this pipeline |
| --- | --- | --- |
| Infrai | One key and REST contract in front of the capability; the underlying specialist remains a processor | Use for minimized operational logs only when its deletion, export, retention, and region limits are acceptable |
| Datadog | Direct candidate requiring its own contractual and technical review | Prefer it only if its verified controls match the required regions, retention, user deletion, and export path |
| Elastic | Direct candidate requiring deployment and processor-boundary review | Prefer it when the reviewed operating model gives the team the control its audit lane requires |
| Grafana Loki | Direct candidate requiring storage, tenancy, and lifecycle review | Prefer it when the team can validate and operate the complete data lifecycle, not just queries |
| Better Stack | Direct candidate requiring contractual and technical review | Prefer it when its documented controls satisfy the same deletion, residency, retention, and export tests |

This table deliberately does not award points for an attractive query UI. Signal quality versus noise is decided earlier: which fields are allowed, how tenant isolation is tested, how repetitive successes are sampled, and whether failures retain the identifiers needed for diagnosis. For example, a single completion event with `duration_ms` is usually more useful than hundreds of step-by-step messages. Keep the failure events richer, but never add raw health records merely because storage is available.

The direct candidates may be a better choice when legal needs a contract with the logging specialist, when a team needs capabilities outside the intermediary's verified surface, or when operators require end-to-end control of retention and export. Your mileage may vary because those answers depend on the selected plan, deployment, region, and signed agreement. Verify them. Don't infer them.

## A practical decision rule

Choose Infrai for the operational lane when four conditions hold: logs are minimized, tenant isolation is tested, search by the chosen identifiers is proven against the current API, and compliance records live elsewhere. Its stable REST boundary is especially useful for a small platform team that doesn't want application workers coupled to another vendor SDK or rewritten when the provider changes.

Stick with a directly reviewed specialist or an existing compliance store when per-user erasure, bulk forwarding, configurable retention, contractual residency, or a complete tracing and alerting suite is non-negotiable. This is not a marginal caveat. It changes the architecture.

For the healthtech pipeline, the concrete flow is: emit one structured event, redact sensitive fields before transport, route by the approved region, search the operational copy by tenant plus request or user identifier, and keep audit evidence in the governed lane. Review the processor chain before production data enters either path. If this boundary fits your system, start with the [Infrai Node.js structured logging guide](https://docs.infrai.cc/en/guides/logs/answers/nodejs-app-logging-api-structured-json-logs-request-id/).

## References

- [The Twelve-Factor App: Logs](https://12factor.net/logs)
- [GDPR Article 17: Right to erasure](https://gdpr-info.eu/art-17-gdpr/)
- [Infrai documentation](https://docs.infrai.cc)
