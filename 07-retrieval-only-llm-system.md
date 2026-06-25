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
```

## Implementing Confidence Thresholds in a RAG System

Confidence thresholds determine whether retrieved content is sufficiently relevant to answer a user's question. If retrieval confidence falls below a predefined threshold, the system should refuse to answer or request clarification.

### 1. Using Vector Similarity Scores

Most vector databases return a similarity score between the query embedding and retrieved document embeddings.

#### Example

| Document | Similarity Score |
| -------- | ---------------- |
| Doc A    | 0.89             |
| Doc B    | 0.82             |
| Doc C    | 0.61             |

**Threshold = 0.75**

Result:

* Doc A → Accepted
* Doc B → Accepted
* Doc C → Rejected

#### Pseudocode

```python
results = vector_store.similarity_search(query, k=5)

threshold = 0.75

valid_docs = [
    doc for doc in results
    if doc.score >= threshold
]

if not valid_docs:
    return "No information available."

return generate_answer(valid_docs)
```

### 2. Using Top Result Threshold

A simple approach is to evaluate only the highest-ranked document.

#### Logic

```python
top_doc = results[0]

if top_doc.score < 0.75:
    return "No information available."
```

#### Advantages

* Easy to implement.
* Works well for FAQ-style knowledge bases.

#### Limitations

* May ignore useful supporting documents.


### 3. Average Score Threshold

Require the average score of the top retrieved documents to exceed a threshold.

#### Example

Top 3 scores:

```text
0.86
0.81
0.79
```

Average:

```text
(0.86 + 0.81 + 0.79) / 3 = 0.82
```

Since 0.82 > 0.75, proceed.

#### Pseudocode

```python
scores = [doc.score for doc in results[:3]]

avg_score = sum(scores) / len(scores)

if avg_score < 0.75:
    return "No information available."
```


### 4. Combining Similarity and Reranking Scores

Many production RAG systems use a reranker model (e.g., Cross-Encoder, Cohere Rerank, BGE Reranker).

#### Pipeline

```text
User Query
    ↓
Vector Search
    ↓
Top 20 Documents
    ↓
Reranker
    ↓
Top 5 Documents + Relevance Scores
```

#### Example

| Document | Vector Score | Reranker Score |
| -------- | ------------ | -------------- |
| Doc A    | 0.82         | 0.95           |
| Doc B    | 0.85         | 0.41           |
| Doc C    | 0.79         | 0.91           |

Accept only documents where:

```text
Vector Score ≥ 0.75
AND
Reranker Score ≥ 0.80
```

This is significantly more reliable than vector similarity alone.

### 5. Context Coverage Threshold

Before generating an answer, verify that retrieved documents contain enough information.

#### Example

Question:

> "What is the refund period for premium subscriptions?"

Retrieved document:

> "Premium subscriptions are available monthly and yearly."

The topic matches, but the answer is missing.

The system should return:

```text
Insufficient information available.
```

rather than generating a guess.

This check can be implemented with:

* LLM-as-judge
* Semantic answer verification
* Retrieval coverage evaluation

### 6. Confidence-Based Response Routing

Different confidence levels can trigger different behaviors.

| Confidence  | Action                  |
| ----------- | ----------------------- |
| ≥ 0.90      | Answer normally         |
| 0.75 – 0.90 | Answer with citation    |
| 0.60 – 0.75 | Ask clarifying question |
| < 0.60      | Refuse answer           |

#### Example

```python
score = top_doc.score

if score >= 0.90:
    return answer()

elif score >= 0.75:
    return answer_with_sources()

elif score >= 0.60:
    return "Can you clarify your question?"

else:
    return "No information available."
```

### Recommended Production Configuration

For enterprise RAG systems:

```text
Vector Similarity Threshold: 0.75
Top-K Retrieval: 10
Reranker Threshold: 0.80
Minimum Supporting Documents: 2
Grounding Validation: Enabled
Fallback Response: Standardized
```

This combination provides much better protection against hallucinations than relying on prompts alone because the system filters weak retrieval results before the LLM generates a response.
