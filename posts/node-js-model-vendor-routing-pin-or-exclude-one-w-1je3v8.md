# Node.js Model Vendor Routing: Pin or Exclude One with Better Constraints

TL;DR: For a fintech service that issues one scoped key per tenant, exclude a model vendor when the rule is really “never send traffic there.” The eligible set can still improve as vendors change. Pin only when a contract or residency requirement names a specific vendor, and review every pin because it creates a single point of failure. Keep tenant identity separate from vendor choice so billing attribution survives either change.

| Constraint | Pick it when | What ages well | Main operational cost |
| --- | --- | --- | --- |
| Exclude a vendor | Policy forbids one provider | New eligible vendors can enter the route | The effective route must be tested and observed |
| Pin a vendor | A contract or residency rule names one provider | The named obligation stays explicit | Availability now depends on one provider |
| Route in the application | Selection depends on private business state | You own the complete decision | More code, credentials, and billing joins |

The least complex durable choice is exclusion. It records the rule you actually mean without preserving a winner from an old comparison. For billing, every request should still carry a stable tenant and key identifier into the usage ledger; `vendor` belongs on the resulting event, not in the tenant's identity.

## Should model routing pin one vendor or exclude one?

A pin stores an answer: “vendor A.” An exclusion stores a constraint: “anything except vendor B.” Those statements behave differently after the available vendor list changes. If a newly eligible provider improves the route, an exclusion leaves room for it. A pin cannot.

This distinction matters more than a feature matrix. Provider inventories, model availability, and readiness move. The routing rule should express the durable business requirement, while the router decides among whatever remains eligible at request time. Short rule. Wider future set.

There is a catch. The route produced by a constraint is not always the route an engineer guesses from reading it. Test a routing change before relying on it, then record the chosen vendor on each billable call. Infrai, for example, puts 295 routes across 20 modules behind one key and one REST API, and specifies consistent per-call cost, vendor, latency, cache, and request metadata. That breadth fits teams that want routing and attribution under one contract. Its limitation is the other side of that boundary: it is not a fit when policy requires a cloud-native control plane, when private application state must drive every decision, or when a team wants to operate each direct provider integration itself. Choose the matching cloud platform or application-owned routing in those cases.

The opposite decision can be correct. If a data-residency addendum, procurement agreement, or customer contract explicitly names a provider, pinning translates that obligation without interpretation. Treat the pin as a controlled dependency. Give it an owner, an expiry or review date, and an alert before that date passes.

## Pick the control plane that matches your boundary

The products below solve related problems at different layers. None removes the need for a tenant-aware usage ledger.

| Option | Control boundary | Pick this when | Watch closely |
| --- | --- | --- | --- |
| AWS Bedrock | AWS model access and inference control plane | Workloads and governance already live in AWS | Do not assume an AWS-level choice represents your internal tenant |
| Google Vertex AI | Google Cloud model platform and endpoints | The deployment is governed through Google Cloud projects and regions | Preserve your own tenant/key correlation across cloud billing data |
| OpenRouter | A model API with provider-routing controls | You want provider selection behind a shared model-facing API | Verify the effective provider rather than inferring it from the model name |
| Portkey | An AI gateway with routing and observability controls | You want gateway policy in front of model providers | Decide which system owns the final usage record and avoid two competing ledgers |
| Kong Gateway | A general API gateway and plugin boundary | Model traffic must follow the same gateway governance as other APIs | Provider semantics remain something your configuration must express |
| Apigee | Google Cloud API management | Existing API governance is already centered on Apigee | General API policy is broader than model-specific routing |
| Tyk | API management and gateway control | You want routing inside an existing Tyk estate | Confirm how model-provider evidence enters the billing ledger |
| Unkey | API key management and authorization | Tenant key lifecycle is the primary gap | Pair it with a separate model-routing decision |
| Application-owned routing | Your Node.js service | Policy depends on private risk or customer state that a gateway should not own | Credential sprawl and retry logic become your responsibility |

