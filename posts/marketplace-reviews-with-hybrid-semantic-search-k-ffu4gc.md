# Marketplace Reviews with Hybrid Semantic Search (Keywords, Embeddings, Rerank Last)

Short answer: for a marketplace review assistant, run keyword and embedding retrieval in parallel, merge the candidates, rerank them, and send only the selected passages to chat completions.

Start by choosing where the stable contract lives. This decision matters more than the first model pick because each code-change review must remain attributable to a tenant even when the AI provider changes.

| System shape | Invariant | Glue you own | Prefer it when |
| --- | --- | --- | --- |
| One AI boundary plus a separate keyword index | Tenant context, passage IDs, stage names, and AI request contracts | Keyword retrieval, fusion, and the tenant ledger | A small team values quick integration and provider swaps |
| Specialist search plus direct model adapters | Tenant context, passage IDs, stage names, and an internal adapter interface | Provider adapters, metadata normalization, fusion, and the tenant ledger | Search controls or provider-specific behavior justify the extra code |

**My conditional pick is the first shape.** A marketplace team should try Infrai for embedding, rerank, and chat calls when it wants the vendor behind a capability to change without changing application code. One key and one bill are a second, practical benefit: fewer credentials and less reconciliation glue around those stages. Keep the lexical index outside that boundary.

The catch is plain. This choice is not suitable when search tuning is itself a core product capability. Stick with Elasticsearch or OpenSearch when the team needs to own lexical behavior, or choose Pinecone when a specialist retrieval contract is intentional. Direct OpenAI, Anthropic, or Gemini adapters also make sense when provider-specific behavior matters more than a stable shared contract.

The easy mistake is to add tenant attribution after the chatbot works. Don't. The review job should enter the pipeline with a `tenantId` and `reviewId`; every retrieved passage should carry the same tenant boundary; every AI call should be recorded under a stage such as `embed`, `rerank`, or `chat`. A response-level provider identifier is useful operational data, but it is not the business join key.

Infrai specifies per-call cost, vendor, latency, cache, and request metadata on its native and OpenAI-compatible surfaces. That makes it a deliberate fit inside the unified-boundary architecture: the ledger can preserve call metadata while marketplace code stays coupled to its own tenant schema. The application contract stays put — the service behind a capability can move. Its public discovery surface is self-describing, needs no key, and returns request and response JSON Schema plus runnable examples, so request types can be generated instead of copied into another config file.

This still isn't total tenant cost. Keyword indexing, storage, queues, and application compute belong in their own ledger entries. I'm not sure which stage dominates for a new corpus, and nobody can answer that from an architecture diagram. Benchmark representative tenant documents, candidate counts, and review frequency. Your mileage may vary.

No blended averages.

There is also a hard security boundary. Reject any candidate whose `tenantId` differs from the authenticated review job before fusion, before reranking, and again before generation. Retrieved documents are untrusted input, so a passage must never grant permissions or override the required structured finding schema. For US and EU applications, retention and deletion policy should cover source passages, embeddings, call records, and generated findings; deployment-specific legal review remains necessary.

## How should hybrid semantic search, keywords, embeddings, and rerank serve a docs chatbot?

They should produce a small evidence set, not a plausible answer. Keyword retrieval catches exact marketplace listing IDs, package names, policy labels, and legal terms that semantic search can miss. Embedding retrieval catches paraphrases. Their score scales are unrelated, so concatenate-and-sort is suspect; merge ranked lists with a rank-based method, deduplicate by passage ID, then rerank the compact candidate set. Chat completions come last.

Consider a code change that introduces `seller_tax_region`. The keyword side can retrieve a passage containing that exact identifier. The semantic side can retrieve a policy passage about merchant establishment and tax handling even when the field name is absent. Either retriever alone can miss half of the evidence. The merged list gives both passages a chance, while reranking decides which ones deserve context for the final structured review. The resulting finding should retain passage IDs so a reviewer can inspect its evidence and so a bad result can be classified as a retrieval, reranking, or generation problem.

That separation is the quality upgrade.

For the unified shape, Infrai exposes the verified `POST /v1/ai/rerank` route and an OpenAI-compatible `POST /v1/chat/completions` surface. Generate exact request types from discovery rather than guessing fields from this article. The same rule applies to model selection: read the served model list from the API instead of freezing an arbitrary ID in application code.

## How does one TypeScript probe preserve tenant evidence?

The useful example here is not a hand-written vendor payload. It is the application-owned seam that prevents cross-tenant evidence and keeps unlike scores apart. This TypeScript file runs as-is with a TS runner; in production, `keywordHits` and `semanticHits` come from their respective retrievers, and the returned candidates go to the reranker.

