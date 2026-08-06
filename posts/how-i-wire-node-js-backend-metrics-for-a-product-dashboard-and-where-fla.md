# How I Wire Node.js Backend Metrics for a Product Dashboard, and Where Flag Stats Fit

Bottom line: build the product analytics dashboard on a backend event ledger with a small custom metrics API in front of it, and keep feature flag exposure stats as a rollout instrument instead of an analytics source. Both can live in the same Node.js service. They answer different questions — flag stats tell you who saw a variant, while a metrics API tells you what those people did afterward, and how many times.

Here's the shape of it in words. A request lands in your Node.js handler; the handler appends one row per business event into a ledger table keyed by an event id; a rollup job folds those rows into per-day counters; the dashboard reads counters only, never raw rows. Flag evaluation logs sit off to the side and feed rollout health.

Two pipes, one dashboard, and a very clear line between them.

| Feed | Answers well | Falls down when | Erasure story |
| --- | --- | --- | --- |
| Backend event ledger + custom metrics API | Counts, funnels, revenue-shaped events, anything you must reconcile against the database | You need it live this afternoon; nobody owns the rollup job | Raw rows carry a subject id, so per-user deletion is a `DELETE` plus a rollup rebuild |
| Feature flag exposure stats | "Did the 10% cohort actually get the variant?", rollout health, kill-switch confidence | You ask it about behavior that happens minutes or days after the evaluation | Evaluation logs are usually short-retention and rarely designed as a system of record |
| Client-side analytics SDK | Page views, scroll depth, anything that never reaches your server | Ad blockers, offline tabs, and mobile network drops eat 10–30% of sends | Depends entirely on the collector you point it at |

## Which feed to pick, and when each one earns its keep

Pick the backend event ledger when the number has to be defensible. If someone in a review can ask "why does the dashboard say 4,812 activations and the billing table say 4,796?", you want one place to answer that, and it should be a table you own with a join key back to accounts. Server-side events survive ad blockers, they carry your own identifiers, and they can be replayed into a new rollup shape when the product question changes six weeks from now. That last property is the one people underrate. Pre-aggregated counters are cheap and fast, but you cannot un-sum them; a raw ledger lets you re-cut the same history by plan tier, by region, by whatever dimension the team invents next, without waiting a month for fresh data to accumulate.

Pick flag exposure stats when the question is about the rollout itself.

They're excellent at that. Evaluation counts per variant, per environment, per minute, with almost no work on your side, and they'll catch a misconfigured targeting rule long before your funnel notices. The catch is that an exposure event fires when the flag is read — which may be on every render, or once per session, or in a background prefetch that no human ever saw. Counting those as "users in the experiment" quietly inflates the denominator of every conversion rate you compute from them.

Client SDKs stay in the picture for pure front-end behavior. Scroll depth and rage clicks never reach your server, so a browser collector is the only thing that sees them.

## What can feature flag exposure stats actually tell a product analytics dashboard?

Three things, reliably: how many evaluations happened, which variant each one returned, and roughly when. That's the honest boundary. Exposure telemetry is a control-plane signal about your rollout, and most flag platforms document retention in days or weeks because that's the job it was built for.

What it can't tell you is what happened next. The evaluation record ends at "user 8814 got variant B at 14:02". Whether that user then completed checkout, invited a teammate, or churned two weeks later lives in your database, not in the flag system, and stitching the two together after the fact means exporting exposure logs and joining them to your own events anyway. At which point you've built the ledger — just with an extra vendor and an export job in the middle of it.

There's a subtler trap. Flag stats are usually aggregated at ingest, so cardinality is capped: variant, environment, maybe a rule id. Product analytics wants plan, cohort, referral source, device class, and the ability to add a seventh dimension next quarter. As far as I can tell, no flag platform is designed to be that store, and it'd be unfair to fault them for it — that's not the product.

## The Node.js ingest path: one idempotent write per event

This is where I spend most of my teaching time, because the naive version looks fine for months and then hands you a number that's wrong in one direction only.

My war story: we shipped an activation counter behind a retrying HTTP client, `retries: 2`, exponential backoff, the default in half the fetch wrappers on npm. A gateway timeout at 30s meant the client retried a write the server had already committed. The response never made it back; the row went in twice. Our weekly activation number ran about 3% high for eleven days before someone reconciled it against the accounts table, and it took me maybe 40 minutes staring at duplicate rows with identical payloads and timestamps 31 seconds apart to accept what had happened. Retries are safe on idempotent operations. `POST /events` is not idempotent unless you make it so — RFC 9110 is explicit that the property has to come from the operation itself, not from the client's good intentions.

The fix is one column and one constraint.