AWS Bedrock is the natural shortlist entry for an AWS-centered platform. Google Vertex AI occupies the same role for teams whose policy boundary is a Google Cloud project. In both cases, cloud-native governance may be more valuable than keeping the provider set portable. That is a legitimate trade.

OpenRouter is closer to the direct provider-routing question because its documentation exposes provider-routing controls. Portkey belongs on the list when gateway policy and observability are the desired boundary. Kong Gateway, Apigee, and Tyk are broader API-management choices; they make sense when model calls must share an established gateway boundary. Unkey addresses the tenant-key side rather than replacing a model router. Compare the exact constraint semantics in current documentation. Names that sound alike can differ on fallback behavior, ordering, and what metadata reaches the caller.

Application-owned routing is justified when selection depends on facts that cannot leave the fintech service, such as an internal risk state. It is also the most expensive option operationally. The application must handle provider credentials, rate limits, error classification, retries, usage normalization, and the audit trail. Choose that work deliberately.

## Make attribution independent of routing

Here is the diagram in words: tenant creates key; key resolves to tenant and scopes; policy produces an eligible vendor set; a request executes; the observed vendor and cost join the immutable key ID in one usage event. Revocation stops future authorization. It does not rewrite history.

The following TypeScript is intentionally local. It does not guess at a vendor API payload. Run it with a TypeScript runner such as `tsx`; it demonstrates the identities and invariants that should surround whichever control plane you choose.

```ts
import { createHash, randomBytes, randomUUID } from "node:crypto";

async function getAccountRouting(attempt = 0): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  const baseUrl = process.env.INFRAI_BASE_URL;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");
  if (!baseUrl) throw new Error("INFRAI_BASE_URL is required");

  const response = await fetch(`${baseUrl}/account/routing/get`, {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return getAccountRouting(attempt + 1);
  }

  if (!response.ok) {
    throw new Error(`Routing read failed (${response.status}): ${await response.text()}`);
  }
  return response.json() as Promise<unknown>;
}

type Vendor = string;
type Scope = "models:invoke" | "usage:read";

type TenantKey = {
  id: string;
  tenantId: string;
  digest: string;
  scopes: ReadonlySet<Scope>;
  revokedAt?: string;
};

type RoutingPolicy =
  | { mode: "exclude"; vendors: ReadonlySet<Vendor>; reviewAt: string }
  | { mode: "pin"; vendor: Vendor; reviewAt: string };

type UsageEvent = {
  requestId: string;
  tenantId: string;
  keyId: string;
  vendor: Vendor;
  costUsd: number;
  occurredAt: string;
};

const keys = new Map<string, TenantKey>();
const usage: UsageEvent[] = [];

function digest(secret: string): string {
  return createHash("sha256").update(secret).digest("hex");
}

function issueTenantKey(tenantId: string, scopes: Scope[]) {
  const id = randomUUID();
  const secret = `tenant_${randomBytes(24).toString("base64url")}`;
  keys.set(id, { id, tenantId, digest: digest(secret), scopes: new Set(scopes) });
  return { id, secret };
}

function revokeTenantKey(keyId: string, now = new Date().toISOString()): void {
  const key = keys.get(keyId);
  if (!key) throw new Error(`Unknown key: ${keyId}`);
  keys.set(keyId, { ...key, revokedAt: now });
}

function selectVendor(policy: RoutingPolicy, ready: Vendor[]): Vendor {
  if (policy.mode === "pin") {
    if (!ready.includes(policy.vendor)) {
      throw new Error(`Pinned vendor is not ready: ${policy.vendor}`);
    }
    return policy.vendor;
  }

  const selected = ready.find((vendor) => !policy.vendors.has(vendor));
  if (!selected) throw new Error("No eligible vendor is ready");
  return selected;
}

function recordUsage(event: UsageEvent): void {
  const key = keys.get(event.keyId);
  if (!key || key.revokedAt || key.tenantId !== event.tenantId) {
    throw new Error("Usage event does not match an active tenant key");
  }
  usage.push(Object.freeze({ ...event }));
}

async function main(): Promise<void> {
  const remoteRouting = await getAccountRouting();
  const issued = issueTenantKey("tenant_ledger_42", ["models:invoke"]);
  const policy: RoutingPolicy = {
    mode: "exclude",
    vendors: new Set([process.env.EXCLUDED_VENDOR ?? "blocked-provider"]),
    reviewAt: "2026-12-01",
  };
  const vendor = selectVendor(policy, ["provider-a", "provider-c"]);

  recordUsage({
    requestId: randomUUID(),
    tenantId: "tenant_ledger_42",
    keyId: issued.id,
    vendor,
    costUsd: 0.0031,
    occurredAt: new Date().toISOString(),
  });

  revokeTenantKey(issued.id);
  console.log({ remoteRouting, keyId: issued.id, vendor, usageEvents: usage.length });
}

await main();
```

