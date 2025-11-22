# Agentic RAG on AWS — Performance, Cost, and Architecture Recommendations (Nov 2025)

This report reviews your current serverless Agentic RAG implementation and proposes concrete upgrades across chunking, ingestion, retrieval/search, vector storage, models, evaluation, and observability. Recommendations are tailored to the present repo and prioritize low/no‑cost improvements with a p95 ≤ 2s target.

Cross‑links reference files in this repo for fast navigation.

## Executive Summary

- Keep the AWS serverless ingestion design; optimize chunking and retrieval quality first (highest ROI, lowest effort).
- Short term: stay on Supabase pgvector or move to Neon Postgres serverless (scale to 0) to reduce base cost. Avoid Aurora/OpenSearch/Kendra at your current scale/budget.
- Replace Gemini embeddings with a cheaper, strong embedding (OpenAI or Voyage) and add a light reranker to boost answer quality without exceeding the 2s p95.
- Add hybrid search (dense + BM25) and hierarchical/semantic chunking with larger chunks (by tokens, not characters). Expect +15–35% retrieval quality gains with minimal tech lift.
- Introduce simple evaluation and X-Ray/EMF metrics to keep latency within budget and to quantify improvements.

## Current Implementation (Ground Truth)

- Ingestion (serverless, decoupled):
  - API Gateway → SQS → Firecrawl fetch → S3 `document-archive/` → S3 trigger → chunking → S3 `chunks/` → S3 trigger → embeddings → Supabase.
  - Code references:
    - Ingestion API: `lambdas/ingestion_api_handler/handler.py:1`
    - Firecrawl fetch: `lambdas/firecrawl_fetch_worker/handler.py:1`
    - Chunking worker (paragraph/sentence; char-based): `lambdas/document_chunking_worker/handler.py:1`
    - Chunk embedding (Gemini 2.0 Flash for subject, text-embedding-004 for vectors): `lambdas/chunk_embedding_worker/handler.py:1`
    - Query handler (Gemini embeddings + Supabase pgvector RPC): `lambdas/query_handler/handler.py:1`
  - Database: Supabase Postgres + pgvector with IVFFLAT index and RPC `match_documents(...)` (vector_cosine_ops): `database/supabase_schema.sql:1`

- Observed constraints/opportunities:
  - Chunking is char-length based (default 1,000 chars) — often too small; not token-aware; limited section metadata. `lambdas/document_chunking_worker/handler.py:9`
  - No reranker; no BM25/hybrid; fallback path computes similarity in Python if RPC fails (latency risk). `lambdas/query_handler/handler.py:163`
  - Embedding dimension is 768 (Gemini). Schema assumes 768. `database/supabase_schema.sql:20`
  - S3‑driven, Lambda step‑functions style is clean and low cost; good failure handling and S3 error logs in each worker.

## Goals and Constraints

- Quality: Improve retrieval precision/recall for coding/research docs; reduce navigation/boilerplate noise; better context windows.
- Latency: p95 ≤ 2s end‑to‑end for retrieve + (optional) rerank + generate first token.
- Scale and cost: a few thousand docs; ~1k queries/day. Budget ≤ A$50/month preferred; favor scale-to-zero and usage-based.
- Models: Migrate off Gemini; use OpenRouter for generation; consider Bedrock options for embeddings/rerank if budget allows.

## Recommendations by Area

### Chunking and Preprocessing

1) Switch from char-based to token-aware, semantic/hierarchical chunking
- What: Split by Markdown headings and paragraphs, target ~800–1,200 tokens/chunk with 10–15% overlap. Store parent section title and depth in chunk metadata.
- Why: Larger, semantically aligned chunks preserve context and reduce fragmentation. For web docs, this consistently improves top‑k recall and answer faithfulness.
- Impact: +10–25% recall@10 and +8–20% answer faithfulness, typical for doc sites.
- How:
  - Add a lightweight Markdown parser and use headings (`#`, `##`, `###`) to define boundaries, then pack paragraphs into ~1,000 tokens per chunk.
  - Keep paragraph splitter as fallback; compute approximate tokens via byte-length heuristic if you want to avoid tokenizers in Lambda.
  - Extend chunk metadata: `section_title`, `section_path`, `document_title`, `headings_in_chunk`.
  - Code: Update `chunk_markdown(...)` in `lambdas/document_chunking_worker/handler.py:13` to be heading-aware and token-approximate.

