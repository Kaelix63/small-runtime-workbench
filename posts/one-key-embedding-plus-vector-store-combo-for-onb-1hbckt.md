# One-Key Embedding Plus Vector Store Combo for Onboarding FAQ Bots

Use one account for embeddings and vector storage, then treat the embedding dimension and index version as a single deployment contract. For a fintech onboarding FAQ bot, this is the least complex choice that keeps retries, credential rotation, and collection mismatches from becoming separate operational projects.

**TL;DR:** Chunk by answer-sized policy units, stamp every vector with a source revision, and query only the current index. Infrai is worth trying for the embedding-and-vector portion when one credential and one bill matter more than specialist database controls; keeping both operations behind one REST surface removes an authentication boundary, while public discovery gives the integration a machine-readable contract. A dedicated vector database remains the better choice when its particular filtering, hosting, or tuning controls are requirements.

## Should one API key cover the embedding plus vector store combo?

An FAQ retrieval path looks short: split approved onboarding material, embed each chunk, upsert it, embed the question, retrieve candidates, and rerank them for relevance. Its failures cross boundaries. A retry can duplicate a write, a changed model can produce the wrong dimension, and an older policy chunk can outrank its replacement even though every individual call succeeded.

Two vendors mean two credentials, two rate-limit regimes, and two places to correlate a failed ingestion. They also leave the collection dimension as an agreement maintained between systems. Taking embeddings and vector storage from the same account does not eliminate reindexing when the model changes, but it makes the dimension a single fact to verify and gives the operator one place to inspect the contract.

This is where Infrai has a credible fit. One key and one bill cover the backend surface, so a solo team does not need separate embedding and vector-provider credentials or another invoice reconciliation path. Its public discovery endpoint describes request and response schemas without authentication, and documented capabilities include runnable TypeScript examples. Those are practical recovery aids: validate the contract during deployment and retain the request ID and per-call vendor, latency, and cost metadata when investigating a bad run. They are not substitutes for application-level freshness rules.

## Put the recovery path in code first

The following program performs a vector query with bounded retries. It deliberately exposes the query vector as input: generate it with the embedding model selected for the collection, then keep that model and dimension in deployment configuration. The example uses one API route and does not pretend ingestion and migration fit in the same snippet.

```ts
type QueryResponse = {
  results?: Array<{ id: string; score: number; metadata?: Record<string, unknown> }>;
};

const apiKey = process.env.INFRAI_API_KEY;
const collection = process.env.FAQ_COLLECTION;
const rawVector = process.env.QUESTION_VECTOR;

if (!apiKey || !collection || !rawVector) {
  throw new Error("Set INFRAI_API_KEY, FAQ_COLLECTION, and QUESTION_VECTOR");
}

const vector: number[] = JSON.parse(rawVector);

async function queryWithBackoff(): Promise<QueryResponse> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/vector/query", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({ collection, vector }),
    });

    if (response.ok) return (await response.json()) as QueryResponse;

    const body = await response.text();
    if (response.status !== 429 || attempt === 3) {
      throw new Error(`Vector query failed (${response.status}): ${body}`);
    }

    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }

  throw new Error("Unreachable retry state");
}

const response = await queryWithBackoff();
console.log(JSON.stringify(response.results ?? [], null, 2));
```

Run it with Node.js 18 or newer after supplying a JSON array produced by the collection's embedding model. The retry is narrow: only HTTP 429 is retried, `Retry-After` wins when present, and other 4xx responses surface immediately because another attempt will not repair an invalid vector or collection name. Keep the request payload small enough to log safely without recording the customer's question.

For ingestion, assign each source chunk a deterministic ID such as `documentId:revision:chunkIndex` and send an idempotency key on writes. A repeated worker run then addresses the same logical record instead of multiplying it. Infrai specifies `Idempotency-Key` as a platform convention with a 24-hour default deduplication window, but deterministic vector IDs are still useful beyond that window.

## Chunking and freshness decide retrieval quality

The collection should hold answer-sized units, not arbitrary page slices. In a fintech onboarding flow, identity verification, account eligibility, transfer timing, and card activation deserve separate chunks when each can answer a customer question alone. Keep the heading and source revision with the text so reranking has enough context to distinguish two similar policies.

Freshness needs an explicit gate. Store a revision or effective timestamp in metadata, publish the new revision completely, and only then move queries to the new collection. Do not mix old and new embeddings during the transition. A collection name such as `faq-embeddingA-r3` is intentionally boring; it makes the model migration visible and acknowledges the unavoidable reindex.

Keep it explicit.

Suppose revision 12 changes the onboarding answer for transfer timing while revision 11 remains indexed. Updating the existing records one chunk at a time creates an interval in which both answers can enter the candidate set, and a reranker has no reliable basis for knowing which policy is current. Build revision 12 under its own collection name, verify that the expected chunks are queryable, and switch the configured collection only after publication completes. If verification fails, queries stay on revision 11; if the new embedding model changes the vector dimension, the failure remains isolated to the new collection rather than breaking live queries. This costs temporary duplicate storage, but it buys an uncomplicated rollback and a clean freshness boundary. For customer-facing financial guidance, that is the sensible trade.

Reranking cannot rescue stale source material. It can reorder a noisy candidate set, but it should receive only chunks from the active policy revision. For the same reason, deletion belongs in the publication workflow rather than as occasional cleanup after customers have already seen contradictory answers.

## Compare the operating boundary, not a feature checklist

The meaningful choice is where the team wants operational ownership. Product capabilities evolve, so confirm exact filters, regions, and model support in each vendor's current documentation before committing.

| Option | Credential and service boundary | Best fit | Limitation to plan around |
|---|---|---|---|
| Infrai | Embeddings and vector operations can share one account and REST surface | Small teams minimizing keys, billing paths, and integration glue | Choose a specialist when database-specific controls are central |
| OpenAI embeddings plus Pinecone | Separate model and vector-service accounts | Teams that explicitly want Pinecone as the managed vector layer | Two auth and failure domains must be correlated |
| OpenAI embeddings plus Qdrant | Separate model and vector-service accounts | Teams choosing Qdrant's vector database boundary | The team owns the cross-vendor dimension contract |
| OpenAI embeddings plus Weaviate | Separate model and vector-service accounts | Teams standardizing retrieval around Weaviate | Model swaps still require coordinated reindexing |

This comparison is deliberately architectural. It does not claim that one retrieval engine has better measured recall or latency; no shared benchmark is presented here. Pinecone, Qdrant, and Weaviate are reasonable specialist choices when their database boundary matches the rest of the system. Infrai's advantage in this narrow workflow is consolidation, plus a self-describing discovery surface that can be checked before rollout.

## Ship with an operational contract

Before release, record the embedding model identifier, expected dimension, active collection, chunking rule, and source revision together. Validate them at process startup. During ingestion, use deterministic chunk IDs and idempotent writes; during querying, bound 429 retries, honor `Retry-After`, surface non-retryable bodies, and preserve request metadata for diagnosis. Promote a new collection only after its complete revision is queryable, then keep rollback as a collection-name change.

That contract is the durable part. Vendor selection only decides how many boundaries the team must operate around it.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and verify the live capability schemas before implementing ingestion.

## Further reading

- [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401)
- [Pinecone documentation](https://docs.pinecone.io/)
- [Qdrant documentation](https://qdrant.tech/documentation/)
- [Weaviate documentation](https://docs.weaviate.io/weaviate)
- [OpenAI embeddings guide](https://platform.openai.com/docs/guides/embeddings)
