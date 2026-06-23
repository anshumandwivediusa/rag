# Retrieval-Only LLM System

## 📖 Overview
This project configures a Large Language Model (LLM) to operate in **retrieval-only mode**.  
In this setup, the model answers **strictly based on data available in the system** (e.g., a vector database, knowledge base, or document store).  
If no relevant data is found, the model **refuses to generate speculative answers** and instead returns a controlled fallback response.

---

## 🎯 Goals
- Prevent **hallucinations** by restricting generative reasoning.
- Ensure all answers are **grounded in retrieved context**.
- Provide a **consistent fallback message** when no data is available.
- Allow developers to enforce **confidence thresholds** for retrieval relevance.

---

## ⚙️ Architecture
The system follows a **Retrieval-Augmented Generation (RAG)** pipeline but with strict restrictions:

1. **[User Query](ca://s?q=User_query_in_RAG)**  
   - Input from the user is passed into the system.

2. **[Embedding Search](ca://s?q=Embedding_search_in_RAG)**  
   - Query is converted into embeddings.  
   - Similarity search is performed against the knowledge base.

3. **[Confidence Check](ca://s?q=Confidence_thresholds_in_RAG)**  
   - If similarity score ≥ threshold → retrieved data is passed to the LLM.  
   - If similarity score < threshold → fallback response is triggered.

4. **[LLM Response](ca://s?q=LLM_response_in_RAG)**  
   - Model generates an answer **only using retrieved context**.  
   - No external reasoning or hallucination is allowed.

5. **[Fallback Handling](ca://s?q=Fallback_handling_in_RAG)**  
   - If no relevant data is found, the system outputs:  
     ```
     No information available in the system.
     ```

---

## 🔒 Restriction Strategies
- **System Prompt Enforcement**  
  - Example: *"You must only answer using retrieved data. If none is found, respond with 'No information available.'"*

- **Guardrails Layer**  
  - Middleware validates whether the response is grounded in retrieved text.  
  - Blocks or replaces ungrounded outputs.

- **Confidence Thresholds**  
  - Define a minimum similarity score (e.g., cosine similarity ≥ 0.75).  
  - Ensures only highly relevant matches are used.

- **Custom Refusal Messages**  
  - Standardized fallback prevents inconsistent or misleading outputs.

---

## 🛠️ Configuration
### Example Settings
```yaml
retrieval:
  vector_store: "faiss"
  similarity_metric: "cosine"
  threshold: 0.75

llm:
  mode: "retrieval-only"
  fallback_message: "No information available in the system."
