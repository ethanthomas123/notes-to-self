# One Decision Per Request: Feature Flag Guards in Express and Node.js

**Short answer:** Resolve a feature flag once in Express middleware, store that decision on the request, and let the Node.js API route guard either continue or return a deliberate denial.

I teach logs, metrics, and alerting, and this is the before/after I draw on the board. Before: several handlers ask for the same flag, possibly receive different answers, and leave no clean record of the branch they took. After: one request enters one guard, receives one decision, emits one bounded metric, and follows one path. Crisp.

The route guard isn't the whole access-control system. Authentication and authorization still have their own jobs. A feature flag answers whether a rollout path is enabled for this request context; it shouldn't become a permanent substitute for permissions, plan entitlements, or input validation.

Keep it boring.

## How should Express middleware check a feature flag per Node.js API request?

Treat the guard as a decision boundary. The flow, in words, is `request -> identity context -> flag evaluation -> recorded decision -> route handler or denial`. Put authentication before the guard when the evaluation needs a stable subject identifier. Put the business handler after it. Don't evaluate again downstream.

That last rule matters. A flag configuration can change while a request is alive, and a provider can apply context differently if separate callers assemble separate inputs. Evaluating once gives every downstream function the same answer. Attaching the result to `res.locals` keeps the data scoped to the current request without modifying a process-wide object. It also makes the behavior easy to test: inject a provider, call the middleware, and assert either `next()` or the denied response.

Choose denial behavior as an API design decision. Returning `404` can avoid advertising an unreleased route; returning `403` can be clearer when an authenticated caller knows the capability exists. Consistency beats cleverness here. Document the choice next to the route contract, and don't let every handler improvise.

Evaluation failure needs an explicit policy too. For a guarded write, I usually choose a disabled fallback because an uncertain rollout decision shouldn't open a new mutation path. Your mileage may vary for a read-only cosmetic feature. Either way, record that the fallback was used; otherwise an evaluation problem looks exactly like a legitimate disabled decision.

Keep context small and intentional. A stable subject ID and a few bounded attributes are easier to reason about than forwarding headers wholesale — and they reduce the chance that sensitive or high-cardinality values leak into telemetry. The result is a narrow contract: context goes in, a typed decision comes out, and the request carries that decision forward.

## A copyable TypeScript route guard

The example below separates evaluation from HTTP behavior. The provider interface can wrap any internal or external flag source, while the middleware owns request scoping, fallback policy, telemetry, and denial. I like this split because unit tests don't need a network connection or a running metrics system.

```ts
import type { NextFunction, Request, RequestHandler, Response } from "express";

type FlagContext = {
  subjectId?: string;
};

type FlagDecision = {
  key: string;
  enabled: boolean;
  reason: "matched" | "default" | "evaluation_failed";
};

interface FlagProvider {
  evaluate(key: string, context: FlagContext): Promise<FlagDecision>;
}

interface Telemetry {
  count(name: string, labels: Record<string, string>): void;
  info(fields: Record<string, unknown>, message: string): void;
  warn(fields: Record<string, unknown>, message: string): void;
}

type GuardLocals = {
  subjectId?: string;
  flagDecisions?: Record<string, FlagDecision>;
};

export function requireFeatureFlag(
  key: string,
  provider: FlagProvider,
  telemetry: Telemetry,
  deniedStatus: 403 | 404 = 404,
): RequestHandler {
  return async (
    _req: Request,
    res: Response<unknown, GuardLocals>,
    next: NextFunction,
  ): Promise<void> => {
    let decision: FlagDecision;

    try {
      decision = await provider.evaluate(key, {
        subjectId: res.locals.subjectId,
      });
    } catch {
      decision = { key, enabled: false, reason: "evaluation_failed" };
    }

    res.locals.flagDecisions = {
      ...res.locals.flagDecisions,
      [key]: decision,
    };

    telemetry.count("feature_flag_checks_total", {
      flag: key,
      enabled: String(decision.enabled),
      reason: decision.reason,
    });

    const fields = { flag: key, reason: decision.reason };
    if (decision.reason === "evaluation_failed") {
      telemetry.warn(fields, "feature flag evaluation used the disabled fallback");
    } else {
      telemetry.info(fields, "feature flag decision resolved");
    }

    if (!decision.enabled) {
      res.status(deniedStatus).json({
        error: deniedStatus === 404 ? "not_found" : "forbidden",
      });
      return;
    }

    next();
  };
}
```

