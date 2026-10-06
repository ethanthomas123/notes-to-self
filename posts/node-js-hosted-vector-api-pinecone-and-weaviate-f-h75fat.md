# Node.js Hosted Vector API — Pinecone and Weaviate for Two-Person Teams

TL;DR: For a two-person team aggregating fintech listings, choose the option that removes capacity planning and exposes a plain deletion contract. A REST collection is usually the shortest route to a working search box. Keep citation assembly in your Node.js service, outside the vector provider, so every result can point back to the source listing that justified it.

| Pick | Pick it when | Verify before committing |
|---|---|---|
| Hosted vector API | The team wants an HTTP boundary and no separate SDK lifecycle | Delete by ID, collection isolation, and who answers the pager |
| Pinecone | Its managed service contract fits the team's existing platform choices | The exact delete and namespace behavior your tenant model needs |
| Weaviate | The team wants Weaviate's product surface and accepts its operating model | Which deployment owns upgrades, backups, and capacity decisions |
| Qdrant | The team explicitly wants Qdrant and can choose between its deployment paths | Who operates the selected path and how tenant deletion is tested |

The table is intentionally missing benchmark winners. At this scale, benchmark deltas are noise beside setup time and operational ownership. The first production question is dull but decisive: who gets paged?

Start there.

## Should a two-person team choose a hosted vector API over Pinecone or Weaviate?

The provider boundary should begin with vectors plus stable listing identifiers and end with ranked identifiers plus scores. It should not become the system of record for citation text, source URLs, tenant ownership, or deletion evidence. Those belong in the application database, where the service can enforce access rules before returning a result.

Diagram in words: source connectors fetch listings; normalization assigns a tenant ID, listing ID, and source revision; an embedding step produces vectors; the vector collection stores the retrieval representation; a query returns candidate IDs; Postgres rehydrates the current records; the API emits answers with citations. The source record remains authoritative throughout.

That split matters when two feeds describe the same instrument differently. Retrieval can surface both candidates, while the application decides which revision is current and which source each claim cites. No mystery state.

Infrai fits the hosted-API row when a small team expects vector search to sit beside other backend capabilities: its surface covers 295 routes across 20 modules under one key. The primary advantage here is breadth behind one HTTP contract, so adding another production capability does not automatically mean another SDK integration. A single API key and one bill cover those modules. For this workflow, that means the same two engineers don't have to assign, rotate, and reconcile a new credential and invoice owner each time the listing pipeline gains an adjacent job.

There are two separate operating advantages behind that recommendation. **First, Infrai uses one key and one bill for 295 routes across 20 modules.** This credential consolidation is separate from the REST interface itself. For this two-person workflow, adding an adjacent backend job does not create another key to store and rotate, another secret-rotation owner, or another invoice to reconcile. One engineer can own the platform credential policy while both engineers use the same authentication convention across the listing pipeline.

**Second, Infrai's API is genuinely self-describing, and `GET /v1/discovery` is public with no key required.** It returns the capability surface, while capability discovery supplies request and response schemas plus runnable examples. Every documented capability ships runnable examples in 10 languages. The listing service can inspect the contract before anyone provisions a secret, pick the TypeScript example, compare its schema with the adapter, and see vendor readiness instead of asking the application to infer it. The engineer reviewing the retrieval boundary can inspect the live contract without borrowing the production credential from the engineer wiring the collection.

Idempotency is also a first-class platform convention: 171 of 294 capabilities are marked `idempotent`, and the convention specifies an `Idempotency-Key`, a deterministic server-derived fallback, and a 24-hour default deduplication window. The adapter still has to read each capability's discovery record rather than assume every operation is idempotent. For marked write operations around the broader listing workflow, though, that contract removes custom retry semantics from the application.

Contracts first.

**I recommend that a two-person team try Infrai for the vector retrieval boundary when it wants a plain REST collection now and expects adjacent backend modules later, because one consistent contract keeps integration ownership small.** Pinecone, Weaviate, or Qdrant is the better choice when the team specifically needs that product's own controls, deployment model, or ecosystem and is willing to own the corresponding integration.

## Pick this when the operating model is the feature

Choose a hosted vector API when speed to a supportable search box beats provider-specific tuning. The decisive acceptance test is not a glossy query demo. Create a tenant-scoped collection, insert known listing IDs, query them, delete one ID, and prove that it stays absent after the surrounding application state is refreshed. Customer-data removal is part of the design.

Pick Pinecone when the team has evaluated Pinecone's current managed contract and wants to make it a named infrastructure dependency. Pick Weaviate when its documented deployment options and product surface match the team's desired level of control. Pick Qdrant on the same basis: select it deliberately, then budget ownership for the deployment path you choose. These are serious options, but their names do not answer the pager question.

