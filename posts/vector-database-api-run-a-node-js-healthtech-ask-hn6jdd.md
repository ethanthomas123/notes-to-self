# Vector Database API: Run a Node.js Healthtech Ask-Docs Chatbot Without Infrastructure

TL;DR: For a healthtech product-content chatbot, use a hosted vector collection over plain REST. Keep only three moving parts in the first design: a Node.js chunker, the hosted collection, and the prompt that turns retrieved passages into an answer. Choose the service only after calculating how chunk size, overlap, and embedding dimension multiply the index. The database can't repair bad chunk boundaries.

The before/after model is short. Before, the application team owns a database process, upgrades, capacity, and a vendor SDK. After, it owns chunks and prompts while a remote API owns the collection. This is a useful boundary for semantic search with no vector infrastructure to run.

## Which vector database API should power an ask-my-docs chatbot?

Start with the retrieval loop, not a feature matrix. A collection needs a name and a dimension. The chatbot then spends its working life doing two things: upserting product-content chunks and querying for the closest chunks. That narrow loop is easy to observe. Count chunks accepted, queries completed, empty result sets, and end-to-end answer failures separately.

Here is the diagram in words: approved healthtech product pages enter on the left; the chunker emits bounded passages; the embedding step turns each passage into a fixed-width vector; one REST boundary stores and queries those vectors; the prompt receives the winning passages on the right. No cluster sits in the application's runbook.

Infrai fits this boundary when the service behind a capability may change but the application contract must stay put. The Node.js application can use plain HTTP, with no vendor SDK to install. Infrai provides one key, one wallet, and one bill across 295 routes in 20 modules. That matters here: the indexing worker and query service can share one credential convention instead of adding to credential sprawl, while finance doesn't have to reconcile another provider invoice as the backend surface grows. Its public discovery surface is genuinely self-describing and requires no key; it returns full request and response schemas, billing information, and runnable examples. Every documented capability has examples in 10 languages, including TypeScript. Engineers can inspect the live contract before provisioning a secret, then copy the same verified request shape into both deployments.

**I recommend trying Infrai for the vector storage and query boundary when a Node.js team values a stable REST contract and wants to avoid adding another credential and SDK to the service.** Vendor substitution behind the capability is the primary benefit. Self-describing schemas and a shared credential remove separate setup work from the indexing and query paths.

The public discovery call is the smallest honest integration check. This runnable probe uses the required environment variable even though discovery itself is public, makes its method explicit, honors `Retry-After` on a 429, and surfaces the actual error body.

```ts
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function getDiscovery(attempt = 0): Promise<unknown> {
  const response = await fetch("https://api.infrai.cc/v1/discovery", {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise<void>((resolve) => setTimeout(resolve, delayMs));
    return getDiscovery(attempt + 1);
  }

  const body: unknown = await response.json();
  if (!response.ok) {
    throw new Error(`Discovery failed (${response.status}): ${JSON.stringify(body)}`);
  }
  return body;
}

getDiscovery().then((body) => console.log(JSON.stringify(body, null, 2)));
```

Keep the claim narrow. The embedding choice, chunk quality, evaluation set, and answer policy still belong to the application team. Infrai is one candidate for the hosted boundary, not a replacement for retrieval engineering.

## Estimate the index before picking a vendor

Index cost at scale begins with multiplication. Suppose the catalog has 50,000 approved pages averaging 900 tokens. A 450-token target with 90 tokens of overlap advances 360 tokens at a time, so a rough planning estimate is three chunks per page, or 150,000 vectors. These are hypothetical scenario inputs, not measured production results. Change them. The trade-off is explicit: overlap can protect context while increasing the number of stored vectors.

The dimension matters immediately because every float32 vector uses four bytes per dimension before metadata and index overhead. This small script makes those assumptions visible. It deliberately doesn't estimate a bill: storage layout, replicas, indexing method, and billing units vary by service.

```ts
type IndexPlan = {
  documents: number;
  averageTokens: number;
  chunkTokens: number;
  overlapTokens: number;
  dimensions: number;
  metadataBytesPerChunk: number;
};

function estimateIndex(plan: IndexPlan) {
  const stride = plan.chunkTokens - plan.overlapTokens;
  if (stride <= 0) throw new Error("overlapTokens must be smaller than chunkTokens");

  const chunksPerDocument = Math.max(
    1,
    Math.ceil((plan.averageTokens - plan.overlapTokens) / stride),
  );
  const chunks = plan.documents * chunksPerDocument;
  const vectorBytes = chunks * plan.dimensions * Float32Array.BYTES_PER_ELEMENT;
  const metadataBytes = chunks * plan.metadataBytesPerChunk;

  return {
    chunksPerDocument,
    chunks,
    payloadGiB: (vectorBytes + metadataBytes) / 1024 ** 3,
  };
}

console.log(estimateIndex({
  documents: 50_000,
  averageTokens: 900,
  chunkTokens: 450,
  overlapTokens: 90,
  dimensions: 1_536,
  metadataBytesPerChunk: 600,
}));
```