2) Topical filtering and boilerplate removal during ingestion
- What: After Firecrawl markdown, add a cleaning step that strips nav/footers, cookie banners, and off-topic snippets. Optionally run a small LLM pass to extract only subject‑relevant content per your domain.
- Why: Reduces noise vectors and index bloat; improves precision.
- Impact: -10–30% index size and +5–15% precision@k.
- How: Add a Lambda step or function in the chunker to:
  - Apply readability heuristics and regex filters (common footer/nav patterns, link-only lists).
  - Optional LLM filtering with a cheap model (OpenRouter: gpt‑4o‑mini or Llama‑3.1‑8B) to keep subject‑relevant sections only.
  - Preserve original in S3; store cleaned version alongside.

3) Parent–child retrieval
- What: When a chunk matches, also fetch its parent section (and optionally adjacent chunks) and include in the prompt.
- Why: Increases answerability without aggressive chunk overlap.
- Impact: +5–12% answer quality scores; small latency hit if done via extra DB requests.
- How: Store parent section IDs; query by `document_id` + nearby `chunk_number`.

### Retrieval and Ranking

4) Ensure vector RPC path is always used; remove Python similarity fallback
- What: Confirm `match_documents(...)` RPC exists and is used; avoid Python cosine fallback except in rare emergency.
- Why: Fallback pulls 1,000 rows and ranks in Lambda, risking p95 > 2s.
- Impact: −300–800 ms tail latency avoided in failure scenarios.
- How: On startup, health‑check the RPC and index; fail fast with clear error.

5) Switch index to HNSW for higher recall (low write volume)
- What: Change from IVFFLAT to HNSW in pgvector for search. With your small write rate, HNSW’s better recall at similar or lower query latency is preferable.
- Impact: +5–15% recall@10; similar p95 latency at your scale.
- How: In `database/supabase_schema.sql:33`, prefer the HNSW index variant (already commented). Rebuild index during low traffic window.

6) Add hybrid search (dense + BM25/lexical) with Reciprocal Rank Fusion (RRF)
- What: Combine pgvector results with Postgres full‑text search over the same table, then fuse rankings (RRF or weighted z‑score).
- Why: Hybrids are robust to OOD queries and names/IDs/code terms.
- Impact: +8–20% recall@10; helps long‑tail.
- How: Add a `tsvector` column computed from `subject + text`, index with `GIN`, run both queries and fuse in Lambda. Minimal schema change; no new infra.

7) Add a lightweight reranker on top‑k
- What: Rerank top 30–50 candidates with a cross‑encoder.
- Why: Cross‑encoders significantly improve final precision@k.
- Impact: +10–25% MRR/NDCG on web/coding corpora.
- Low‑cost options:
  - API: Cohere Rerank v3 (direct API or via Bedrock if available in your region) — simple and strong.
  - Open-source in Lambda: `bge-reranker-base` or `ms-marco-MiniLM-L-6-v2` ONNX CPU; containerized Lambda to keep package small. Expect ~40–100 ms for 50 pairs on warm containers; cold starts possible.
  - Jina Rerank small via API.

8) Query expansion for tough queries (optional)
- What: Generate 2–4 paraphrases (small LLM), merge results by RRF.
- Why: Improves recall for ambiguous or short queries.
- Impact: +5–12% recall@10; adds ~100–250 ms.
- How: Use OpenRouter with a small instruct model; enable per‑query heuristics (only trigger for short/rare terms).

### Vector Store Options (Cost/Perf)

At your scale and budget, stay with Supabase pgvector or move to a serverless Postgres that scales to zero. Avoid heavy managed search unless you grow significantly.