Mount this after middleware that establishes `res.locals.subjectId` and before the protected handler. The handler may read the stored decision for a variant or audit field, but it shouldn't call `evaluate` again. One request, one answer.

## Why the flag guard cannot make writes safe

A route guard controls reachability. It doesn't provide idempotency, transaction isolation, or retry safety. This objection comes up whenever I teach the pattern: “If the guard runs once per request, haven't we prevented duplicate work?” No. A retry is a new request, so the guard correctly runs again.

Retries happen.

I learned this with a duplicate-write bug: a naive client retry submitted the same operation twice, both requests received an enabled decision, and the database recorded 2 writes for one user action. I had instrumented the flag branch, so I could see both decisions, but visibility didn't undo either mutation. The repair was to give the operation an idempotency key and enforce uniqueness where the write was committed. I'm not sure why teams keep putting retry safety into a later cleanup ticket; the flag often increases exposure to a path before that ticket is touched. Build the write contract alongside the guard. Carry an idempotency key from the caller, bind it to the authenticated subject and operation, and make the uniqueness check atomic with the mutation. Decide what a replay returns. Then log a safe correlation value so two requests representing one intent can be connected without putting raw credentials or personal data into logs. This is the longer path through the example, but it's the important one: the guard can explain why code became reachable, while only the write boundary can guarantee that repeated intent doesn't become repeated state.

There is another boundary: a disabled route isn't authorization. If a caller must never perform an operation, enforce that with an authorization rule even when the feature is enabled. Flags change. Permissions describe who may act. Mixing them produces a nasty failure mode — a rollout edit quietly becomes a security-policy edit — and makes eventual flag removal dangerous.

The lifecycle choice is just as practical. Put an owner and removal condition beside the flag declaration. Once the rollout is permanent, remove the guard and its dead branch. Stick with ordinary configuration when the value changes only at deployment time and doesn't need per-request targeting; the extra evaluation path, telemetry volume, and operational dependency aren't suitable for a static setting.

## What should the guard log and measure?

Use metrics for rates and logs for individual explanations. I don't make one pretend to be the other.

Prometheus naming guidance favors a single base unit, meaningful suffixes, and labels that represent bounded dimensions. `feature_flag_checks_total` reads as a cumulative event counter. Labels such as `flag`, `enabled`, and a controlled `reason` set support useful aggregation. A user ID, request ID, or raw route parameter does not belong in metric labels because each new value creates another time series. Put request-level correlation in structured logs instead.

| Signal | Good fields | Answers | Main trade-off |
| --- | --- | --- | --- |
| Counter | flag, enabled, reason | How often did each decision occur? | Aggregation loses request detail |
| Structured log | request ID, flag, reason | Why did this request take that path? | Volume and retention grow with traffic |
| Latency measure | flag and duration in seconds | Is evaluation affecting request time? | Fine-grained labels can multiply series |
| Idempotency record | subject, operation, key | Is this write a replay? | Requires durable, atomic storage |

RFC 5424 gives severity levels an ordered meaning. In this design, an expected disabled decision is informational. An evaluation failure that activates a known fallback is a warning because normal behavior was degraded even though the request received a controlled response. Don't emit an error for every ordinary denial — noisy severity trains operators to ignore the channel.

The alert should describe user risk rather than raw activity. Watch the share of fallback decisions over all evaluations and pair it with route outcomes; a count alone rises whenever traffic rises. Keep a dashboard for rollout shape, a log query for request reconstruction, and a test that proves the metric and decision are emitted on both allowed and denied paths. The catch is volume: logging every successful decision may be unsuitable on a hot API. Sample routine success logs if needed, but retain complete counters and unsampled warning events. That balance preserves the before/after view I care about without turning observability into the dominant cost of the route.

## References

- Prometheus, “Metric and label naming”: https://prometheus.io/docs/practices/naming/
- IETF, “RFC 5424: The Syslog Protocol”: https://datatracker.ietf.org/doc/html/rfc5424
