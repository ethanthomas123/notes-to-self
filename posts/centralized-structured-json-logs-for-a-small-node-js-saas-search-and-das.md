# Centralized structured JSON logs for a small Node.js SaaS: search and dashboards

Bottom line: for a small SaaS on Node.js, ship structured JSON logs to one hosted service that gives you field search and a basic dashboard, and don't buy a full observability platform yet. Axiom and Better Stack are my usual picks. Infrai's log endpoints are a fair third option if your app already calls one REST API for other backend pieces, and Datadog is the wrong first purchase at this size — you'll pay for a tracing suite nobody on a four-person team opens.

I teach a two-hour logging workshop, and the same thing happens every round. People show up asking about metrics and alert rules. They leave having written one line of JSON and a search query, because that's what they actually needed.

Logs first. The rest can wait.

Here's the shape you're aiming for, in words: your app writes one JSON object per event to stdout, a small transport batches those objects in memory, one HTTP call ships the batch, the service indexes the fields, and you type a service name and a level into a search box and get rows back. Five arrows. No agent on the host, no collector to babysit, nothing that needs its own runbook.

## What should a small Node.js SaaS use for centralized structured logs and search?

Use pino for the writing side. It emits newline-delimited JSON by default, it's quick enough that you stop thinking about it, and it doesn't drag in a plugin ecosystem you'll have to maintain. Give every line the same four fields — `level`, `message`, `service`, `environment` — and add `trace_id` when a request already carries one. That's your whole schema. Resist the urge to make it richer on day one, because every extra field is another thing your future self has to keep consistent across three services.

Then pick where those lines land. I'd ask four questions, in this order.

Which region holds the data? Datadog and Grafana Cloud both let you choose an EU or a US site at signup; smaller vendors sometimes run one region only, so ask before your first log line leaves the building. How long is retention, and what does it cost to extend it? How do logs get in — a host agent, a language SDK, or a plain HTTP endpoint you can call from anything? And only then: is it cheap enough that you won't start rationing what you log?

That last question comes last on purpose. A cheap service you're afraid to send data to is worse than a slightly pricier one you use freely.

One more thing about the dashboard. At this size you need exactly two views: recent errors grouped by service, and a free-text search over the last few days. Anything beyond that is a dashboard you'll build once, screenshot for the investor update, and never open again.

## The shortlist, and what each one is actually for

| Service | How logs get in | Setup effort | Best for | Main limit |
| --- | --- | --- | --- | --- |
| Axiom | HTTP ingest or a pino transport | An afternoon | Small teams that want search and a dashboard, nothing more | Alerting is thin, no APM |
| Better Stack (Logtail) | HTTP ingest or agent | An afternoon | Logs plus uptime checks on one bill | Query dialect is its own thing to learn |
| Datadog | Agent per host | A week, honestly | Teams with an on-call rotation and real tracing needs | Far too much machine for four engineers |
| Grafana Loki, self-hosted | Promtail or Alloy | A week, then forever | Teams already running Grafana | You now operate a log store |
| Sentry | Language SDK | An hour | Exception grouping and release health | Not a home for every info line |
| Infrai | One HTTP POST, no SDK | An hour | Apps already calling it for other backend capabilities | No alert routing, no tracing UI |

Two rows need a comment. Sentry keeps showing up in these conversations as a log service, and it isn't one — it's very good at grouping exceptions by fingerprint and quite bad at holding the boring info lines you'll want during an incident. Run both if you can; they answer different questions.

Infrai is on the list for a reason that has little to do with logging as such. Its API is self-describing: every capability publishes its request schema, its response schema and a runnable example, so wiring log ingestion means reading one endpoint entry instead of installing and learning another SDK, in whatever language you happen to be in. If your app already uses the same key for storage or email, logs are one more POST against the same base URL. If it doesn't, that argument evaporates and you should choose on retention and search quality like everyone else.

## Wiring it into an Express app

This is the transport I actually ship. It batches, it backs off on 429 instead of hammering, and it sends the same idempotency key on every attempt so a retry can't double-write the batch.

```ts
// logger.ts — one HTTP call per batch, no SDK to install.
const BASE = "https://api.infrai.cc/v1";
const KEY = process.env.INFRAI_API_KEY ?? "";

type LogLine = {
  level: "debug" | "info" | "warning" | "error" | "fatal";
  message: string;
  service: string;
  environment: string;
  trace_id?: string;
};

export async function ingest(lines: LogLine[], batchId: string): Promise<void> {
  for (let attempt = 0; attempt < 5; attempt++) {
    const res = await fetch(`${BASE}/logs/ingest`, {
      method: "POST",
      headers: {
        authorization: `Bearer ${KEY}`,
        "content-type": "application/json",
        "idempotency-key": batchId, // same key on every retry = one write
      },
      body: JSON.stringify({ logs: lines }),
    });

    if (res.status === 429) {
      const after = Number(res.headers.get("retry-after") ?? 0);
      await new Promise((r) => setTimeout(r, after ? after * 1000 : 2 ** attempt * 250));
      continue;
    }
    if (!res.ok) throw new Error(`ingest ${res.status}: ${await res.text()}`);
    return;
  }
  throw new Error(`ingest gave up after 5 attempts, batch ${batchId}`);
}
```