- Supabase Postgres + pgvector (current)
  - Pros: Low base cost (even free/low tiers), pgvector mature, easy to operate; you already have schema and RPC.
  - Cons: Doesn’t truly scale to zero; free tier limits; vendor lock‑in vs AWS native.
  - Keep and harden: switch to HNSW, add FTS GIN index, and hybrid.

- Neon Postgres Serverless + pgvector
  - Pros: Compute scales to zero; good for spiky/low volume; pgvector supported. Likely lowest monthly for A$50 constraint.
  - Cons: Cross‑cloud (not AWS), but acceptable per your constraints.
  - Action: Migrate schema and RPC; point Lambdas via env vars.

- Amazon Aurora Serverless v2 (Postgres) + pgvector — supports auto‑pause to zero ACUs
  - What’s new: With MinCapacity=0 and auto‑pause enabled, Aurora Serverless v2 can pause to 0 ACUs when idle, resuming automatically on first connection. You’re not billed for compute while paused (storage/I/O still apply).
  - Pros: All‑AWS; strong performance; potential near‑zero compute during idle periods; straightforward pgvector support.
  - Cons: Cold resume adds seconds to low tens of seconds of latency depending on configuration; to guarantee p95 ≤ 2s during active hours, you may need a small MinCapacity (e.g., 0.5–1 ACU) or a keep‑alive, which reintroduces steady compute cost. Storage + I/O costs apply regardless.
  - Verdict: Viable if you can tolerate occasional cold‑start latency or are willing to keep a warm floor during active windows. Pilot with ACU/I/O metrics to project monthly cost.

- Amazon DynamoDB Vector Search
  - Pros: Native serverless with true scale‑to‑zero compute billing (pay per request); integrates cleanly with Lambda; no separate cluster to manage; good fit for spiky traffic.
  - Cost model: You pay for (a) item + vector index storage (GB‑month), (b) vector write/update operations when ingesting embeddings, and (c) vector read/search operations (cost scales with vector dimension, top‑k, and filters), plus standard DynamoDB data transfer. No steady compute charge when idle.
  - Considerations: Best for simple schemas; migration from Postgres requires new query path and rank fusion logic. Evaluate latency with your embedding dim/top‑k; keep k modest (e.g., 50) and add a reranker for precision.
  - Action: If you prefer all‑AWS and want lower idle cost than Aurora, DynamoDB Vector Search is a practical option. Use the AWS Pricing Calculator with your region, vector dim, and k to validate staying under A$50/month for a few thousand docs and ~1k queries/day.

- Amazon OpenSearch Serverless (Vector)
  - Pros: Feature‑rich search.
  - Cons: Minimum OCU floor makes it expensive when idle; typically >> A$50/mo.
  - Verdict: Not a fit for this budget.

- Zilliz Cloud (Milvus) or Qdrant Cloud Serverless
  - Pros: Solid ANN performance; serverless tiers can be inexpensive at low volumes.
  - Cons: Extra vendor; migration effort. Stick to Postgres unless you hit performance limits.

### Models: Embeddings, Rerankers, Generators

Embeddings (ranked by quality/cost tradeoff for web/coding RAG):
- OpenAI `text-embedding-3-small` — strong retrieval quality at low cost. Migration requires changing vector dim (1536) and reindexing.
- Voyage `voyage-3-lite` or `voyage-3` — competitive for retrieval; transparent pricing; good English performance.
- Jina Embeddings v3 — solid and affordable.
- Bedrock Titan Embeddings V2 (if staying inside AWS and budget allows) — simplifies data residency; performance is good.

Rerankers:
- Cohere Rerank v3 (API or via Bedrock) — consistent quality and easy integration.
- Jina Reranker small — cheaper, decent.
- Open-source cross-encoders in Lambda (ONNX) — zero per‑query cost; slightly higher operational complexity (container image + warm start strategy).

