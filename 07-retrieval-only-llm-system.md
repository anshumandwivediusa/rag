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

## **ChromaDB**, implementing **Relevance** and **Confidence**
For **ChromaDB**, implementing **Relevance** and **Confidence** are two separate concerns:

* **Relevance** is handled by **ChromaDB** during retrieval.
* **Confidence** is **not provided by ChromaDB**. You must implement it yourself using guardrails, LLM evaluation, or grounding verification.

| **Aspect** | **Similarity** | **Confidence** | **Relevance** |
| --- | --- | --- | --- |
| **Signal type** | Numeric vector closeness | Retriever’s trust in similarity | Semantic fit to query intent |
| **Computed by** | Embedding math | Ranking top‑N by similarity | Cross‑encoder, LLM, heuristics |
| **Strength** | Objective, fast | Easy to use for retrieval | Accurate, context‑aware |
| **Weakness** | Ignores meaning | Can be fooled by overlap | Slower, more compute |
| **Best use** | Raw scoring | Candidate selection | Final reranking & grounding |
### Overall Architecture

```text
                 User Query
                      │
                      ▼
              Generate Embedding
                      │
                      ▼
                 ChromaDB Search
                      │
         Similarity Score (Relevance)
                      │
      ┌───────────────┴───────────────┐
      │                               │
 Score >= Threshold?             Score < Threshold
      │                               │
      ▼                               ▼
   Send Context                 Return "No Information"
      │
      ▼
      LLM Generates Answer
      │
      ▼
 Grounding Verification (Confidence)
      │
      ├── Supported → Return Answer
      └── Unsupported → Regenerate / Refuse
```



### Step 1. Implement Relevance in ChromaDB

When querying ChromaDB:

```python
results = collection.query(
    query_texts=["What is RAG?"],
    n_results=5
)
```

If using LangChain:

```python
docs = vectorstore.similarity_search_with_score(
    query,
    k=5
)
```

Example output:

```python
[
(Document(...), 0.18),
(Document(...), 0.24),
(Document(...), 0.71)
]
```

> **Note:** Depending on the embedding function and API, Chroma may return **distance** rather than **similarity**. Lower distance usually means a better match. If you're using cosine distance:
>
> * **0.0** = perfect match
> * Larger values = less similar

Always check whether your configuration returns **distance** or **similarity** before setting thresholds.



#### Example Relevance Threshold

If using cosine distance:

```python
MAX_DISTANCE = 0.30

relevant_docs = [
    doc
    for doc, distance in docs
    if distance <= MAX_DISTANCE
]
```

If no documents pass:

```python
return "No information available."
```

This is your **Relevance Guardrail**.



### Step 2. Context Compression

Suppose Chroma returns:

```
Chunk 1
Chunk 2
Chunk 3
Chunk 4
Chunk 5
```

Some chunks may be redundant.

Compress:

```
Chunk 1
Chunk 3
Chunk 5
```

This reduces tokens and improves answer quality.

LangChain provides:

```python
ContextualCompressionRetriever
```



### Step 3. Generate the Answer

Prompt:

```text
Answer ONLY from the supplied context.

If the answer cannot be found,

reply:

"No information available."

Never make assumptions.
```



### Step 4. Confidence Check (Grounding)

Now verify the answer.

Suppose:

Context:

```
Refund period = 30 days.
```

Generated:

```
Refund period = 45 days.
```

This should fail.



#### Method 1 (Recommended): LLM-as-a-Judge

Call the LLM again:

```text
Context:

<retrieved documents>

Answer:

<generated answer>

Question:

Is every factual statement in the answer supported by the context?

Return JSON:

{
 "supported": true,
 "confidence": 0.92,
 "unsupported_claims":[]
}
```

Example output:

```json
{
 "supported": false,
 "confidence": 0.31,
 "unsupported_claims":[
   "Refund period is 45 days"
 ]
}
```

Now reject the answer.



### Step 5. Confidence Threshold

```python
if confidence < 0.80:
    regenerate()
```

or

```python
return "The retrieved information is insufficient to answer this question."
```



### Step 6. Citation Check

Every statement should originate from one of the retrieved chunks.

Good:

```
Refund period: 30 days

Source:
EmployeePolicy.pdf
Section 4.2
```

Bad:

```
Refund period: 45 days
```

No supporting chunk.

Reject.



### Step 7. Policy Guardrails

Run:

* PII detection
* Toxicity detection
* Company policy validation

Example:

```
Employee SSN:

123-45-6789
```

↓

```
Employee SSN:

***-**-6789
```



### Complete Flow

```python
query = "What is the refund policy?"

# Retrieve
docs = chroma.similarity_search_with_score(query)

# Relevance Guardrail
relevant_docs = [
    doc
    for doc, distance in docs
    if distance < 0.30
]

if len(relevant_docs) == 0:
    return "No information available."

# Generate
answer = llm.generate(query, relevant_docs)

# Confidence Guardrail
evaluation = judge_llm(
    answer=answer,
    context=relevant_docs
)

if not evaluation.supported:
    return "The retrieved information does not fully support an answer."

return answer
```



### Production Pipeline

```text
User Question
      │
      ▼
Embedding
      │
      ▼
ChromaDB
      │
      ▼
Similarity Score
      │
      ▼
Relevance Threshold
      │
      ▼
Metadata Filter
      │
      ▼
Context Compression
      │
      ▼
LLM
      │
      ▼
Grounding Verification
      │
      ▼
Citation Validation
      │
      ▼
Policy Check
      │
      ▼
Confidence Score
      │
      ▼
Approve / Reject
      │
      ▼
Final Answer
```

#### Enterprise Best Practice

For a production RAG system using **ChromaDB + LangChain**, a robust implementation typically includes:

| Stage                | Implementation                                                                    |
| -------------------- | --------------------------------------------------------------------------------- |
| Retrieval            | `Chroma.similarity_search_with_score()` or `as_retriever()`                       |
| Relevance            | Distance/similarity threshold + metadata filters                                  |
| Retrieval Quality    | Re-ranking (e.g., CrossEncoder, Cohere Rerank, BGE Reranker)                      |
| Context Optimization | `ContextualCompressionRetriever`                                                  |
| Generation           | Prompt instructing the LLM to answer only from retrieved context                  |
| Confidence           | LLM-as-a-Judge or a groundedness checker to verify support for the answer         |
| Safety               | PII redaction, toxicity checks, and organizational policy enforcement             |
| Fallback             | Return a standardized message when relevance or confidence thresholds are not met |

**Key point:** ChromaDB's responsibility ends at **retrieving relevant documents**. It does **not** determine whether the generated answer is correct. That confidence check is part of the guardrails layer you build around your RAG application.
