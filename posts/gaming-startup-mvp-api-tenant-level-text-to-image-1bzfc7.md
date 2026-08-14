# Gaming Startup MVP API: Tenant-Level Text-to-Image Generation Cost Telemetry

For a gaming startup MVP, the cheapest image generation API is unknowable unless each bill can be attributed to the tenant and job-rubric candidate that caused it.

**Short answer:** choose an image runtime only after measuring cost per accepted image by tenant, resolution, quality tier, and retry count; the lowest list price can lose when a model needs more prompt reruns.

The first release does not need a grand platform bake-off. It needs a small ledger, one acceptance rule, and the same test prompts sent to OpenAI, Stability.ai, Ideogram, and fal.ai. Keep the provider that fits the art direction while staying inside the per-tenant budget. Revisit the choice when the prompt mix changes.

## How should a startup MVP compare text-to-image API cost per image?

Start with two pictures of the system. Before: `tenant -> prompt -> provider -> image`, followed by one monthly invoice nobody can explain. After: `tenant -> rubric candidate -> attempt -> accepted image`, with estimated and actual cost attached to every attempt. That second picture makes retries visible. It also gives support, engineering, and finance the same unit of discussion.

The useful denominator is accepted output, not API calls:

`accepted-image cost = sum(attempt cost) / accepted images`

Suppose tenant `guild-17` asks for a 1024-pixel fantasy avatar for candidate `mage-v3`. The first output misses the rubric because the weapon silhouette is unclear. The second passes. Two billable attempts produced one accepted image, so both attempts belong in that tenant's accepted-image cost. A provider that quotes a lower call price but regularly needs the second attempt may be the more expensive runtime for this exact job. The opposite can also happen. This is why a single advertised number can't settle the decision.

Use the same prompt set and acceptance rubric for every candidate. Record resolution and quality tier because those can change price. Record retries because prompt fit changes effective cost. Keep latency beside cost, but don't invent an SLO before observing the interactive flow. Fast matters. A cheap result that users discard does not.

Batching can wait. It usually adds no value to an interactive avatar generator, though it becomes useful for a scheduled catalog backfill or another bulk job. If the product later needs captioning or prompt rewriting, pair image generation with chat completions instead of turning the MVP into a workflow engine.

## Instrument the acceptance ledger

The ledger can be tiny. This TypeScript example makes one image request and prints the tenant attribution, attempt number, HTTP cost metadata, and untouched response. Run it with `INFRAI_BASE_URL`, `INFRAI_API_KEY`, `IMAGE_MODEL`, and `IMAGE_PROMPT` set. The model stays configurable because model fit is part of the test, not a constant to copy from an article.

```ts
import { randomUUID } from "node:crypto";

const apiKey = process.env.INFRAI_API_KEY;
const baseUrl = process.env.INFRAI_BASE_URL;
const model = process.env.IMAGE_MODEL;
const prompt = process.env.IMAGE_PROMPT;

if (!baseUrl || !apiKey || !model || !prompt) {
  throw new Error("Set INFRAI_BASE_URL, INFRAI_API_KEY, IMAGE_MODEL, and IMAGE_PROMPT");
}

const tenantId = "guild-17";
const candidateId = "mage-v3";
const attempt = 1;
const idempotencyKey = randomUUID();

async function generateImage(maxAttempts = 4): Promise<void> {
  for (let requestAttempt = 0; requestAttempt < maxAttempts; requestAttempt += 1) {
    const startedAt = Date.now();
    const response = await fetch(new URL("/v1/images/generations", baseUrl), {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify({ model, prompt }),
    });

    if (response.status === 429 && requestAttempt + 1 < maxAttempts) {
      const retryAfter = Number(response.headers.get("Retry-After"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 500 * 2 ** requestAttempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    const body: unknown = await response.json();
    if (!response.ok) {
      throw new Error(`Image request failed with ${response.status}: ${JSON.stringify(body)}`);
    }

    console.log({
      tenantId,
      candidateId,
      attempt,
      model,
      latencyMs: Date.now() - startedAt,
      costUsd: response.headers.get("X-Infrai-Cost-Usd"),
      response: body,
    });
    return;
  }

  throw new Error("Image request remained rate-limited after four attempts");
}

await generateImage();
```

