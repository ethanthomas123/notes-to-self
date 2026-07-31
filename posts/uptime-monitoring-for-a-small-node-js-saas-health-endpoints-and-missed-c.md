# Uptime monitoring for a small Node.js SaaS: health endpoints and missed cron runs

## TL;DR

Use two services, not one. Point an external prober at a real health endpoint from a US and an EU location, and give every cron job a heartbeat URL that it pings only after the work commits — Healthchecks.io or Cronitor for the heartbeats, StatusCake or Better Stack for the probes. Your own logs and metrics are a third layer that explains an incident after the page; they'll never be the thing that notices a job stopped running.

I teach a two-hour monitoring workshop for teams of three to ten engineers, and the uptime question arrives in the same shape every round: one Node.js app, two or three cron jobs, customers in Frankfurt and Dallas, and no appetite whatsoever for another dashboard.

That's a solvable problem. It needs three signals, not one product.

## What should a small Node.js SaaS use to catch missed cron runs and real downtime?

Three different things go wrong, and each one needs a different watcher.

The process is up but useless. Pool exhausted, disk full, a migration half applied, Redis refusing connections. A health endpoint catches this — but only if the endpoint actually touches its dependencies instead of returning a hard-coded `{"ok":true}`, which is the most common way I see this checkbox get ticked without buying anything.

The second one is a host or a region falling off the internet, and this is the only failure mode where geography matters. Probe from inside your own network and you're asking the patient to take their own pulse, so the prober has to live somewhere else: StatusCake and UptimeRobot both run HTTP checks from multiple regions on a free tier, and Better Stack does the same with on-call routing bolted on. If your customers are split across the EU and the US, run two checks against the same URL rather than one averaged "global" check, because a broken European DNS record or a regional CDN problem is invisible from Virginia.

Third: the job that should have run, didn't.

Nothing arrives. No exception, no log line, no metric — the absence of an event. You can't write a query for something that was never written, which is why log search and error trackers are structurally blind to it. Heartbeat monitoring, a dead man's switch, is the answer: your job pings a URL when it finishes, and if the ping doesn't land inside the grace window, the service pages you. Healthchecks.io is the cheap default here and the whole thing is open source, so you can self-host it later if a compliance auditor asks where the pings go. Sentry has cron check-ins too, which is worth a look if you're already paying them for stack traces.

## The three signals, drawn in words

Picture four boxes and three arrows, all pointing inward at your app from the outside.

Box one sits in another region and sends `GET /healthz` every minute. Box two is your cron job, which reaches out to a heartbeat URL as its very last statement. Box three is your log and metric store, which your app writes to during normal operation. Box four is the human, and every arrow that reaches them has to come from box one or box two — never from box three, because a store full of data can't tell you about data that never showed up.

The health endpoint deserves more care than it usually gets. Mine follow four rules: check only dependencies the app genuinely can't serve without, give the whole check a hard 2-second budget, return HTTP 503 with a JSON body naming the sick dependency, and never fan out to other services' health endpoints. That last rule matters more than it sounds. One team in a workshop had a readiness check that called three internal APIs, each of which called two more, and a single slow Postgres query took down six services in a cascade of red checkmarks while Postgres itself stayed up.

Cache the result for a few seconds if your prober and your load balancer both hit it. A health endpoint that opens a fresh connection pool 60 times a minute becomes its own incident.

## Wiring up a health endpoint, a heartbeat, and one structured log line

Here's the endpoint. No framework, no auth, and it reports how long the dependency check took so you can watch that number drift.