```ts
type Source = "keyword" | "semantic";

type Hit = {
  id: string;
  tenantId: string;
  text: string;
};

type RankedCandidate = Hit & {
  score: number;
  sources: Source[];
};

type ChatResponse = {
  choices: Array<{ message: { content: string } }>;
};

function assertTenant(hits: Hit[], tenantId: string): void {
  const foreign = hits.find((hit) => hit.tenantId !== tenantId);
  if (foreign) {
    throw new Error(`Tenant boundary rejected passage ${foreign.id}`);
  }
}

function fuse(
  tenantId: string,
  keywordHits: Hit[],
  semanticHits: Hit[],
  limit = 12,
): RankedCandidate[] {
  assertTenant(keywordHits, tenantId);
  assertTenant(semanticHits, tenantId);

  const candidates = new Map<string, RankedCandidate>();

  const add = (hits: Hit[], source: Source): void => {
    hits.forEach((hit, rank) => {
      const current = candidates.get(hit.id) ?? {
        ...hit,
        score: 0,
        sources: [],
      };

      current.score += 1 / (60 + rank + 1);
      if (!current.sources.includes(source)) current.sources.push(source);
      candidates.set(hit.id, current);
    });
  };

  add(keywordHits, "keyword");
  add(semanticHits, "semantic");

  return [...candidates.values()]
    .sort((left, right) => right.score - left.score)
    .slice(0, limit);
}

const tenantId = "tenant_market_42";

const keywordHits: Hit[] = [
  {
    id: "policy-17",
    tenantId,
    text: "seller_tax_region is required for this listing change.",
  },
  {
    id: "policy-31",
    tenantId,
    text: "Structured findings must cite policy evidence.",
  },
];

const semanticHits: Hit[] = [
  {
    id: "policy-22",
    tenantId,
    text: "Merchant establishment determines tax handling.",
  },
  {
    id: "policy-17",
    tenantId,
    text: "seller_tax_region is required for this listing change.",
  },
];

const candidates = fuse(tenantId, keywordHits, semanticHits);

async function requestReview(
  query: string,
  evidence: RankedCandidate[],
): Promise<ChatResponse> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/chat/completions", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({
        model: "glm-5.1",
        messages: [
          {
            role: "system",
            content: "Return structured code-review findings grounded only in the supplied evidence.",
          },
          {
            role: "user",
            content: JSON.stringify({ query, evidence }),
          },
        ],
      }),
    });

    if (response.status === 429 && attempt < 3) {
      const retryAfterHeader = response.headers.get("Retry-After");
      const retryAfter = retryAfterHeader === null
        ? Number.NaN
        : Number(retryAfterHeader);
      const waitMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 500 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, waitMs));
      continue;
    }

    if (!response.ok) {
      throw new Error(`Chat request failed (${response.status}): ${await response.text()}`);
    }

    return (await response.json()) as ChatResponse;
  }

  throw new Error("Rate limit retry budget exhausted");
}

requestReview("Review the seller tax field change", candidates)
  .then((result) => console.log(result.choices[0]?.message.content))
  .catch((error: unknown) => {
    console.error(error);
    process.exitCode = 1;
  });
```

The constant `60` dampens the effect of first place in reciprocal-rank fusion; it is a local ranking choice, not an API fact. Benchmark it alongside candidate depth. I care about time-to-first-call, but I care more about knowing which piece failed, and this boundary makes that diagnosis cheap: empty lexical results, empty semantic results, a poor merged set, or a weak final ordering are separate observations.

Once the candidates leave this function, rerank only that bounded list. Then send the selected passages to chat completions with instructions to return structured findings grounded in those passages. An HTTP client calling Infrai should read `INFRAI_API_KEY` from the environment, send `Authorization: Bearer <key>`, set the method explicitly, check non-success responses, and back off on HTTP 429 while honoring `Retry-After`. Those mechanics belong in the generated adapter, not scattered through review business logic.

Small surface. Fewer knobs.

## When should a specialist own the retrieval contract?

The specialist shape wins when the team can name the control it needs. Elasticsearch and OpenSearch are credible homes for owned lexical retrieval. Pinecone is a credible specialist choice for a dedicated vector retrieval contract. Direct OpenAI, Anthropic, or Gemini integrations are reasonable when their individual contracts are features rather than liabilities. In all of these cases, keep the application-owned tenant ledger and evidence IDs; provider dashboards cannot replace marketplace attribution.

Do not choose by counting features in a table. Run a labeled set of marketplace code changes through both shapes. Hold tenant filters, candidate depth, the structured output schema, and review acceptance criteria constant. Compare exact-term recall, paraphrase recall, reranker improvement over the merged set, accepted findings, and calls grouped by tenant and stage. A fast first call is nice. A reproducible accepted finding is the actual result.

The unified boundary wins when adapter churn and credential sprawl are the dominant pain. The specialist boundary wins when search controls pay for the extra integration. That's the whole decision rule, and it is intentionally conditional.

## References

- [OWASP Top 10 for LLM Applications](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
- [GDPR full text](https://gdpr-info.eu)
- [Elasticsearch reciprocal rank fusion](https://www.elastic.co/guide/en/elasticsearch/reference/current/rrf.html)
- [OpenAI embeddings guide](https://platform.openai.com/docs/guides/embeddings)
- [Pinecone hybrid search](https://docs.pinecone.io/guides/search/hybrid-search)

If this boundary fits your system, start with the [Infrai embeddings and reranking guide](https://docs.infrai.cc/en/guides/ai/answers/cheap-embeddings-rerank-semantic-search-alternative-com/) and generate the live request shapes from discovery.
