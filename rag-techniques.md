| Technique | Description | Why Use It | Pros | Cons | Example |
| --- | --- | --- | --- | --- | --- |
| **Naive RAG** | Basic RAG: embed → retrieve → answer. | Simple apps, prototypes, small datasets. | Easy to build; fast; low cost. | Retrieval quality is limited; struggles with vague queries. | `SELECT content FROM documents ORDER BY embedding <-> $1 LIMIT 5;` |
| **Hybrid RAG** | Combines vector search + keyword search (BM25). | When queries need both meaning + exact match. | More accurate; handles rare terms; robust. | Slightly more complex; requires two search systems. | <pre><code>WITH
vec AS (
    SELECT id, content,
           1 - (embedding <-> $2) AS vector_score
    FROM documents
),
kw AS (
    SELECT id,
           ts_rank(to_tsvector('english', content), plainto_tsquery($1)) AS keyword_score
    FROM documents
)
SELECT
    d.id,
    d.content,
    (0.6 * vec.vector_score + 0.4 * kw.keyword_score) AS hybrid_score
FROM documents d
LEFT JOIN vec ON d.id = vec.id
LEFT JOIN kw ON d.id = kw.id
ORDER BY hybrid_score DESC
LIMIT 5;
</code></pre> |
| **HyDE RAG** | LLM generates a hypothetical answer → embed → retrieve. | When queries are short, vague, or unclear. | Big boost in retrieval quality; great for reasoning tasks. | Extra LLM call increases cost + latency. | <pre><code>-- Step 1: LLM generates hypothetical answer  
-- Step 2: Embed hypothetical answer  
SELECT content
FROM documents
ORDER BY embedding <-> $1
LIMIT 5;
</code></pre> |
| **Parent Document Retriever** | Splits docs into chunks but returns the full parent doc. | When context is lost in small chunks. | Preserves full meaning; reduces hallucinations. | Larger context → higher token cost. | <pre><code>SELECT parent_id, parent_content
FROM documents
ORDER BY embedding <-> $1
LIMIT 5;
</code></pre> |
| **RAG Fusion** | Creates multiple sub‑queries → retrieves → merges using RRF. | Complex questions needing multi‑angle retrieval. | Very strong retrieval accuracy; robust to query phrasing. | More compute; more retrieval calls. | <pre><code>-- q1, q2, q3 retrieved separately  
-- Combine using Reciprocal Rank Fusion  
SELECT id, content,
       SUM(1.0 / (rank + 60)) AS rrf_score
FROM temp_results
GROUP BY id, content
ORDER BY rrf_score DESC
LIMIT 5;
</code></pre> |
| **Contextual RAG** | Compresses retrieved docs to keep only relevant parts. | High accuracy with low token cost. | Saves tokens; improves answer focus. | Compression may remove useful context. | <pre><code>-- Step 1: Retrieve  
SELECT content
FROM documents
ORDER BY embedding <-> $1
LIMIT 5;
-- Step 2: LLM compresses context
</code></pre> |
| **Rewrite–Retrieve–Read** | LLM rewrites the query → retrieves → answers. | When user queries are unclear or poorly phrased. | Better retrieval; more natural answers. | Extra LLM step adds cost + latency. | <pre><code>-- Step 1: LLM rewrites query  
-- Step 2: Retrieve using rewritten query  
SELECT content
FROM documents
ORDER BY embedding <-> $1
LIMIT 5;
</code></pre> |
| **Unstructured RAG** | Handles mixed data: PDFs, tables, images, HTML. | Real‑world messy documents. | Works with any document type; flexible. | Requires preprocessing; slower ingestion. | <pre><code>-- After parsing with Unstructured.io  
INSERT INTO documents (content, embedding) VALUES (...);
SELECT content
FROM documents
ORDER BY embedding <-> $1
LIMIT 5;
</code></pre> |

