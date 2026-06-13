# Real Explanation: How the System “Extracts” the Document

**Documents are NOT stored “in embedded format only.”**  
They are stored in **two parts**:

1. **The original document text** (PDF text, paragraph, FAQ, etc.)  
2. **The embedding vector** (a numerical representation used only for search)


## **Step 1 — You store BOTH the text and the embedding in Postgres**
A typical pgvector table looks like this:

| id | document_text | embedding_vector |
|----|----------------|------------------|
| 1  | "Factory reset steps..." | [0.12, -0.44, ...] |
| 2  | "Battery troubleshooting..." | [0.88, 0.02, ...] |

So the **text is still there**.  
The embedding is just an *extra column* used for similarity search.

---

## Step 2 — Query is converted into an embedding  
Your query:

> “How do I reset my TechCare X15 laptop”

becomes a vector like:

\[
[0.11, -0.52, 0.77, \dots]
\]

---

## Step 3 — Postgres searches ONLY the embedding column  
Postgres finds the **closest vectors** using cosine similarity.

It returns the **rows**, not just the vectors.

So the result includes:

- the document ID  
- the original text  
- metadata  
- and the embedding (if needed)

---

## Step 4 — The system pulls the original text  
Once Postgres returns the matching rows, the system extracts:

- **document_text**  
- **title**  
- **sections**  
- **metadata**

This is the content fed into the LLM.

**The model NEVER reads the embedding.**  
It only reads the **original text** stored alongside the embedding.

---

## ⭐ Real‑World Example (Very Clear)

### **Stored in Postgres**
```
id: 1
document_text: "To factory reset the X15, press F11 during boot..."
embedding_vector: [0.12, -0.44, 0.91, ...]
```

### **User query → embedding → similarity search**

Postgres returns row 1 because its embedding is closest.

### **System extracts:**
```
"To factory reset the X15, press F11 during boot..."
```

### **LLM uses this text to answer.**

---

# ⭐ One‑Sentence Summary  
> Documents are stored normally; embeddings are stored alongside them. The system searches using embeddings but retrieves the original text, which is then passed to the model.

---

If you want, I can also show you a **sample Postgres schema**, a **full RAG pipeline diagram**, or a **step‑by‑step code example** using pgvector.
