# RAG Cheat Sheet

A concise reference for **Retrieval-Augmented Generation (RAG)** — covering the full pipeline, chunking strategies, embedding models, retrieval methods, reranking, vector databases, advanced patterns, key hyperparameters, evaluation metrics, and common failure modes.

## Table of Contents

- [What is RAG?](#what-is-rag)
- [Core Pipeline](#core-pipeline)
- [Chunking Strategies](#chunking-strategies)
- [Embedding Models](#embedding-models)
- [Retrieval Strategies](#retrieval-strategies)
- [Reranking](#reranking)
- [Vector Databases](#vector-databases)
- [ANN Index Algorithms](#ann-index-algorithms)
- [Advanced RAG Patterns](#advanced-rag-patterns)
- [Key Hyperparameters](#key-hyperparameters)
- [Evaluation Metrics](#evaluation-metrics)
- [Common Failure Modes & Fixes](#common-failure-modes--fixes)

## What is RAG?

RAG grounds LLM responses in external knowledge by retrieving relevant documents at query time and injecting them into the prompt. This reduces hallucination, keeps knowledge current without retraining, and provides source attribution.

---

## Core Pipeline

**Indexing (offline)**

```
Documents → Chunking → Embedding → Vector Store
```

**Querying (online)**

```
Query → Embed Query → Retrieve Top-k → Augment Prompt → LLM → Answer
```

---

## Chunking Strategies

| Strategy | Description | Best For |
|---|---|---|
| **Fixed-size** | Split by token/char count | Fast prototyping |
| **Recursive character** | Split on `\n\n` → `\n` → ` ` | Preserving paragraphs |
| **Semantic** | Split where sentence similarity drops | High-coherence chunks |
| **Document-aware** | Respect headings, tables, code blocks | Markdown, HTML, PDFs |
| **Parent-child** | Index small chunks; retrieve large parent | Precision + context |

> **Rule of thumb:** Start with 512 tokens and 10–15% overlap, then tune based on eval metrics.

---

## Embedding Models

| Model | Provider | Dims | Notes |
|---|---|---|---|
| `text-embedding-3-large` | OpenAI | 3072 | Strong general purpose |
| `BGE-M3` / `E5-large` | Open source | 1024 | SOTA on MTEB, multilingual |
| `embed-v3` | Cohere | 1024 | int8 + binary quantization |
| `nomic-embed-text` | Nomic | 768 | 8192-token context, Apache 2.0 |

---

## Retrieval Strategies

### Dense (vector similarity)
Embed query and chunks; find nearest neighbours by cosine / dot-product / L2. Best for semantic and paraphrased queries.

### Sparse (BM25 / keyword)
Term-frequency matching. Best for exact terms, proper nouns, IDs, and rare words that embeddings compress away.

### Hybrid (dense + sparse)
Fuse both scores using **Reciprocal Rank Fusion (RRF)** or a weighted sum. Best overall recall in practice — use this as your baseline.

### Advanced

- **HyDE** — generate a hypothetical answer, embed it, retrieve. Bridges the query-document vocabulary gap.
- **Multi-query** — LLM rewrites the query N ways; take the union of results, dedup, then rerank.
- **Multi-vector (ColBERT)** — token-level late interaction. Higher precision, slower than bi-encoders.

---

## Reranking

Run a second-stage ranker on the top-k candidates to improve precision before sending context to the LLM.

| Method | Quality | Speed | Notes |
|---|---|---|---|
| Cross-encoder | ★★★★★ | Slow | Score (query, chunk) jointly; use on top-20 to 50 |
| Cohere Rerank / Jina | ★★★★☆ | Medium | API-based cross-encoders, plug-and-play |
| RRF | ★★★☆☆ | Fast | `1 / (k + rank)` — no score calibration needed |
| LLM-as-reranker | ★★★★☆ | Slowest | Flexible, expensive; good for high-stakes queries |

---

## Vector Databases

| Database | Type | Highlights |
|---|---|---|
| **Pinecone** | Managed | Serverless, auto-scaling |
| **Weaviate** | OSS / Cloud | Built-in hybrid search, GraphQL |
| **Qdrant** | OSS / Cloud | Rust-based, payload filtering |
| **pgvector** | Postgres extension | No new infra if already on Postgres |
| **ChromaDB** | Local / OSS | Lightweight, great for prototyping |
| **FAISS** | Library | Meta's library, no server, max control |

---

## ANN Index Algorithms

| Algorithm | Recall | Speed | Memory | Best For |
|---|---|---|---|---|
| **HNSW** | Very high | Fast | High | Default for most use cases |
| **IVF-PQ** | High | Fast | Low | Billions of vectors, compressed |
| **ScaNN** | Very high | Very fast | Medium | Large-scale production at Google |
| **Flat (exact)** | Perfect | Slow | Medium | < 100k vectors, ground truth eval |

---

## Advanced RAG Patterns

| Pattern | What It Does |
|---|---|
| **Self-RAG** | LLM decides when to retrieve; critiques its own output for faithfulness |
| **RAPTOR** | Cluster chunks → summarise → index summaries for hierarchical global retrieval |
| **GraphRAG** | Build entity/relation graph; traverse for multi-hop reasoning |
| **Agentic RAG** | Agent iterates: retrieve → reason → retrieve more if needed → answer |
| **Contextual compression** | LLM distils each chunk to only the relevant portion before prompt stuffing |
| **Metadata filtering** | Pre-filter by date, source, category before ANN search to reduce noise |

---

## Key Hyperparameters

| Parameter | Recommended Starting Point |
|---|---|
| Chunk size | 512 tokens |
| Chunk overlap | 10–20% of chunk size |
| Top-k retrieved | 3–10 (rerank from 20–50) |
| Similarity threshold | Cosine ≥ 0.7 (tune per domain) |
| HNSW `ef_construction` | 200 (higher = better recall, slower build) |
| HNSW `M` | 16–64 (neighbours per node) |

---

## Evaluation Metrics

| Metric | What It Measures |
|---|---|
| **Context precision** | % of retrieved chunks that are relevant |
| **Context recall** | % of relevant chunks that were retrieved |
| **Faithfulness** | Is the answer grounded in retrieved context? |
| **Answer relevance** | Does the answer address the question? |
| **MRR / NDCG** | Rank quality of retrieval |

**Evaluation frameworks:** [RAGAS](https://github.com/explodinggradients/ragas), [TruLens](https://github.com/truera/trulens), [Arize Phoenix](https://github.com/Arize-ai/phoenix)

---

## Common Failure Modes & Fixes

| Problem | Fix |
|---|---|
| **Lost in the middle** — LLM ignores middle chunks | Reorder: put best chunks first and last |
| **Chunk boundary cuts** — key info split across chunks | Increase overlap; use semantic chunking |
| **Query-document mismatch** — short query vs long doc | Use HyDE or multi-query expansion |
| **Hallucination despite retrieval** — LLM ignores context | Add explicit citation instructions in prompt; run faithfulness eval |
| **Stale index** — new documents not indexed | Event-driven upsert pipeline on document change |
| **Low recall on rare terms** — dense model misses exact tokens | Add BM25 (hybrid retrieval) |

---

## Further Reading

- [LangChain RAG docs](https://python.langchain.com/docs/concepts/rag/)
- [LlamaIndex documentation](https://docs.llamaindex.ai/)
- [RAGAS evaluation framework](https://github.com/explodinggradients/ragas)
- [Pinecone Learn — RAG guide](https://www.pinecone.io/learn/retrieval-augmented-generation/)
- [Weaviate blog — Hybrid Search](https://weaviate.io/blog/hybrid-search-explained)