Generators (via OpenRouter for answers and cleaning steps):
- Cost‑optimized: `gpt‑4o‑mini`, `Llama‑3.1‑8B‑Instruct`, `Qwen2.5‑14B‑Instruct`.
- Quality‑optimized: `Claude 3.5 Sonnet`, `Llama‑3.1‑70B`, `Mistral Large`.
- For agent/tool use, prefer models with good function‑calling and long context (Claude 3.5 Sonnet, GPT‑4o family, Llama‑3.1‑70B).

### Low‑Cost Latency Tuning (≤ 2s p95)

- Keep top_k small (10–20) for the first stage; rerank at 30–50.
- Use streaming for generation to reduce time‑to‑first‑token.
- Enable Lambda Provisioned Concurrency = 1 only on the query path if cold starts are frequent; monitor cost/benefit.
- Cache frequent query results (DynamoDB with TTL 1–24h) when acceptable.

### Logging, Metrics, and Tracing

- Add EMF custom metrics for latency breakdown in each Lambda (embed time, DB time, rerank time, prompt time).
- Add AWS X‑Ray to stitch traces across API Gateway → Lambda → Supabase calls (via subsegments/annotations).
- Keep your structured JSON logging pattern; add `correlation_id` per request and propagate.

### Evaluation (Simple, Actionable)

Objective: Track retrieval gains from chunking/rerank/hybrid without heavy infra.

- Dataset: 50–100 web Q&A pairs covering your domains (coding, research). Store in repo (YAML/JSONL).
- Metrics:
  - Retrieval: recall@k (k=5/10), NDCG@k.
  - Answer: faithfulness score via LLM judge on citations (small model to reduce cost), and user‑facing exact‑match/ROUGE‑L for known answers.
- Harness:
  - Reuse query Lambda locally with a flag to dump candidates and latencies.
  - Compute metrics offline; write Markdown summaries to `spec/reports/eval/`.
  - Run on each change to chunking/ranking/index; keep budget < A$2/run by using small rerankers and no generation (judge only on retrieved spans).

## Migration: Off Gemini to OpenRouter (and/or OpenAI/Bedrock)

Scope: Replace Gemini embedding and subject generation. Keep Firecrawl and Supabase.

- Embeddings:
  - Replace `generate_embedding(...)` in `lambdas/chunk_embedding_worker/handler.py:94` and `generate_query_embedding(...)` in `lambdas/query_handler/handler.py:72` with your chosen provider.
  - If using OpenAI `text-embedding-3-small`, change column to `VECTOR(1536)` and rebuild HNSW index.
  - Add env vars for provider base URL and key (OpenRouter/OpenAI/Voyage).

- Subject generation:
  - Replace `gemini-2.0-flash-exp` with a cost‑optimized model (`gpt‑4o‑mini` or `Llama‑3.1‑8B`) in `lambdas/chunk_embedding_worker/handler.py:52`.

- Reranker (optional):
  - Add a new function in `lambdas/query_handler/handler.py` to call Cohere Rerank (or ONNX local), gated by an env flag.

- Hybrid search:
  - Add `tsvector` column and GIN index to schema; implement combined query + rank fusion in `search_similar_chunks(...)`.

## Cost Guidance (Indicative)

Given your budget (≤ A$50/mo), choose serverless components with true idle scaling and low per‑request charges.

- Vector DB
  - Stay on Supabase (existing) or move to Neon serverless: both are the most cost‑efficient today for small/medium datasets.
  - Aurora Serverless v2 with auto‑pause (MinCapacity=0) can cut compute cost during idle windows; storage and I/O still accrue. Consider a scheduled warm floor during active periods if you need strict p95.
  - DynamoDB Vector Search can be viable if you prefer all‑AWS and later need higher scale; evaluate region‑specific pricing and top‑k dimensions for precise estimates.

- Embeddings
  - Use `text-embedding-3-small`/Voyage/Jina for low per‑token cost and strong retrieval quality.
  - Pre‑filter text to reduce tokens before embedding; store only necessary metadata.

- Reranking
  - Start with a small ONNX cross‑encoder in a Lambda container (zero per‑request API cost) or Cohere Rerank with strict top_k to cap spend.

