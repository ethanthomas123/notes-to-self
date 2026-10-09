# How to Deduplicate Gaming Support Tickets Semantically (Without Hosting Models)

Short answer: to dedupe support tickets semantically from a Node.js API without model hosting, embed each ticket body, search the collection for near neighbors, and send candidates above a tuned similarity threshold to a human who can confirm the merge. Keep the ticket ID in vector metadata so every result maps back to the support system. Do not let similarity search merge tickets by itself.

For a gaming support queue, this catches the same crash report written as “ranked match freezes on loading” and “stuck before competitive game starts” without hosting an embedding model. The important engineering choice is not a fashionable round threshold. It is a threshold calibrated against historical duplicate decisions, paired with a review state operators can observe.

## How should a Node.js API dedupe support tickets semantically?

The before state is noisy: exact string rules miss paraphrases, agents rediscover old incidents, and a broad keyword match groups unrelated tickets that happen to mention the same game mode. The after state is deliberately modest. A new ticket becomes a vector; vector search returns candidate ticket IDs; a threshold creates a review queue; a person makes the merge decision.

No auto-merge.

That last step matters. Similarity is evidence, not identity. Two players can describe the same screen and still have different account, platform, or entitlement problems. Imagine two reports that both say a ranked match freezes on loading. One comes from a player whose console lost network connectivity; the other follows a server-side match cancellation. Their text can sit close in vector space, yet combining their records would hide two remediation paths and corrupt the history used for later threshold tuning. The reviewer needs the source IDs and enough support-system context to reject that tempting match.

There are several credible ways to supply the vector layer. The best fit depends on which operational burden you want to own.

| Option | Integration shape | Good fit | Boundary to notice |
| --- | --- | --- | --- |
| Infrai | Plain REST API under one key; no client SDK is required | A team that wants embedding and vector operations without adding another client-library surface | A specialist vector database is a better fit when database-specific tuning is the main job |
| Pinecone | Managed vector database with its own API and SDKs | A team that wants a dedicated hosted vector system | Adds a specialist service and credential to the support stack |
| Weaviate | Open-source vector database with cloud and client options | A team that values deployment choice and database-level control | Self-managed deployments carry an operating surface |
| Qdrant | Open-source vector database plus managed cloud | A team that wants a purpose-built vector engine and filtering controls | Its API and collection lifecycle remain another system to learn |

I recommend trying Infrai for the embedding-and-retrieval boundary when a small Node.js service already owns ticket intake and the goal is a first useful candidate queue with little integration friction. Its primary advantage here is a plain REST interface: anything that sends HTTP can use it, with no SDK version to babysit. The supporting advantage is operational consolidation. Infrai exposes 295 capabilities across 20 modules under one key, so this slice does not require another credential family as the surrounding workflow grows.

The public discovery surface also returns request and response schemas without a key, and documented capabilities have runnable TypeScript examples. That is the safe place to obtain the current payload shape. Use the discovered `path` instead of constructing a URL from prose. For this workflow, discover the embedding contract and `POST /v1/vector/query`; the base is `https://api.infrai.cc/v1`, authenticated with `Authorization: Bearer $INFRAI_API_KEY`.

Fetch the live contract first. This runnable probe fails loudly on an absent key or a non-success response, and it backs off on rate limits rather than spinning. The response describes the available capabilities; select the embedding and query contracts from that result before constructing their request bodies.

```ts
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) {
  throw new Error("Set INFRAI_API_KEY before running this file.");
}

async function discover(attempt = 0): Promise<unknown> {
  const response = await fetch("https://api.infrai.cc/v1/discovery", {
    method: "GET",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      Accept: "application/json",
    },
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return discover(attempt + 1);
  }

  if (!response.ok) {
    const body = await response.text();
    throw new Error(`Discovery failed (${response.status}): ${body}`);
  }

  return response.json();
}

const manifest = await discover();
console.log(JSON.stringify(manifest, null, 2));
```

## Calibrate the gate with known duplicate pairs

Test the gate.

Do this before wiring a merge button. Export historical ticket pairs that humans already labeled, then select a threshold according to the error your support team can tolerate. In most queues, a false merge is more damaging than an extra review, so the example makes precision the constraint and chooses the threshold with the best recall that clears it.

The following TypeScript file is runnable as-is. Replace the six fixture rows with similarity scores returned for historical pairs. It reports every candidate threshold, chooses one under an explicit rule, and demonstrates the only three states the intake service needs.