```ts
import { randomUUID } from "node:crypto";
import express from "express";
import { Pool } from "pg";

const pool = new Pool({ connectionString: process.env.DATABASE_URL });
const app = express();
app.use(express.json());

// event_id is supplied by the caller and unique-indexed in the ledger table:
//   create table events (
//     event_id   uuid primary key,
//     name       text not null,
//     subject_id text not null,
//     props      jsonb not null default '{}',
//     occurred_at timestamptz not null
//   );
app.post("/api/events", async (req, res) => {
  const { eventId, name, subjectId, props, occurredAt } = req.body ?? {};
  if (!eventId || !name || !subjectId) {
    return res.status(400).json({ error: "eventId, name and subjectId are required" });
  }

  const { rowCount } = await pool.query(
    `insert into events (event_id, name, subject_id, props, occurred_at)
     values ($1, $2, $3, $4, coalesce($5::timestamptz, now()))
     on conflict (event_id) do nothing`,
    [eventId, name, subjectId, props ?? {}, occurredAt ?? null],
  );

  // Same reply either way — a retry is a no-op, not an error the caller must handle.
  res.status(202).json({ accepted: true, duplicate: rowCount === 0 });
});

// Caller side: mint the id once, reuse it across every retry of that same event.
export async function trackActivation(subjectId: string, plan: string) {
  const eventId = randomUUID();
  await postWithRetry("/api/events", {
    eventId, name: "account.activated", subjectId, props: { plan },
    occurredAt: new Date().toISOString(),
  });
}
```

The id has to be minted by the caller, before the first attempt, and reused on every retry. Mint it inside the retry loop and you're back where you started with a fresh uuid per attempt. I've reviewed that exact mistake three times now.

Rollups then become boring, which is the goal:

```ts
// Runs every 5 minutes; recomputes only the days that changed.
const ROLLUP = `
  insert into metrics_daily (day, name, plan, value)
  select date_trunc('day', occurred_at) as day,
         name,
         coalesce(props->>'plan', 'unknown') as plan,
         count(*) as value
  from events
  where occurred_at >= now() - interval '2 days'
  group by 1, 2, 3
  on conflict (day, name, plan) do update set value = excluded.value
`;

export async function rollup() {
  const started = Date.now();
  await pool.query(ROLLUP);
  console.log(JSON.stringify({ msg: "rollup.done", ms: Date.now() - started }));
}

// The dashboard's read path never touches the ledger.
app.get("/api/metrics/daily", async (req, res) => {
  const { name, from, to } = req.query as Record<string, string>;
  const { rows } = await pool.query(
    `select day, plan, value from metrics_daily
     where name = $1 and day between $2 and $3 order by day`,
    [name, from, to],
  );
  res.json({ series: rows });
});
```

Because the rollup is idempotent too, you can re-run it over any window after a backfill and the numbers converge instead of doubling. Test it by inserting the same event twice in a fixture and asserting the counter moves once. That single test would have saved me eleven days.

## Retention, erasure, and the cost of keeping raw events

Keeping raw rows is what makes re-cutting possible, and it's also what puts you inside GDPR Article 17 — a data subject can request erasure, and "it's aggregated somewhere in a counter" is not an answer if the row that fed the counter still has their id on it. Design for that on day one; retrofitting a deletion path into a ledger with no subject key is genuinely painful.

The pattern that has held up for me: keep `subject_id` on every raw row, keep rollups free of any identifier, and make deletion a two-step — delete the subject's rows, then recompute the affected days. Rollups stay correct because they're derived, and the derivation is repeatable. Pair that with a retention window on the ledger itself (90 or 180 days of raw events, counters kept forever) and storage stops growing without bound. In my case a mid-sized B2B app produced roughly 2 GB of raw events a month, which is nothing for Postgres, but the same shape at consumer scale is where teams start reaching for columnar storage.

Instrumentation cost deserves a mention too. Every event you emit is a schema you now maintain, and the failure mode isn't disk — it's forty event names with overlapping meanings and nobody sure which one the dashboard uses. I cap teams at a dozen named business events until someone can articulate why the thirteenth matters. If you'd rather not invent that vocabulary yourself, the OpenTelemetry metrics data model is a reasonable spine to borrow — counter, gauge, histogram, plus attributes — even when you're storing rows in your own database rather than shipping them anywhere.

## Where this setup stops being the right answer

A hand-rolled ledger and metrics API is not a good fit for a two-person team that needs a funnel chart by Friday. You're signing up for a rollup job, a backfill story, a deletion path, and a dashboard nobody else can debug at 2am; a hosted analytics product absorbs all four for the price of less flexible querying, and that's often the better trade at small scale. Stick with client-side collection when the behavior you care about genuinely never touches your backend.

And if all you need is "how many people saw variant B", flag stats already answer it. Don't build anything.

I'm not sure there's a clean threshold where the custom path starts paying off — the teams I've seen do it well usually had a specific number they had to reconcile against billing, and that requirement, more than volume, is what pushed them. Your mileage may vary.

## References

- https://gdpr-info.eu/art-17-gdpr/
- https://datatracker.ietf.org/doc/html/rfc9110#name-idempotent-methods
- https://opentelemetry.io/docs/specs/otel/metrics/data-model/
- https://www.postgresql.org/docs/current/sql-insert.html#SQL-ON-CONFLICT
- https://developer.mozilla.org/en-US/docs/Web/API/Beacon_API
