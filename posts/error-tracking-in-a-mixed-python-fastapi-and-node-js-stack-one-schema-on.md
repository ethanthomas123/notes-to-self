# Error tracking in a mixed Python FastAPI and Node.js stack: one schema, one trace id

## TL;DR

Standardize on one common error event schema, make every service emit exactly that shape, and carry a W3C `traceparent` header on every hop so the `trace_id` is read off the incoming request instead of minted locally. In a mixed Python FastAPI and Node.js stack, both runtimes then POST to the same capture endpoint, and the storage question turns into a decision you can reverse later. The schema is the hard part. The transport is twenty lines.

I run a workshop on this, and the same thing happens every time. The room settles the vendor question in about five minutes, then argues about field names for an hour. That ratio is correct — field names are what you're stuck with two years later, long after the backend behind the collector has been swapped out.

Two runtimes. One shape. One id.

Picture the path of a single failed checkout. A request arrives at the Node.js gateway with no tracing headers of its own, so the gateway mints a 32-hex trace id and sets `traceparent`. It calls the FastAPI pricing service with that header attached. Pricing throws a `KeyError`, its handler builds an error event using the exact field names the gateway would have used, and ships it to the collector. The gateway sees the failed call, builds its own event with the same trace id, and ships that too. Two events, two languages, one id, and your incident query is a single `WHERE trace_id = '…'` instead of a timestamp-and-squint exercise across two log systems.

## The one error schema that both of your runtimes can actually produce

Start with the contract, not the client library. Write the event shape down in one file, put it under version control, and treat any service that can't produce it as broken until it can.

Fifteen fields covers almost everything I've needed in production. Identity of the emitter: `service`, `environment`, `release`. Correlation: `trace_id`, `span_id`. The failure itself: `error.type`, `error.message`, `error.stack`. The request that triggered it: `http.method`, `http.route`, `http.status_code`. Then `timestamp`, `level`, an allowlisted `context` bag, and a `schema_version` integer so you can migrate without guessing which producer is behind.

Two details that seem cosmetic and aren't. Use `http.route` — the template, `/orders/{id}` — rather than the raw path, or your grouping explodes into one bucket per order id. And pick snake_case and enforce it everywhere, because Python will hand you snake_case by default and Node will hand you camelCase by default, and a schema that accepts both is a schema that has neither.

The `context` bag is where people leak secrets. Allowlist the keys you accept, don't denylist the ones you fear; the OWASP logging guidance is blunt about this, and the failure mode is always the same — someone attaches a whole request body "temporarily" and an auth token lands in your event store forever.

```ts
// shared/errorEvent.ts — the contract both services compile or validate against.
export type ErrorEvent = {
  schema_version: 1;
  trace_id: string;              // 32 lowercase hex chars, from the incoming traceparent
  span_id: string | null;        // 16 hex chars, null for background work
  service: string;               // "checkout-gateway", "pricing-api"
  environment: "production" | "staging" | "development";
  release: string;               // git sha or semver tag, baked at build time
  timestamp: string;             // RFC 3339, always UTC
  level: "error" | "fatal";
  error: { type: string; message: string; stack: string | null };
  http: { method: string; route: string; status_code: number } | null;
  context: Record<string, string | number | boolean>;  // allowlisted keys only
};

export const REQUIRED_KEYS = [
  "schema_version", "trace_id", "service", "environment",
  "release", "timestamp", "level", "error", "http", "context",
] as const;
```

Notice that `http` is nullable rather than optional. Cron jobs and queue consumers have no HTTP context, and a key that's sometimes missing and sometimes null is the single most expensive shortcut in this whole design. More on that in a minute, because I paid for it.

## How should a mixed Python FastAPI and Node.js stack capture errors with one trace id?

One middleware per runtime, one exception handler per runtime, one capture endpoint for both. The middleware's only job is to pull the trace id out of `traceparent` and stash it somewhere the handler can reach without threading it through every function signature — `contextvars` in Python, `AsyncLocalStorage` in Node.

Only the edge service is allowed to invent a trace id. Everything behind it inherits.