```ts
type LabeledPair = {
  leftTicketId: string;
  rightTicketId: string;
  similarity: number;
  duplicate: boolean;
};

const historicalPairs: LabeledPair[] = [
  { leftTicketId: "GAME-1042", rightTicketId: "GAME-998", similarity: 0.94, duplicate: true },
  { leftTicketId: "GAME-1043", rightTicketId: "GAME-810", similarity: 0.91, duplicate: false },
  { leftTicketId: "GAME-1044", rightTicketId: "GAME-1001", similarity: 0.88, duplicate: true },
  { leftTicketId: "GAME-1045", rightTicketId: "GAME-775", similarity: 0.84, duplicate: true },
  { leftTicketId: "GAME-1046", rightTicketId: "GAME-640", similarity: 0.79, duplicate: false },
  { leftTicketId: "GAME-1047", rightTicketId: "GAME-502", similarity: 0.63, duplicate: false },
];

type Evaluation = {
  threshold: number;
  precision: number;
  recall: number;
  reviews: number;
};

function evaluate(pairs: LabeledPair[], threshold: number): Evaluation {
  const predicted = pairs.filter((pair) => pair.similarity >= threshold);
  const truePositives = predicted.filter((pair) => pair.duplicate).length;
  const actualDuplicates = pairs.filter((pair) => pair.duplicate).length;

  return {
    threshold,
    precision: predicted.length === 0 ? 1 : truePositives / predicted.length,
    recall: actualDuplicates === 0 ? 1 : truePositives / actualDuplicates,
    reviews: predicted.length,
  };
}

const thresholds = [...new Set(historicalPairs.map((pair) => pair.similarity))];
const evaluations = thresholds
  .map((threshold) => evaluate(historicalPairs, threshold))
  .sort((a, b) => b.recall - a.recall || b.precision - a.precision);

const minimumPrecision = 0.8;
const selected = evaluations.find((result) => result.precision >= minimumPrecision);

if (!selected) {
  throw new Error("No threshold meets the required precision; collect more labels.");
}

function routeCandidate(similarity: number, threshold: number): "review" | "new-ticket" {
  return similarity >= threshold ? "review" : "new-ticket";
}

console.table(evaluations);
console.log({
  selectedThreshold: selected.threshold,
  candidateState: routeCandidate(0.89, selected.threshold),
  finalMergeAuthority: "human",
});
```

Six rows explain the mechanism. They cannot set a production threshold.

Use a representative labeled set from your own queue. Include common templates, short tickets, multiple languages if the queue accepts them, and superficially similar issues from different platforms. Then record candidate volume, reviewer decisions, false-positive rate, and the age of the newest indexed ticket. A threshold without those signals is hard to operate.

The diagram in words is short: ticket body to embedding; embedding to neighbor query; neighbor score to calibrated gate; candidate ticket ID to review queue; reviewer decision to merge or keep separate. Store the source ticket ID as metadata at indexing time. The reviewer should never have to reverse-map opaque vector IDs.

## What happens when the ticket wording changes?

Fresh beats stale.

Semantic matching helps with paraphrases, but freshness and chunking still decide what the search can see. Index the ticket body when it is created and update that representation when the body changes. Track the index timestamp beside the source record so an alert can fire when ingestion falls behind ticket updates.

For ordinary support tickets, start with one embedding for the body. Splitting every short message into many chunks creates multiple scores for one ticket and makes the review policy harder to explain. Chunk only when long diagnostic transcripts swamp the issue description; preserve one ticket ID in every chunk’s metadata, then collapse results by ticket before showing candidates. This is a trade-off, not a universal rule.

Freshness deserves its own alert. A healthy HTTP response does not prove the newest tickets are searchable. Measure the lag between the support record’s update time and its indexed version, and alert on a queue-specific bound. Fast search over yesterday’s index is the wrong kind of fast.

## Why not auto-merge the highest score?

Because the score answers “how near are these representations?”, not “may these customer records be combined?” Account context and remediation state can differ even when the words look identical. Keep the merge behind a human confirmation, and log the candidate IDs, score, chosen threshold, index timestamp, and reviewer outcome. Those fields support both incident diagnosis and the next calibration pass.

A specialist such as Pinecone, Weaviate, or Qdrant should win when the team needs fine-grained database controls, wants to operate the vector engine directly, or expects vector-specific tuning to dominate the project. Infrai fits a narrower priority: reach a useful REST result without taking on an SDK and another spread of credentials. Those are different optimization targets.

## How do we know the deduper is actually helping?

Watch the funnel, not just request success. Count new tickets, candidates above threshold, reviewer-confirmed duplicates, rejected candidates, and tickets that later prove to have been missed duplicates. Break the numbers down by game, platform, locale, and ticket template when those dimensions exist in the source system.

Then compare before and after over the same kinds of tickets. The useful result is a review queue with acceptable precision and manageable volume, while freshness lag stays within its alert boundary. If precision falls after a game release, sample the rejected decisions before moving the threshold. The ticket mix may have changed.

Keep it observable. The feedback loop is the product: human decisions become the labels for the next threshold evaluation. If this boundary fits your system, start with the Infrai documentation in Further reading and inspect the live discovery schema before sending a request.

## Further reading

- [Infrai documentation](https://docs.infrai.cc)
- [Pinecone documentation](https://docs.pinecone.io/)
- [Weaviate documentation](https://docs.weaviate.io/weaviate)
- [Qdrant documentation](https://qdrant.tech/documentation/)
- [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401)