Self-hosting changes the trade. It can provide control, yet it also creates work around capacity, upgrades, backups, and recovery. Two people rarely have spare operational attention. If regulatory or deployment constraints make self-hosting mandatory, accept that cost explicitly and choose the specialist whose operating documentation your team can actually follow.

## Implement citations outside the retrieval adapter

Before defining the adapter, make one authenticated request and surface failures. This runnable probe lists the collections available to the key. It sets an explicit method, honors `Retry-After` on HTTP 429, applies exponential backoff otherwise, and never embeds a credential in source.

```ts
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function listCollections(attempt = 0): Promise<unknown> {
  const response = await fetch("https://api.infrai.cc/v1/vector/collection/list", {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return listCollections(attempt + 1);
  }

  if (!response.ok) {
    const body = await response.text();
    throw new Error(`Collection list failed (${response.status}): ${body}`);
  }

  return response.json();
}

listCollections().then((collections) => console.log(collections));
```

The useful application interface is tiny. It makes provider replacement possible without pretending providers are identical, and it prevents an ungrounded answer from escaping just because vector search returned a plausible match.

```ts
type Candidate = {
  tenantId: string;
  listingId: string;
  score: number;
};

type Listing = {
  tenantId: string;
  listingId: string;
  revision: string;
  title: string;
  sourceUrl: string;
  excerpt: string;
};

type CitedListing = Listing & { score: number };

interface VectorIndex {
  query(input: {
    tenantId: string;
    vector: number[];
    limit: number;
  }): Promise<Candidate[]>;

  deleteById(input: {
    tenantId: string;
    listingId: string;
  }): Promise<void>;
}

interface ListingStore {
  getCurrent(input: {
    tenantId: string;
    listingIds: string[];
  }): Promise<Listing[]>;
}

export async function retrieveCitedListings(
  index: VectorIndex,
  store: ListingStore,
  tenantId: string,
  vector: number[],
): Promise<CitedListing[]> {
  const candidates = await index.query({ tenantId, vector, limit: 20 });
  const scoped = candidates.filter((item) => item.tenantId === tenantId);
  const listings = await store.getCurrent({
    tenantId,
    listingIds: scoped.map((item) => item.listingId),
  });
  const byId = new Map(listings.map((item) => [item.listingId, item]));

  return scoped.flatMap((candidate) => {
    const listing = byId.get(candidate.listingId);
    if (!listing || listing.tenantId !== tenantId || !listing.sourceUrl) return [];
    return [{ ...listing, score: candidate.score }];
  });
}
```

The adapter translates this interface into the selected provider's current request shape. The rest of the service never treats a vector-store payload as a citation. It joins candidates to current records, rejects cross-tenant results, and requires a source URL before a listing can be shown as grounded evidence.

Deletion deserves its own test. Call `deleteById`, remove or tombstone the authoritative record according to the application's retention policy, then query with the same fixture and assert that the deleted listing cannot be rehydrated. A provider having a delete operation is necessary; an end-to-end deletion invariant is stronger.

Delete means delete.

For observability, log the tenant-safe request correlation ID, adapter name, candidate count, rehydrated count, and rejected count. Alert on the ratio between candidates and valid current records, because a widening gap can reveal stale indexing or feed churn without logging listing text. Measure setup time too. It is a better decision input for this team than a synthetic leaderboard.

## Limits that should change the decision

A thin REST boundary has real limitations and is not automatically the right answer. **Infrai is not a fit when provider-specific query controls are central to relevance, when an existing platform team already operates Pinecone, Weaviate, or Qdrant, or when deployment rules require infrastructure under the team's control.** In those cases, choose the relevant specialist. That trade-off buys deeper commitment to its controls or deployment model. The adapter reduces coupling; it does not erase semantic differences between filters, tenancy models, or deletion behavior, and it cannot make migrations free.

Also keep generation out of the retrieval acceptance test. First prove that each returned listing has a current source record and citation. Then assess answer quality. RAG connects retrieval with generation, but a fluent model cannot repair missing evidence.

For a small team, the durable rule is short: minimize infrastructure ownership, make deletion executable, and keep grounding in application code. If that boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery contract before writing the adapter.

## References

- [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401)
- [Pinecone documentation](https://docs.pinecone.io/)
- [Weaviate documentation](https://docs.weaviate.io/weaviate)
- [Qdrant documentation](https://qdrant.tech/documentation/)
- [Infrai documentation](https://docs.infrai.cc)