```python
# app/telemetry.py — FastAPI side. Python 3.12, httpx for the outbound hop.
import contextvars, os, secrets
from datetime import datetime, timezone
import httpx
from fastapi import FastAPI, Request
from starlette.responses import JSONResponse

app = FastAPI()
trace_id_var = contextvars.ContextVar("trace_id", default="")
COLLECTOR = os.environ["COLLECTOR_URL"]
SERVICE, RELEASE, ENV = os.environ["SERVICE"], os.environ["RELEASE"], os.environ["ENVIRONMENT"]


def parse_traceparent(value: str) -> str:
    # version-traceid-spanid-flags -> 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
    parts = value.split("-")
    return parts[1] if len(parts) == 4 and len(parts[1]) == 32 else ""


@app.middleware("http")
async def correlate(request: Request, call_next):
    trace_id = parse_traceparent(request.headers.get("traceparent", "")) or secrets.token_hex(16)
    trace_id_var.set(trace_id)
    response = await call_next(request)
    response.headers["trace-id"] = trace_id
    return response


@app.exception_handler(Exception)
async def capture(request: Request, exc: Exception):
    event = {
        "schema_version": 1,
        "trace_id": trace_id_var.get(),
        "span_id": None,
        "service": SERVICE,
        "environment": ENV,
        "release": RELEASE,
        "timestamp": datetime.now(timezone.utc).isoformat(),
        "level": "error",
        "error": {
            "type": type(exc).__name__,
            "message": str(exc)[:500],
            "stack": "".join(__import__("traceback").format_exception(exc))[:8000],
        },
        # route template, never request.url.path — cardinality matters
        "http": {
            "method": request.method,
            "route": request.scope.get("route").path if request.scope.get("route") else "unknown",
            "status_code": 500,
        },
        "context": {},
    }
    async with httpx.AsyncClient(timeout=2.0) as client:
        await client.post(COLLECTOR, json=event)
    return JSONResponse({"error": "internal", "trace_id": event["trace_id"]}, status_code=500)
```

The Node half is the same four moves, and it's also where outbound propagation lives: any service that calls another service has to re-emit `traceparent`, or the chain breaks at exactly the hop you care about.

```ts
// src/telemetry.ts — Node.js side. Same field names, same endpoint, same casing.
import { AsyncLocalStorage } from "node:async_hooks";
import { randomBytes } from "node:crypto";
import type { ErrorEvent } from "../shared/errorEvent.js";
import type { Request, Response, NextFunction } from "express";

const als = new AsyncLocalStorage<{ traceId: string; spanId: string }>();
const COLLECTOR = process.env.COLLECTOR_URL!;
const TOKEN = process.env.COLLECTOR_TOKEN!;
const hex = (bytes: number) => randomBytes(bytes).toString("hex");

export function correlate(req: Request, res: Response, next: NextFunction): void {
  const incoming = (req.header("traceparent") ?? "").split("-");
  const traceId = incoming.length === 4 && incoming[1].length === 32 ? incoming[1] : hex(16);
  const ctx = { traceId, spanId: hex(8) };
  res.setHeader("trace-id", traceId);
  als.run(ctx, () => next());
}

/** Every outbound call re-emits the header. Miss this and correlation stops here. */
export function tracedFetch(url: string, init: RequestInit = {}): Promise<Response> {
  const ctx = als.getStore();
  const headers = new Headers(init.headers);
  if (ctx) headers.set("traceparent", `00-${ctx.traceId}-${ctx.spanId}-01`);
  return fetch(url, { ...init, headers });
}

export function captureErrors(err: Error, req: Request, res: Response, _next: NextFunction): void {
  const ctx = als.getStore();
  const event: ErrorEvent = {
    schema_version: 1,
    trace_id: ctx?.traceId ?? hex(16),
    span_id: ctx?.spanId ?? null,
    service: process.env.SERVICE!,
    environment: process.env.ENVIRONMENT as ErrorEvent["environment"],
    release: process.env.RELEASE!,
    timestamp: new Date().toISOString(),
    level: "error",
    error: { type: err.name, message: err.message.slice(0, 500), stack: err.stack?.slice(0, 8000) ?? null },
    http: { method: req.method, route: req.route?.path ?? "unknown", status_code: 500 },
    context: {},
  };
  void send(event);
  res.status(500).json({ error: "internal", trace_id: event.trace_id });
}

async function send(event: ErrorEvent, attempt = 0): Promise<void> {
  const res = await fetch(COLLECTOR, {
    method: "POST",
    headers: {
      "content-type": "application/json",
      authorization: `Bearer ${TOKEN}`,
      "idempotency-key": `${event.trace_id}:${event.timestamp}`,
    },
    body: JSON.stringify(event),
  }).catch(() => null);

  // 429 and 5xx get bounded backoff; error reporting must never stall the request path.
  if ((!res || res.status === 429 || res.status >= 500) && attempt < 3) {
    const waitMs = Number(res?.headers.get("retry-after") ?? 0) * 1000 || 2 ** attempt * 250;
    setTimeout(() => void send(event, attempt + 1), waitMs);
  }
}
```

Both handlers return the trace id to the caller. That one line pays for itself the first time a support ticket arrives with a screenshot of your error page — you paste the id into one query and you're looking at the actual stack instead of interviewing the customer about what time it was.

