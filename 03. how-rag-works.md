# Real Explanation: How the System “Extracts” the Document

**Documents are NOT stored “in embedded format only.”**  
They are stored in **two parts**:

1. **The original document text** (PDF text, paragraph, FAQ, etc.)  
2. **The embedding vector** (a numerical representation used only for search)

## **1. Step 1 — Create Embeddings Using a Text Embedding Model**

Assume we use a **text‑embedding model** such as:

- **text‑embedding‑3‑large**  
- **sentence‑transformers**  
- **InstructorXL embeddings**  

These models convert text into **high‑dimensional vectors** (e.g., 768 or 1024 dimensions).

Example document:

```
"To factory reset the TechCare X15 laptop, press F11 during boot and select 'System Recovery'."
```

Embedding output (shortened):

```
[0.12, -0.44, 0.91, -0.02, 0.55, ...]
```

## **2. Step 2 — Store Documents + Embeddings in Postgres**

### **Postgres Table (Real Example)**

```sql
CREATE TABLE techcare_docs (
    id SERIAL PRIMARY KEY,
    document_text TEXT,
    metadata JSONB,
    embedding_vector VECTOR(768)  -- depends on embedding model dimension
);
```

### **Insert Example Rows**

| id | document_text | metadata | embedding_vector |
|----|---------------|----------|------------------|
| 1 | "To factory reset the TechCare X15 laptop, press F11 during boot and select 'System Recovery'." | {"category": "reset", "device": "X15"} | `[0.12, -0.44, 0.91, -0.02, ...]` |
| 2 | "If your battery drains quickly, update the BIOS and reduce screen brightness to 60%." | {"category": "battery", "device": "X15"} | `[0.88, 0.02, 0.55, -0.31, ...]` |
| 3 | "The TechCare X15 supports up to 32GB RAM. Use DDR4 3200MHz modules for best performance." | {"category": "hardware", "device": "X15"} | `[0.03, -0.91, 0.14, 0.77, ...]` |


## **3. Step 3 — User Query → Embedding**

User asks:

> “How do I reset my TechCare X15 laptop”

Embedding model produces a vector:

```
[0.11, -0.52, 0.77, 0.01, ...]
```

This is the **query embedding**.


## **4. Step 4 — Similarity Search in Postgres**

We now search for the closest documents using **cosine similarity**.

```sql
SELECT id, document_text, metadata
FROM techcare_docs
ORDER BY embedding_vector <-> '[0.11, -0.52, 0.77, 0.01, ...]'
LIMIT 3;
```

Postgres returns:

| id | document_text | metadata |
|----|---------------|----------|
| **1** | "To factory reset the TechCare X15 laptop..." | {"category": "reset"} |
| 3 | "The TechCare X15 supports up to 32GB RAM..." | {"category": "hardware"} |
| 2 | "If your battery drains quickly..." | {"category": "battery"} |

Document **1** is the closest match.

## **5. Step 5 — Build Context for the LLM**

We take the top documents and build a context block:

```
Relevant Document 1:
"To factory reset the TechCare X15 laptop, press F11 during boot..."

Relevant Document 2:
"The TechCare X15 supports up to 32GB RAM..."

Relevant Document 3:
"If your battery drains quickly..."
```

## **6. Step 6 — LLM Generates the Final Answer**

The model uses the retrieved context to answer:

> To reset your TechCare X15 laptop, restart the device and press F11 during boot to enter System Recovery. Select “Factory Reset” and follow the on‑screen instructions.

This answer is **grounded** in the retrieved documents.


## **7. Full Pipeline Summary**

| Step | What Happens | Why It Matters |
|------|--------------|----------------|
| **Query Encoding** | Convert user text → embedding | Enables semantic search |
| **Vector Search** | Postgres finds closest documents | Retrieves relevant knowledge |
| **Context Fusion** | Combine query + documents | Gives LLM real evidence |
| **Generation** | LLM produces grounded answer | Reduces hallucinations |


## **Sentence Summary**

> A text‑embedding model converts both documents and queries into vectors, Postgres (pgvector) finds the closest matches, and the LLM uses the retrieved text—not the embeddings—to generate accurate, grounded answers.