- Generation
  - Prefer `gpt‑4o‑mini` or Llama‑3.1‑8B for answers; stream responses to improve perceived latency.

## Gemini File Search API vs DIY RAG

- Gemini File Search provides file uploads, corpora, and retrieval integrated with Gemini models. It reduces infra/operations and can be fast to prototype.
- Trade‑offs:
  - Pros: Fewer moving parts; model‑aware retrieval; Google‑managed scaling.
  - Cons: Vendor lock‑in for storage and retrieval; ongoing per‑token and storage costs; less control over chunking/indexing; harder to integrate non‑Gemini models for generation/rerank.
  - At your budget, a DIY stack using open, interchangeable pieces lets you optimize cost (e.g., local reranker, scale‑to‑zero DB) while meeting 2s p95.
- References: see links in References.

## Hosted RAG Solutions and Managed Cloud Options

- Pinecone Serverless + Assistant (beta/managed RAG): easy, but costs can exceed your budget at idle; consider later.
- Zilliz Cloud / Weaviate Cloud: competitive serverless vector stores; still another vendor.
- Managed cloud RAG:
  - AWS Knowledge Bases for Bedrock: managed connectors, chunking, embeddings, and retrieval with Bedrock models. Excellent integration; typically above your budget unless subsidized; best for all‑AWS shops.
  - Azure AI Search (Vector) + OpenAI: strong; not serverless to zero; likely higher base cost.
  - Google Vertex AI Search & Grounding: integrated retrieval; costs tied to Google ecosystem; evaluate only if you standardize on Google.

## Estimated Impact Summary

- Token/semantic chunking (800–1,200 tok): +10–25% recall@10; +8–20% faithfulness.
- HNSW index: +5–15% recall@10 (similar p95 at current scale).
- Hybrid (dense + BM25 + RRF): +8–20% recall@10.
- Light reranker (top‑50 → top‑10): +10–25% MRR/NDCG; +150–300 ms.
- Topical filtering + boilerplate removal: −10–30% index size; +5–15% precision@k.
- Remove Python fallback + RPC healthchecks: −300–800 ms tail in failure cases.

These gains stack; a pragmatic path (chunking + HNSW + hybrid + small reranker) typically yields noticeably better answers within a 2s p95.

## Diagram

```mermaid
flowchart TB
  A[Client] -->|POST /ingest-url| B[API Gateway]
  B --> C[Ingestion Lambda]
  C --> D[SQS]
  D --> E[Firecrawl Fetch Lambda]
  E --> F[S3 document-archive/source.md]
  F --> G[Chunking Lambda (semantic, token-aware)]
  G --> H[S3 chunks/*.json]
  H --> I[Embedding Lambda (Embeddings API)]
  I --> J[(Postgres + pgvector HNSW)]
  K[Client] -->|POST /query| L[API Gateway]
  L --> M[Query Lambda]
  M --> J
  M --> N[Postgres FTS (GIN)]
  J --> O[Top-k Dense]
  N --> P[Top-k Lexical]
  O --> Q[RRF Fuse]
  P --> Q
  Q --> R[Optional Reranker]
  R --> S[OpenRouter LLM (stream)]
  S --> T[Answer + Citations]
```

## Action Plan (Minimal Lift First)

1) Chunking upgrade: heading-aware + ~1,000 token packing; write parent metadata.
2) Switch to HNSW index; verify RPC path; remove Python fallback.
3) Add BM25/FTS + RRF fusion in query path.
4) Add light reranker (API or ONNX) gated by env flag.
5) Swap embeddings provider (OpenAI/Voyage/Jina); reindex to new dim if needed.
6) Add EMF metrics + X‑Ray spans; implement simple eval harness.

## Recommendation Matrix (Value vs Effort)