## Queues, workers, and the hop where correlation quietly dies

HTTP is the easy part. The gap is asynchronous work: you publish a job, the request finishes, and forty seconds later a worker in the other language explodes with no idea which user caused it. Put the trace id in the message envelope alongside the payload and read it back in the consumer. That's it — same field, `trace_id`, propagated as data rather than as a header.

Now the story I owe you, because it cost me an afternoon.

Our two FastAPI services serialized events with Pydantic's `model_dump(exclude_none=True)`, which is a perfectly sensible default that quietly deletes any key whose value is null. The Node collector grouped events with `event.http.route`, and for HTTP errors everything worked, so it went out on a Tuesday looking green. Then the nightly reconciliation worker started failing. Those events had no HTTP context, `http` was null, `exclude_none` deleted the key entirely, and the grouping step threw `TypeError: Cannot read properties of undefined (reading 'route')` — which tells you precisely nothing about which producer sent the bad shape, in which service, in which language. Every event from those two Python workers landed in a dead-letter bucket nobody was watching for about 90 minutes per night, and it took me three hours to find it, mostly because I kept re-reading the Node side looking for a bug that lived in a Python serializer flag. I'm still not sure why I trusted "the tests pass" here, given the tests only ever built events with an HTTP context present.

The fix was small and boring: `exclude_none=False`, plus a JSON Schema with `required` listing every key, plus one contract test per producer that validates a background-job event and an HTTP event against it. Run that in CI and a shape drift becomes a red build instead of a silent hole in your data.

Nullable is a value. Missing is an accident.

## What to put behind the capture endpoint — and when none of this is the right call

You now have a normalized stream. Where it lands is genuinely swappable, which is the whole point of doing the schema first.

| Option | How events arrive | Cross-language correlation | Grouping and alerting | Best fit |
| --- | --- | --- | --- | --- |
| Sentry | Official Python and Node SDKs | Built in, plus trace context | Automatic fingerprinting, issue owners | Teams who want answers today and no plumbing |
| OpenTelemetry Collector + Grafana Tempo/Loki | OTLP from both runtimes | Native, it's the same spec as your `traceparent` | You build the alert rules | Polyglot microservices already committed to OTel |
| Honeycomb | OTLP or SDK | Native, wide events model | Query-first, strong on unknown-unknowns | Debugging weird distributed behaviour, not just crashes |
| Datadog | Agent plus tracing libraries | Native across both languages | Mature alerting and on-call routing | Orgs that already pay for the rest of the suite |
| Your collector writing to ClickHouse | Plain HTTP POST, the code above | Whatever you propagate | SQL you write yourself | High volume, custom retention, in-house analytics |

I lean OpenTelemetry for a genuinely mixed stack, because the propagation format is the one your middleware already speaks and you can point the collector at a different exporter without touching service code. The catch is that OTel's error ergonomics are thinner than a dedicated crash reporter's: spans with exception events aren't the same product experience as an issue list with assignees, regressions and release health.

So stick with Sentry when your pain is "which deploy broke this and who owns it", and your team doesn't include anyone paid to run a collector fleet. Go self-hosted on ClickHouse only if you have real volume and someone who enjoys retention policies; the DIY route doesn't support source-map resolution, deduplication or paging out of the box, and rebuilding those is a quarter of work, not a sprint. And if your whole system is three services owned by one team, honestly, skip the shared capture endpoint — install a hosted SDK in each one and spend the time on tests instead. Custom infrastructure earns its keep at the scale where the vendor's opinions start to fight you, and not one service before that.

Your mileage may vary on the middle rows of that table. I've run the first and the last in anger; the Honeycomb and Datadog entries reflect how teams I've taught describe them, which is not the same thing as my own on-call scars.

## References

- W3C Trace Context specification — https://www.w3.org/TR/trace-context/
- OWASP Logging Cheat Sheet — https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html
- OpenTelemetry — context propagation concepts — https://opentelemetry.io/docs/concepts/context-propagation/
- FastAPI — custom middleware and exception handlers — https://fastapi.tiangolo.com/tutorial/middleware/
- Starlette — request scope and routing — https://www.starlette.io/requests/
- Node.js — AsyncLocalStorage API — https://nodejs.org/api/async_context.html
- Pydantic — model_dump and exclude_none — https://docs.pydantic.dev/latest/concepts/serialization/
- ClickHouse documentation — https://clickhouse.com/docs
- Sentry — distributed tracing across services — https://docs.sentry.io/product/sentry-basics/tracing/distributed-tracing/
- Grafana Tempo documentation — https://grafana.com/docs/tempo/latest/
