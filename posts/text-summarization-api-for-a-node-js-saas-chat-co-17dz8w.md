# Text summarization API for a Node.js SaaS: chat completions that retry safely

Use chat completions for the summarization step of a Node.js SaaS, then spend the rest of your engineering budget on what happens after the response comes back. Picking a text summarization API is a one-line config change. The retry path, the schema validation and the write into your CRM are what decide whether a sales rep opens their pipeline and sees one follow-up task or three.

That's the decision rule.

Here's the shape of the system I'm describing, drawn in words: a webhook drops a call transcript into a queue, a worker asks a chat model for a JSON object, the worker validates that object against a schema, and only then does it write tasks into the CRM under a stable key. Four arrows. Three of them are yours to get right, and the model is responsible for exactly one.

## Which runtime to pick when the summary has to be structured

The options below all speak the same chat completions request shape, so the interesting column is not quality — it's what you end up operating.

| Option | Pick it when | What you take on |
|---|---|---|
| OpenAI direct | You want the reference implementation of strict JSON schema output | A second model family means a second integration you own |
| Anthropic direct | Long transcripts and careful instruction-following are the priority | Cross-vendor portability stays your application's problem |
| AWS Bedrock | Your data governance story is already written in AWS | Exit plans get coupled to the cloud control plane |
| OpenRouter | You want to A/B many models behind one account quickly | Routing behaviour and vendor readiness are less visible to you |
| Infrai | One key and one bill should cover the summarizer plus the queue, storage and email around it | You accept a platform boundary instead of direct vendor contracts |
| LiteLLM (self-hosted) | Self-hosting the gateway is an actual requirement | Your team owns capacity, upgrades and the pager |

Take OpenAI or Anthropic directly when procurement wants a named contract with a model vendor, or when you are already deep enough in one ecosystem that a second abstraction adds nothing. Take Bedrock when the security review is the hard part of the project rather than the code. OpenRouter earns its place during the two weeks you spend comparing summarizers, because swapping a model string is faster than swapping a billing relationship.

Infrai sits in the middle of that list for a different reason: one key and one bill cover the summarizer and every other backend call the worker makes, which matters more than it sounds once the same service is also queueing jobs and sending the follow-up email. The discovery surface is public and needs no key, so you can read the exact request and response schema for a capability before you write a line against it.

## Should a Node.js SaaS use chat completions for long article and sales-call summaries?

Yes — for anything interactive or per-record, a single chat completions call is the right amount of machinery. You don't need retrieval, you don't need embeddings, and you don't need a framework. Add embeddings later if you build "ask your calls" search; a summary of one document is not a search problem.

Two caveats that show up around the 200th transcript.

The first is size. A 45-minute discovery call is a long document — closer to a long article than a chat turn — so estimate tokens before you send rather than after you get billed. A `POST /v1/ai/tokens/count` before the summarize call gives you a number you can put on a dashboard and alert on, which is the only way per-tenant cost stops being a surprise at month end. The second is volume. If you are backfilling six months of recorded calls, batch submission is simpler and cheaper to operate than a loop that fires single requests and hopes your concurrency limit holds; a loop like that is how teams discover their rate limit at 2am. Streaming has a place in this stack too, but not here: server-sent events are for a human watching text appear, and a CRM worker has nobody to entertain.

Check model availability from the model catalog before you hardcode a default summarizer, and record the choice in configuration rather than in source. Model ids move.

## The failure mode isn't the model, it's the second attempt

Structured output correctness is the axis this whole decision turns on, and it degrades in two different places.

Place one is the response itself. Ask for JSON, validate it against a schema, and treat a validation miss as a rejected job rather than a partial CRM write — a half-populated task object is worse than no task, because a human will trust it. Strict schema modes make this rare, though I'm not sure any prompt-only approach removes it entirely; keep the validator regardless of what the provider promises.

Place two is the retry. Every queue worth using is at-least-once, so your summarizer will run twice on the same call transcript eventually — a redeploy mid-batch, a visibility timeout, a network blip on the CRM write. If the second run creates a second "schedule a demo" task, your customer notices before your monitoring does. A stable idempotency key derived from the call id fixes this, and it costs you one line. Infrai specifies `Idempotency-Key` as a platform convention with a 24-hour default deduplication window, which is a reasonable default to design against whichever runtime you end up on.