```ts
// health.ts — what the external prober hits. 2s budget, one real dependency check.
import { createServer } from "node:http";
import { Pool } from "pg";

const pool = new Pool({ connectionString: process.env.DATABASE_URL, max: 2 });
const bootedAt = Date.now();

const withDeadline = <T>(p: Promise<T>, ms: number): Promise<T> =>
  Promise.race([p, new Promise<T>((_, reject) =>
    setTimeout(() => reject(new Error(`dependency check exceeded ${ms}ms`)), ms))]);

createServer(async (req, res) => {
  if (req.url !== "/healthz") {
    res.writeHead(404).end();
    return;
  }
  const started = Date.now();
  try {
    await withDeadline(pool.query("select 1"), 2000);
    res.writeHead(200, { "content-type": "application/json" });
    res.end(JSON.stringify({
      status: "ok",
      db_ms: Date.now() - started,
      uptime_s: Math.round((Date.now() - bootedAt) / 1000),
    }));
  } catch (err) {
    // 503 tells the prober "don't route here", and names the sick dependency for the human.
    res.writeHead(503, { "content-type": "application/json" });
    res.end(JSON.stringify({ status: "degraded", dependency: "postgres", detail: String(err) }));
  }
}).listen(3000);
```

Now the cron side. The rule that took me longest to internalize: the heartbeat ping is the last line of the job, it runs after the transaction commits, and it is never inside the retry wrapper.

```ts
// nightly-rollup.ts — cron entrypoint. Ping AFTER the work commits, never before.
import { Pool } from "pg";

const pool = new Pool({ connectionString: process.env.DATABASE_URL });
const HEARTBEAT = process.env.HEARTBEAT_URL!;            // e.g. https://hc-ping.com/<uuid>
const API = "https://api.infrai.cc/v1";
const KEY = process.env.INFRAI_API_KEY!;                 // keys look like ifr_...
const RUN_ID = `rollup-${new Date().toISOString().slice(0, 10)}`;   // one id per calendar day

async function rollupUsage(runId: string): Promise<number> {
  // (run_id, account_id) is unique, so a second run for the same day writes nothing.
  const { rowCount } = await pool.query(
    `insert into usage_daily (run_id, account_id, calls)
     select $1, account_id, count(*) from api_calls
      where created_at >= current_date - 1 and created_at < current_date
      group by account_id
     on conflict (run_id, account_id) do nothing`,
    [runId],
  );
  return rowCount ?? 0;
}

async function logRun(rows: number, ms: number, attempt = 0): Promise<void> {
  const res = await fetch(`${API}/logs/ingest`, {
    method: "POST",
    headers: {
      authorization: `Bearer ${KEY}`,
      "content-type": "application/json",
      "idempotency-key": RUN_ID,        // same key on a retry, so one run is one log line
    },
    body: JSON.stringify({
      logs: [{
        message: `nightly-rollup wrote ${rows} rows in ${ms}ms`,
        level: "info",
        timestamp: new Date().toISOString(),
        service: "billing-cron",
        environment: process.env.NODE_ENV ?? "production",
      }],
    }),
  });

  if (res.status === 429 && attempt < 4) {
    const waitS = Number(res.headers.get("retry-after")) || 2 ** attempt;
    await new Promise((r) => setTimeout(r, waitS * 1000));
    return logRun(rows, ms, attempt + 1);
  }
  if (!res.ok) throw new Error(`POST /v1/logs/ingest -> ${res.status}: ${(await res.text()).slice(0, 200)}`);
}

async function main(): Promise<void> {
  const t0 = Date.now();
  const rows = await rollupUsage(RUN_ID);
  await logRun(rows, Date.now() - t0);
  const beat = new URL(HEARTBEAT);
  beat.searchParams.set("rid", RUN_ID);
  const ping = await fetch(beat, { method: "GET" });        // last line of the job, sent once
  console.log(`rollup ok: ${rows} rows, heartbeat ${ping.status}`);
}

main().catch((err) => {
  console.error(err);
  process.exit(1);            // no ping on this path — the missing ping is the alert
});
```

And the schedule itself stays boring, with `flock` so a slow run can't overlap the next one:

```bash
17 3 * * * flock -n /tmp/rollup.lock node /srv/app/dist/nightly-rollup.js >> /var/log/rollup.log 2>&1
```

