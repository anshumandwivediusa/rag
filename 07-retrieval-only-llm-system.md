# Retrieval-Only LLM System

## Overview
This project configures a Large Language Model (LLM) to operate in **retrieval-only mode**.  
In this setup, the model answers **strictly based on data available in the system** (e.g., a vector database, knowledge base, or document store).  
If no relevant data is found, the model **refuses to generate speculative answers** and instead returns a controlled fallback response.

## Goals
- Prevent **hallucinations** by restricting generative reasoning.
- Ensure all answers are **grounded in retrieved context**.
- Provide a **consistent fallback message** when no data is available.
- Allow developers to enforce **confidence thresholds** for retrieval relevance.

## Architecture
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

## Restriction Strategies

#### 1. System Prompt Enforcement

The system prompt serves as the primary control mechanism for model behavior. It should explicitly instruct the AI to generate responses only from the information retrieved from the approved knowledge base.

**Example Instruction:**

> "You must answer only using the information provided in the retrieved context. If the required information is not available, respond with: 'No information available.' Do not infer, assume, or generate additional facts."

**Benefits:**

* Reduces hallucinations and unsupported claims.
* Establishes clear operational boundaries for the model.
* Ensures consistency across user interactions.

#### 2. Guardrails and Validation Layer

A guardrails layer acts as an intermediary between the retrieval system and the final response generation process. It verifies whether the generated answer is adequately supported by the retrieved content.

**Key Functions:**

* Compare generated responses against retrieved documents.
* Detect unsupported statements or fabricated information.
* Block, revise, or replace responses that lack sufficient grounding.
* Enforce organizational compliance and content policies.

**Benefits:**

* Prevents dissemination of inaccurate information.
* Enhances trustworthiness and reliability of outputs.
* Provides an additional layer of governance beyond prompt engineering.

#### 3. Confidence Thresholds

Confidence thresholds ensure that only highly relevant retrieved content is used to generate responses. Retrieval results are evaluated using similarity metrics such as cosine similarity, semantic relevance scores, or reranker confidence scores.

**Example Configuration:**

* Minimum cosine similarity score: **0.75**
* Minimum reranker confidence score: **80%**

**Operational Logic:**

* If retrieved documents exceed the threshold, proceed with response generation.
* If no document meets the threshold, trigger a fallback or refusal response.

**Benefits:**

* Improves response accuracy.
* Reduces the risk of using weakly related or irrelevant content.
* Enhances overall retrieval quality.

#### 4. Standardized Refusal and Fallback Responses

When the system cannot locate sufficiently relevant information, it should provide a consistent and predefined response instead of attempting to generate an answer.

**Example Responses:**

* "No information available."
* "The requested information could not be found in the approved knowledge sources."
* "Insufficient context was retrieved to provide a reliable answer."

**Benefits:**

* Eliminates inconsistent refusal behavior.
* Prevents misleading or speculative responses.
* Improves user transparency regarding system limitations.

### Recommended Multi-Layer Protection Approach

For maximum reliability, implement all four controls together:

1. **System Prompt Enforcement** – Defines model behavior.
2. **Confidence Thresholds** – Filters low-quality retrieval results.
3. **Guardrails Validation Layer** – Verifies response grounding.
4. **Standardized Refusal Messages** – Handles cases with insufficient evidence.

This layered approach significantly reduces hallucinations, improves response accuracy, and ensures that all generated outputs remain grounded in verified retrieved content.


## Configuration
### Example Settings
```yaml
retrieval:
  vector_store: "faiss"
  similarity_metric: "cosine"
  threshold: 0.75

llm:
  mode: "retrieval-only"
  fallback_message: "No information available in the system."