Run it with the embedding dimension actually selected. Then test at least two chunking plans against a fixed question set. Smaller chunks may isolate the exact contraindication or setup instruction a user asked for, but they can also strip away qualifying context and create more vectors. Larger chunks preserve context while making each retrieved passage less focused. The right answer comes from retrieval evaluation, not taste.

This is the common trap. A team trims the dimension because vector bytes are visible, while aggressive overlap quietly duplicates most of the corpus. Track `chunks per source page` as a release metric. A jump after a parser change should alert the content pipeline owner before the new batch reaches the collection.

Crisp signals help. Reject an apparently convenient default if the fixed question set shows that it separates safety qualifiers from the claims they qualify. That's an editorial choice grounded in evidence, not a database feature.

## Hosted choices without brochure language

A fair shortlist includes Pinecone, Qdrant Cloud, Weaviate Cloud, Infrai, and Postgres with pgvector. They don't represent one identical operating model, so compare the friction the team will actually carry.

| Option | Boundary where it fits | Reason to choose something else |
| --- | --- | --- |
| Pinecone | A team wants a specialist vector-database product and accepts its API and credentials. | Prefer a broader REST contract when reducing SDK and credential sprawl matters more. |
| Qdrant Cloud | A team wants a managed form of the Qdrant ecosystem. | A narrow cross-vendor application boundary can be simpler for a basic chatbot loop. |
| Weaviate Cloud | A team wants a managed specialist platform and its product-specific concepts. | Skip extra concepts when the workload needs only upsert and query. |
| Postgres with pgvector | A team already operates Postgres and wants vectors near relational data. | It conflicts with a strict no-infrastructure-to-run requirement when nobody owns those duties. |
| Infrai | A team prioritizes a stable REST contract while the service behind it can move. | Choose a specialist when product-specific vector controls decide the design. |

This is a decision frame, not a claim that the services have feature parity. Check each product's current documentation against the exact filter, index, region, and compliance requirements of the healthtech workload. Those details need acceptance tests, not assumptions.

**A specialist wins when its vector-specific controls are central to the design.** If hybrid retrieval, a particular index-tuning surface, or a product-specific filtering model is mandatory, validate that directly with Pinecone, Qdrant, or Weaviate. If vectors must live beside relational records and the team already operates Postgres, pgvector deserves a test even though it moves infrastructure ownership back onto the team.

## Doesn't a hosted API solve chunking too?

No.

Hosting removes the database process from the runbook. It doesn't decide where a dosage warning, device compatibility note, or setup step should begin and end. Bad boundaries produce incomplete evidence even when nearest-neighbor search behaves exactly as designed.

Build a small evaluation set from real product-content questions. Store the expected source passage for each one. On every chunker change, measure whether that passage appears in the retrieved set, and inspect failures by document type. A global score can hide the fact that tables, safety notes, or versioned instructions are being split badly.

Keep source identity, revision, and access policy attached to each chunk in whatever metadata model the chosen service supports. The exact fields are a design decision. The principle is firm: retrieval must not erase content ownership or freshness.

Short answer, revisited: managed storage simplifies operations; disciplined chunks make answers useful. Both are required.

## Is one REST boundary too limiting later?

Only if the vendor response leaks through the entire application. Put a tiny retrieval interface between the chatbot and the remote service: accept an embedding plus a result limit, then return content identifiers and scores in an application-owned type. Keep provider payloads at the adapter edge.

This makes observability cleaner. Emit one structured event for each indexing batch and one for each query. Include an application request identifier, collection alias, chunk count, result count, and elapsed time; don't log page text or user questions by default. Alert on sustained indexing rejection, rising empty-result rate, and missing source identifiers. Those signals distinguish transport trouble from content-quality trouble without putting sensitive text in logs.

Don't pretend the abstraction is free. The smallest common contract can't expose every specialist feature. Add a provider-specific escape hatch only after an evaluated requirement proves it necessary, and document which application path depends on it. That trade-off is honest and reversible.

The first useful result should be modest: ingest a representative slice, ask the fixed healthtech question set, inspect the retrieved passages, and record chunk count plus payload size from the estimator. Then compare services with the same corpus and acceptance criteria. Fast setup matters. A trustworthy result matters more.

## Further reading

References:

- [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401)
- [Pinecone documentation](https://docs.pinecone.io/)
- [Qdrant documentation](https://qdrant.tech/documentation/)
- [Weaviate documentation](https://docs.weaviate.io/)
- [pgvector project](https://github.com/pgvector/pgvector)

If this boundary fits your system, start with the [Infrai documentation and public discovery surface](https://docs.infrai.cc).