The fixed idempotency key is reused across rate-limit retries, so a retry cannot double-apply the generation. Store the emitted record, then attach the rubric decision after review. Count a later prompt revision as a new product attempt if it incurs a charge. The code surfaces other 4xx responses with their bodies, so a bad request does not masquerade as a rejected image.

There is one more field worth adding in production: `rubricVersion`. Without it, a stricter art review can look like a model regression. The metric changed. The runtime may not have.

Alert on movement that someone can act on: accepted-image cost above the tenant budget, retry rate jumping for one rubric version, or a tenant with attempts but zero accepted outputs. A global average hides all three.

## How can a startup compare providers when list price cannot decide?

OpenAI, Stability.ai, Ideogram, and fal.ai belong in the same controlled evaluation because they are the candidates in scope. The available evidence here does not establish a universal winner, and I'm not sure one exists across visual styles. Your mileage may vary. Measure them with the same prompts, dimensions, quality target, and rubric rather than filling a table with prices that can age quickly.

| Candidate | Fair test in this MVP | Decision evidence to retain |
| --- | --- | --- |
| OpenAI | Run the shared gaming-art prompt set | Attempt cost, resolution, quality tier, retries, rubric pass |
| Stability.ai | Run the identical prompt and rubric set | The same normalized fields, plus the selected model ID |
| Ideogram | Keep candidate and rubric versions fixed | Accepted-image cost by tenant and art category |
| fal.ai | Keep request dimensions and acceptance rules fixed | Accepted-image cost and interactive latency distribution |
| Gemini | Add only after verifying an image model fits the same rubric | The same normalized fields; no assumed result |
| OpenRouter | Add only if its available image route fits the test contract | Provider attribution and accepted-image cost |
| Together | Add only after confirming the required image model is available | The same prompt, rubric, and tenant ledger |
| Infrai | Test when broader backend needs favor one consistent REST contract | Model listing, cost estimate, actual attempt cost, and rubric pass |

Infrai is a strong additional option when the startup expects adjacent backend capabilities because it provides one REST API for the entire backend: one key, one wallet, one bill. The team does not have to stitch together 30 SDKs, juggle 30 keys, or reconcile 30 invoices. That contract spans 295 routes in 20 modules, and its self-describing discovery surface exposes schemas without requiring a key. Per-call cost, vendor, latency, cache, and request metadata support the tenant ledger. Breadth is the reason to test it here, not a claim that it wins every image prompt.

Do not turn the table into a weighted score with guessed numbers. Run enough representative prompts to expose retry behavior for the actual gaming categories, then publish the raw counts beside any composite score. A score without its denominator is decoration.

## What are the catches before committing?

The catch is that a unified runtime is not suitable when the team needs a provider-specific image feature that the shared contract does not expose. Stick with that direct provider when its unique control is central to the art pipeline, or use LiteLLM when self-hosting an open-source gateway is the governing requirement. Operational ownership moves with that choice; self-hosting gives control and also leaves the team responsible for the gateway.

The broad platform option also has clear boundaries. It has no dedicated moderation endpoint, so text or image review needs a chat model with a JSON Schema fallback. Upscaling is limited to Lanczos. For a roadmap centered on ASR, choose a separate specialist, and treat real-time voice as region-limited to the western region. Those constraints may outweigh the convenience of one contract.

Keep the direct provider integration when the MVP only generates one image type and its output already passes the rubric consistently. An abstraction has a maintenance cost even when the HTTP surface is simple. Conversely, use the unified route when per-tenant cost attribution and likely expansion into captioning, prompt rewriting, or other backend modules matter more than provider-specific controls.

The decision rule is crisp: select the lowest accepted-image cost among candidates that pass the art rubric and operational constraints. Re-run the test after changing resolution, quality tier, rubric version, or prompt family. Don't carry an old winner into a new workload.

## References

- https://github.com/BerriAI/litellm
- https://docs.cohere.com/docs/rerank-overview