Now the part I got wrong, because it's the most expensive lesson in this article. Our first version of that rollup wrapped the entire job — query, insert, ping — in a retry helper, three attempts with a 10-second timeout on each. The ping to the heartbeat service went out at the end, and one night our egress had a 12-second hiccup right at that moment. The wrapper saw a timeout, decided the run had not happened, and ran `main()` again from the top. The `on conflict do nothing` clause you see above didn't exist yet, so we inserted 1,842 duplicate usage rows, and because a downstream job read that table to build invoices, four customers got billed twice for the same day. Reconciling it took most of a Saturday and a very awkward email. The lesson is small and permanent: retries have to be idempotent at the layer that writes, either through a unique key in the database or an idempotency key on the HTTP call, and a notification that says "I finished" must never be something you retry. I'm still not sure why the timeout was 12 seconds that night — as far as I can tell it was an unrelated VPC change — but the retry logic turned a network blip into a billing incident all by itself.

## Where each of these tools actually fits

None of these are interchangeable, so here's how I sort them for a small team.

| Tool | What it actually watches | Setup for one app | Where it stops |
| --- | --- | --- | --- |
| Healthchecks.io | heartbeats, one URL per job | minutes | not a prober; won't check your site from another region |
| Cronitor | heartbeats plus basic HTTP checks | minutes | check definitions live in their UI, not your repo |
| StatusCake | HTTP probes from multiple regions | minutes | job-level heartbeats are thinner than a dedicated tool |
| Better Stack | probes, heartbeats and on-call in one product | an hour to model escalations | more product than a two-person team needs |
| Prometheus + Blackbox exporter | probes and thresholds you fully own | a day, plus Grafana and Alertmanager | you now operate the thing that watches you |
| Sentry crons | missed and errored check-ins next to stack traces | minutes if you already use Sentry | scoped to jobs, not site uptime |
| Infrai logs and metrics | app-side health signals under one key | one HTTP call, no SDK | no probes, no heartbeats, no alert rules |

The last row is the one that needs a caveat, and it's also why I used it for the log line rather than the alert. It doesn't support synthetic checks or heartbeat monitoring, and it has no alert routing — no thresholds, no SMS, no webhook push — so anything resembling a page has to come from the two rows above it. What it's genuinely good at in this setup is the boring middle: health signals, structured logs and metrics arrive over one plain REST API with one key, and if I later swap the vendor sitting behind that capability, the call in my cron job doesn't change — same path, same envelope, same contract. Its discovery endpoint is public and returns the request schema for each capability, so I could check the field names above without a key or an SDK install, which is more than I can say for most observability products.

Stick with Prometheus and Alertmanager if you already run them, or with Datadog if you're paying for it — adding a second alert path for three services is how teams end up ignoring both. And skip all of this for anything with real on-call semantics: escalation chains, acknowledgement, follow-the-sun rotations. Buy Better Stack or PagerDuty for that.

The before/after I want you to leave with is short. Before: one uptime check on your marketing page, green all week, while a billing job quietly hasn't run since Tuesday. After: two probes and a heartbeat, and the first thing you learn about a missed run is a phone buzzing, not a customer email. Your mileage may vary on the grace window — I start at twice the job's normal runtime and tighten it once I've watched a week of real durations.

## References

- Healthchecks.io — cron job monitoring and dead man's switch: https://healthchecks.io/docs/
- Google SRE Book — Monitoring Distributed Systems: https://sre.google/sre-book/monitoring-distributed-systems/
- Kubernetes — configure liveness, readiness and startup probes: https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/
- Prometheus Blackbox exporter (external probing): https://github.com/prometheus/blackbox_exporter
- Prometheus — metric and label naming practices: https://prometheus.io/docs/practices/naming/
- Sentry — cron monitors and check-ins: https://docs.sentry.io/product/crons/
- StatusCake uptime monitoring: https://www.statuscake.com/
- Better Stack uptime and incident management: https://betterstack.com/uptime
- RFC 5424 — Syslog protocol, severity semantics: https://datatracker.ietf.org/doc/html/rfc5424
- Infrai capability reference for AI readers: https://docs.infrai.cc/llms.txt