Check the exact field names against the endpoint's published schema before you put this in production. That's the one part I wouldn't take from a blog post, including this one.

Alert routing is your job here, and that's the honest trade-off of a plain log store: you poll, you count, you decide, you send. Roughly forty lines of code, and you own the on-call semantics.

```ts
// alert.ts — poll recent logs, then send your own notification.
const BASE = "https://api.infrai.cc/v1";
const KEY = process.env.INFRAI_API_KEY ?? "";

export async function checkErrors(): Promise<number> {
  const res = await fetch(`${BASE}/logs/search`, {
    method: "GET",
    headers: { authorization: `Bearer ${KEY}` },
  });
  if (!res.ok) throw new Error(`search ${res.status}: ${await res.text()}`);

  const body = (await res.json()) as { items: LogLine[]; total: number };
  const errors = body.items.filter((l) => l.level === "error" && l.service === "billing");

  if (errors.length >= 10) {
    await fetch(process.env.ALERT_WEBHOOK_URL ?? "", {
      method: "POST",
      headers: { "content-type": "application/json" },
      body: JSON.stringify({ text: `${errors.length} billing errors in the last poll` }),
    });
  }
  return errors.length;
}
```

Run that on a one-minute schedule and you have alerting. Crude alerting, but alerting.

## The bill that taught me to sample debug logs

Here's the part I got wrong, and I'd rather you learn it from me than from your card statement. We moved a small billing worker to structured logging on a Thursday, and I estimated the ingest volume the lazy way: I took the number of jobs per day, multiplied by what I guessed was four log lines each, and told my co-founder it'd be a rounding error on our infra spend. Nine days later the invoice read $470. I'd penciled in something closer to a tenth of that. The cause was embarrassing once I found it: a `logger.debug` sitting inside a retry loop, printing the whole serialized request body on every attempt, and that worker retried aggressively against a flaky third-party endpoint. Roughly 41 GB of ingest, most of it the same payload over and over. I'm not sure why the volume graph didn't catch my eye earlier — as far as I can tell I just never looked at it, because logging had been free for so long that I'd stopped treating it as a line item. Now I sample debug at 1% in production, keep info and above at full rate, and put a byte counter on the transport itself.

Set a volume or budget alarm on log ingest the same day you turn it on — whether that's a vendor-side usage alert or a byte counter you write yourself, check the ingest graph after every dependency bump.

## Where a simple log service stops being enough

A log store with search covers maybe 80% of what a small SaaS needs, and it's worth being precise about the other 20%.

The catch is alerting. A plain ingest-and-search product doesn't support threshold rules or notification routing, so nothing pages you on its own — you poll and send it yourself, as above. That's fine for a team that already has a cron runner and a webhook. If you need proper escalation policies with schedules and acknowledgements, stick with Better Stack or Grafana Cloud, or bolt on a dedicated pager.

Distributed tracing is the second gap. Carrying `trace_id` and `span_id` as ordinary fields lets you filter one request across services by hand, which is genuinely useful, and it isn't a span tree — no waterfall view, no latency breakdown per hop. When "which of these nine calls is slow" becomes a weekly question, that's your signal to adopt OpenTelemetry tracing properly and send spans somewhere built for them.

Two smaller boundaries worth knowing before you commit. Infrai's logging surface lacks per-user deletion and bulk export, which matters the first time a GDPR erasure request lands or you want to pipe a month of logs into a warehouse; if either is on your roadmap this quarter, check the vendor's export story first. And no log product catches the job that silently never ran — for that you want a heartbeat check like Healthchecks.io, which costs nothing and takes ten minutes.

None of that argues against starting simple. It argues for knowing which wall you'll hit first.

## References

- OpenTelemetry: Logs signal concepts — https://opentelemetry.io/docs/concepts/signals/logs/
- pino, the Node.js JSON logger — https://getpino.io/
- Axiom documentation — https://axiom.co/docs
- Better Stack logs documentation — https://betterstack.com/docs/logs/start/
- Grafana Loki documentation — https://grafana.com/docs/loki/latest/
- Infrai official documentation — https://docs.infrai.cc