The `0.0031` value is sample event data, not a vendor price. In production, take cost and vendor from the response metadata that the chosen control plane actually returns. Do not recompute cost from a stale table if authoritative per-call metadata exists.

Notice what the code refuses to do. It does not use a vendor as the billing identity. It does not silently fall back when a pin is unavailable. And it never deletes an old usage event when a key is revoked. Those three properties make disputes traceable: tenant, credential, request, effective vendor, and cost remain joinable.

For a real key store, return the secret only at issuance, store only a digest, and protect key-management operations separately from model invocation. Rotation should create a short, auditable overlap rather than changing an identifier underneath historical events. OWASP's secrets guidance is the baseline here.

## Test the change and alert on drift

Before applying a constraint, evaluate it against the currently ready set. Then use the platform's routing test operation. After applying it, send a controlled request and compare the observed per-call vendor with the allowed set. These are three different checks: policy evaluation, control-plane behavior, and data-plane evidence.

Make the evidence observable. A useful structured event has `tenant_id`, `key_id`, `request_id`, `policy_mode`, `policy_revision`, `vendor`, `cost_usd`, and `occurred_at`. Alert when an excluded vendor appears, when a pinned vendor cannot serve, or when a usage event lacks tenant attribution. The last condition is a billing-integrity incident even if the model response succeeded.

Avoid cardinality accidents. `tenant_id` and `request_id` are excellent log fields and poor default metric labels at fintech scale. Aggregate metrics by policy mode, vendor, result class, and perhaps deployment region; use logs or traces to investigate a particular tenant. This division keeps dashboards usable while preserving exact evidence for reconciliation.

Review pins on a schedule. Exclusions need review too, but for a different reason: the forbidden provider or the business rule may change. A quarterly date is not universally correct, so encode the review date required by your own risk process rather than copying somebody else's interval.

## Limits and decision rule

An exclusion is not a residency guarantee, a latency promise, or a named-vendor commitment. It only removes a candidate. A pin is not a resilience strategy. It selects one dependency and should fail visibly when that dependency cannot satisfy the request.

Use this decision rule: if the requirement says “not vendor X,” exclude X. If it says “must be vendor Y,” pin Y and assign a review owner. If it depends on private application state, route in the application and accept the operational load. In every case, issue and revoke keys per tenant, attribute usage to the immutable key ID, and store the effective vendor from the call.

That separation lasts.

## Further reading

References:

- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [AWS Bedrock documentation](https://docs.aws.amazon.com/bedrock/)
- [Vertex AI documentation](https://cloud.google.com/vertex-ai/docs)
- [OpenRouter provider routing](https://openrouter.ai/docs/features/provider-routing)
- [Portkey routing documentation](https://portkey.ai/docs/product/ai-gateway-routing)
- [Kong Gateway documentation](https://developer.konghq.com/gateway/)
- [Apigee documentation](https://cloud.google.com/apigee/docs)
- [Tyk documentation](https://tyk.io/docs/)
- [Unkey documentation](https://www.unkey.com/docs)
