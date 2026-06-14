| Technique | Description | Why Use It | Pros | Cons | Example |
| --- | --- | --- | --- | --- | --- |
| **Naive RAG** | Basic RAG: embed → retrieve → answer. | Simple apps, prototypes, small datasets. | Easy to build; fast; low cost. | Retrieval quality is limited; struggles with vague queries. |
| **[Hybrid RAG](ca://s?q=Explain_Hybrid_RAG)** | Combines vector search + keyword search (BM25). | When queries need both meaning + exact match. | More accurate; handles rare terms; robust. | Slightly more complex; requires two search systems. |
| **[HyDE RAG](ca://s?q=Explain_HyDE_RAG)** | LLM generates a hypothetical answer → embed → retrieve. | When queries are short, vague, or unclear. | Big boost in retrieval quality; great for reasoning tasks. | Extra LLM call increases cost + latency. 
| **[Parent Document Retriever](ca://s?q=Explain_Parent_Document_Retriever)** | Splits docs into chunks but returns the full parent doc. | When context is lost in small chunks. | Preserves full meaning; reduces hallucinations. | Larger context → higher token cost. 
| **[RAG Fusion](ca://s?q=Explain_RAG_Fusion)** | Creates multiple sub‑queries → retrieves → merges using RRF. | Complex questions needing multi‑angle retrieval. | Very strong retrieval accuracy; robust to query phrasing. | More compute; more retrieval calls. 
| **[Contextual RAG](ca://s?q=Explain_Contextual_RAG)** | Compresses retrieved docs to keep only relevant parts. | When you want high accuracy with low token cost. | Saves tokens; improves answer focus. | Compression may remove useful context. 
| **[Rewrite–Retrieve–Read](ca://s?q=Explain_Rewrite_Retrieve_Read)** | LLM rewrites the query → retrieves → answers. | When user queries are unclear or poorly phrased. | Better retrieval; more natural answers. | Extra LLM step adds cost + latency. 
| **[Unstructured RAG](ca://s?q=Explain_Unstructured_RAG)** | Handles mixed data: PDFs, tables, images, HTML. | Real‑world messy documents. | Works with any document type; flexible. | Requires preprocessing; slower ingestion. 

# **Naive RAG — Postgres + ChromaDB**

### **PostgreSQL (pgvector)**  
```sql
SELECT content
FROM documents
ORDER BY embedding <-> $1
LIMIT 5;
```

### **ChromaDB**  
```python
collection.query(
    query_embeddings=[query_embedding],
    n_results=5
)
```

# **Hybrid RAG — Postgres + ChromaDB**

### **PostgreSQL (pgvector + BM25 full‑text search)**  
```sql
WITH
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
```

### **ChromaDB (Hybrid = vector + keyword)**  
ChromaDB doesn’t have BM25 built‑in, so hybrid is done like this:

```python
results = collection.query(
    query_texts=["your query"],
    n_results=5,
    include=["documents", "distances"]
)
# keyword filtering done in Python
```

Or using **Chroma + BM25** via external library:

```python
vector_results = collection.query(query_texts=["your query"], n_results=5)
keyword_results = bm25.search("your query")

# combine scores manually (RRF or weighted)
```

# **HyDE RAG — Postgres + ChromaDB**

### **PostgreSQL**  
```sql
SELECT content
FROM documents
ORDER BY embedding <-> $1
LIMIT 5;
```

*(Where `$1` is the embedding of the **hypothetical answer**.)*

### **ChromaDB**  
```python
collection.query(
    query_embeddings=[hyde_embedding],
    n_results=5
)
```

# **Parent Document Retriever — Postgres + ChromaDB**

### **PostgreSQL**  
```sql
SELECT parent_id, parent_content
FROM documents
ORDER BY embedding <-> $1
LIMIT 5;
```

### **ChromaDB**  
```python
results = collection.query(
    query_embeddings=[query_embedding],
    n_results=5
)
parent_docs = [doc["parent_id"] for doc in results["ids"]]
```

# **RAG Fusion — Postgres + ChromaDB**

### **PostgreSQL (RRF fusion)**  
```sql
SELECT id, content,
       SUM(1.0 / (rank + 60)) AS rrf_score
FROM temp_results
GROUP BY id, content
ORDER BY rrf_score DESC
LIMIT 5;
```

### **ChromaDB**  
```python
q1 = collection.query(query_texts=[q1], n_results=5)
q2 = collection.query(query_texts=[q2], n_results=5)
q3 = collection.query(query_texts=[q3], n_results=5)

# apply RRF in Python
```

---

# **Contextual RAG — Postgres + ChromaDB**

### **PostgreSQL**  
```sql
SELECT content
FROM documents
ORDER BY embedding <-> $1
LIMIT 5;
```

*(LLM compresses the retrieved text.)*

### **ChromaDB**  
```python
docs = collection.query(query_embeddings=[query_embedding], n_results=5)
compressed = llm.compress(docs)
```

# **Rewrite–Retrieve–Read — Postgres + ChromaDB**

### **PostgreSQL**  
```sql
SELECT content
FROM documents
ORDER BY embedding <-> $1
LIMIT 5;
```

*(Where `$1` is the embedding of the **rewritten query**.)*

### **ChromaDB**  
```python
collection.query(
    query_embeddings=[rewritten_embedding],
    n_results=5
)
```

# **Unstructured RAG — Postgres + ChromaDB**

### **PostgreSQL**  
```sql
INSERT INTO documents (content, embedding) VALUES (...);

SELECT content
FROM documents
ORDER BY embedding <-> $1
LIMIT 5;
```

### **ChromaDB**  
```python
collection.add(documents=[parsed_text], embeddings=[embedding])
collection.query(query_embeddings=[query_embedding], n_results=5)
```