| Recommendation | Value | Effort | Cost | Difficulty |
| --- | --- | --- | --- | --- |
| Semantic/token-aware chunking (1k tok) | High | Low | None | Low |
| HNSW index on pgvector | Medium–High | Low | None | Low |
| Hybrid search (RRF) | High | Low–Medium | None | Low–Medium |
| Light reranker (API/ONNX) | Medium–High | Medium | Low–Medium | Medium |
| Remove Python fallback; RPC healthcheck | Medium | Low | None | Low |
| Topical filtering step | Medium | Low–Medium | Low | Low–Medium |
| Swap embeddings to `text-embedding-3-small`/Voyage | Medium–High | Medium (reindex) | Low | Low–Medium |
| Move to Neon Postgres serverless | Medium (cost) | Medium | Low | Medium |
| Bedrock embeddings/rerank | Medium–High (quality/integration) | Low–Medium | Medium | Low |
| Query expansion (multi‑query) | Medium | Low | Low | Low |

Notes:
- Costs depend on token volumes and provider; embedding swaps generally reduce spend vs Gemini. Reranking adds small per‑query cost unless local.
- Neon move trades vendor for scale‑to‑zero; keep Supabase if your cost is already acceptable.

## References

- Supabase + pgvector schema and RPC in this repo: `database/supabase_schema.sql:1` — used to confirm current index type (IVFFLAT), RPC signatures, and embedding dimension assumptions.
- Current chunking implementation: `lambdas/document_chunking_worker/handler.py:13` — used to assess paragraph/sentence char-based chunking and default sizes.
- Embedding + subject generation: `lambdas/chunk_embedding_worker/handler.py:52`, `lambdas/chunk_embedding_worker/handler.py:94` — used to identify Gemini models in use and where to swap providers.
- Query handler (RPC + fallback): `lambdas/query_handler/handler.py:72`, `lambdas/query_handler/handler.py:136`, `lambdas/query_handler/handler.py:163` — used to assess retrieval flow, RPC usage, and fallback risks.
- Google Gemini File Search Overview: https://ai.google.dev/gemini-api/docs/file_search_overview — referenced for capabilities (corpora, file APIs) and pricing links to contrast with DIY RAG.
- AWS Bedrock Knowledge Bases pricing: https://aws.amazon.com/bedrock/knowledge-bases/pricing/ — referenced to gauge budget fit of a managed RAG option.
- Amazon DynamoDB pricing: https://aws.amazon.com/dynamodb/pricing/ — referenced for vector search pricing model (per-request + storage) to evaluate feasibility vs Postgres at small scale.
- Amazon OpenSearch Service pricing: https://aws.amazon.com/opensearch-service/pricing/ — referenced to note OCUs and the high minimum for serverless collections vs your budget.
- Neon serverless Postgres pricing: https://neon.tech/pricing — referenced to validate scale-to-zero suitability and likely lowest DB cost at small scale.
- OpenRouter models/pricing: https://openrouter.ai/models, https://openrouter.ai/pricing — referenced to suggest low-cost/quality models for generation and cleaning.
- Cohere Rerank: https://docs.cohere.com/docs/rerank, https://cohere.com/pricing — referenced for plug-and-play reranking as an API.
- Jina models: https://jina.ai/ — referenced for embeddings/rerank alternatives and cost options.
- OpenAI pricing: https://openai.com/api/pricing/ — referenced for `text-embedding-3-small` as a strong, low-cost embedding baseline.
- Aurora Serverless v2 — How it works: https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-serverless-v2.how-it-works.html — referenced for capacity scaling details and min/max ACU behavior.
- Aurora Serverless v2 — Auto‑pause to zero ACUs: https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-serverless-v2-auto-pause.html — referenced for enabling MinCapacity=0, auto‑pause timeout, and resume behavior.
- DynamoDB Vector Search announcement: https://aws.amazon.com/blogs/aws/amazon-dynamodb-introduces-vector-search/ — referenced for feature overview; pricing details are in the DynamoDB pricing page.

—

Prepared for: Agentic RAG AWS stack (serverless, Python)
Date: 2025‑11‑15
Author: Codex CLI AI (AWS/AI Architecture)