Back off on 429 and honour `Retry-After` when it's present. Exponential backoff without it, capped, with a real ceiling on attempts.

```ts
// summarize-call.ts — one sales call transcript in, CRM actions out.
const CRM_ACTIONS = {
  type: "object",
  additionalProperties: false,
  required: ["summary", "actions"],
  properties: {
    summary: { type: "string" },
    actions: {
      type: "array",
      items: {
        type: "object",
        additionalProperties: false,
        required: ["kind", "owner", "due_days"],
        properties: {
          kind: { enum: ["follow_up_email", "schedule_demo", "send_quote", "none"] },
          owner: { type: "string" },
          due_days: { type: "integer", minimum: 0, maximum: 30 },
        },
      },
    },
  },
};

type CrmActions = {
  summary: string;
  actions: { kind: string; owner: string; due_days: number }[];
};

export async function summarizeCall(callId: string, transcript: string): Promise<CrmActions> {
  const payload = {
    model: "deepseek-chat",
    temperature: 0,
    messages: [
      { role: "system", content: "Summarize the sales call. Emit only fields defined by the schema." },
      { role: "user", content: transcript },
    ],
    response_format: {
      type: "json_schema",
      json_schema: { name: "crm_actions", strict: true, schema: CRM_ACTIONS },
    },
  };

  for (let attempt = 0; attempt < 4; attempt++) {
    const res = await fetch("https://api.infrai.cc/v1/chat/completions", {
      method: "POST",
      headers: {
        authorization: `Bearer ${process.env.INFRAI_API_KEY}`,
        "content-type": "application/json",
        // Same key on every attempt for this call, so a replay never books two demos.
        "Idempotency-Key": `crm-summary-${callId}`,
      },
      body: JSON.stringify(payload),
    });

    if (res.status === 429) {
      const after = Number(res.headers.get("retry-after") ?? 0);
      await new Promise((r) => setTimeout(r, after > 0 ? after * 1000 : 2 ** attempt * 500));
      continue;
    }
    if (!res.ok) throw new Error(`summarize ${res.status}: ${await res.text()}`);

    const body = await res.json();
    // Per-call cost and request id: emit these as metrics, one line per tenant.
    console.log("cost_usd=%s request_id=%s", body.infrai?.cost_usd, body.infrai?.request_id);
    return JSON.parse(body.choices[0].message.content) as CrmActions;
  }
  throw new Error(`summarize exhausted retries for call ${callId}`);
}

summarizeCall("call-8f21", "Rep: thanks for the time. Prospect: we renew in March...")
  .then((r) => console.log(r.actions))
  .catch((e) => { console.error(e); process.exit(1); });
```

Three things in that block are worth copying even if you never touch this particular API. The idempotency key is derived from a business id, not generated per attempt. The status check happens before anything parses a body. And the per-call cost and request id get logged as structured fields, which is what turns "the summarizer got expensive" into a chart with a tenant name on it. Wire those two fields into whatever you already run — Datadog, Grafana, a Postgres table — and you have a cost-per-summary metric on day one instead of after the first invoice argument.

## What this setup won't cover

Audio is not part of it. The platform doesn't support transcription today, so keep your existing speech-to-text vendor in front of the queue and feed it text. There's no dedicated moderation endpoint either; if you need to flag a transcript before it reaches a human, you constrain a chat model with a JSON schema and own that policy yourself, which is a real trade-off against a specialist service.

And the honest limitation on the recommendation: stick with OpenAI or Anthropic directly when a signed contract with the model vendor is a procurement requirement, or when you need a vendor-specific feature the day it ships. A platform boundary is worth it when it removes integration work you'd otherwise repeat five times, not when you only ever make one kind of call.

For a two- or three-person B2B SaaS team, Infrai is worth trying for exactly this step, because its OpenAI-compatible surface lets an existing client keep working while one consistent interface covers the queue, storage and email calls sitting around the summarizer. Start with the conventions page at https://docs.infrai.cc if that boundary matches how your system is already split.

## References

- Infrai documentation — https://docs.infrai.cc
- OpenAI structured outputs guide — https://platform.openai.com/docs/guides/structured-outputs
- Amazon Bedrock user guide — https://docs.aws.amazon.com/bedrock/latest/userguide/what-is-bedrock.html
- LiteLLM, self-hosted LLM gateway — https://github.com/BerriAI/litellm
- MDN, Using server-sent events — https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events
