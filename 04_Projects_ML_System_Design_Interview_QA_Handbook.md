# Deloitte Interview Handbook 4 --- Projects + ML System Design + Interview Q&A

## AI / ML Engineer II \| Project Deep Dive + Architecture + Interview Preparation

> **Goal:** Turn your existing projects into strong Engineer-II stories
> and prepare for system-design and deep follow-up questions.

------------------------------------------------------------------------

# 1. Your interview positioning

Your strongest narrative is:

``` text
ML Engineer + Financial-domain experience + Document Intelligence
+ GenAI/RAG + Agentic workflows + Production-oriented engineering
```

Do not position yourself as only a "LangChain developer." Present
yourself as an ML Engineer who builds end-to-end AI systems.

# 2. Project 1 --- Financial Document Intelligence

## Medium interview answer

> "One of my major projects was a template-agnostic financial document
> intelligence pipeline. The goal was to process financial documents
> from different markets and formats and convert unstructured and
> semi-structured content into structured outputs for downstream
> systems.
>
> The pipeline started with ingestion and OCR/text extraction, with
> table extraction handled separately. We combined textual and tabular
> information and generated document representations using BERT-based
> embeddings. A classification component identified the document
> category, followed by an entity-extraction layer using spatial/graph
> relationships and rule-based patterns where appropriate. The final
> output was structured JSON consumed by downstream systems.
>
> The key challenge was template variability across markets, so the
> architecture avoided relying on a fixed document template. We
> evaluated classification and extraction separately and achieved
> roughly 96% classification accuracy and 94% extraction accuracy across
> 10 markets.
>
> From a production perspective, I would monitor OCR quality,
> document-type distribution, extraction confidence, latency, failure
> rates and drift across markets/templates."

# 3. Project 1 architecture

``` text
Financial Documents
        ↓
      Ingestion
        ↓
  OCR / Text Extraction
        ↓
   Table Extraction
        ↓
 Structured + Text Content
        ↓
    Preprocessing
        ↓
 BERT Representation
        ↓
 Document Classification
        ↓
 Entity Extraction
   ├── Graph / spatial relationships
   └── Regex / rules
        ↓
 Schema / Validation
        ↓
       JSON
        ↓
 Downstream System
```

# 4. Why template-agnostic?

Fixed templates assume a field always appears at a known location. This
breaks when layouts, vendors, markets or table positions change.

A flexible pipeline uses semantic information, spatial relationships,
document structure and adaptable extraction logic.

# 5. Why BERT?

BERT produces contextual representations. For classification:

``` text
Text → tokenizer → BERT → representation → classifier
```

Compared with bag-of-words, contextual representations can capture
semantic relationships more effectively.

Strong answer:

> "I would choose BERT when contextual semantics are important and
> labeled data justifies the model complexity."

# 6. Why graph-based entity extraction?

Documents have spatial and structural relationships:

``` text
Field label → value
Table row → label + amount + date
```

Graph/spatial signals can include coordinates, proximity, reading order
and neighboring tokens.

# 7. Why not use an LLM everywhere?

> "I wouldn't assume an LLM is automatically better. For deterministic
> extraction patterns or high-volume classification, a smaller
> specialized model can be cheaper and faster. LLMs are valuable where
> semantic reasoning or ambiguity justifies their latency and cost. I
> would choose components based on accuracy, latency, cost,
> explainability and maintainability."

# 8. Project 1 evaluation

Classification can be evaluated with accuracy, precision, recall, F1 and
confusion matrices.

Extraction can be evaluated with suitable entity-level
precision/recall/F1 where available.

If the project metric was reported as extraction accuracy, do not invent
a different metric. Say:

> "The reported project metric was extraction accuracy; for a more
> rigorous production evaluation I would additionally track entity-level
> precision, recall and F1."

# 9. Project 1 production questions

### OCR failure?

Retry transient failures, use an alternate path if available, then route
to a failure/manual-review queue. Avoid infinite retries.

### Low confidence?

Use a confidence threshold and route uncertain cases to human review
when the business process requires it.

### Monitoring?

OCR success rate, classification quality, extraction quality, latency,
failures, distribution and drift.

# 10. Project 2 --- Multi-Agent compliance workflow

## Medium answer

> "Another project involved a multi-agent workflow for compliance
> automation. The objective was to reduce manual effort by combining
> OCR, retrieval-augmented context retrieval and schema-validated LLM
> extraction.
>
> I designed the workflow using LangGraph so different stages had
> explicit responsibilities and state transitions. OCR handled document
> content extraction, retrieval supplied relevant context, and the LLM
> generated structured output validated against a predefined schema. The
> workflow included controlled transitions and failure handling rather
> than allowing the model to run indefinitely.
>
> The main engineering challenge was reliability: an LLM can generate
> syntactically valid text that is semantically incorrect, so retrieval,
> schema validation and deterministic workflow controls were important."

# 11. Project 2 architecture

``` text
Document → OCR → preprocessing → Retriever → relevant context
                                           ↓
                                    LLM extraction
                                           ↓
                                     schema validation
                                      ↙            ↘
                                   valid       invalid
                                     ↓          retry/fallback
                                  output
```

If multiple agents are used:

``` text
Supervisor/Orchestrator
 ├── OCR
 ├── Retrieval
 ├── Validation
 └── LLM
```

# 12. Why LangGraph?

> "The workflow had explicit states, transitions and conditional paths,
> so a graph-based orchestration model was more appropriate than
> treating the process as one opaque chain."

Benefits: explicit state, conditional edges, retries, controlled loops,
observability and human-in-the-loop options.

# 13. Why RAG?

RAG separates knowledge retrieval from generation:

``` text
Question → retrieval → relevant context → LLM → answer
```

It is useful for private/current knowledge and can reduce dependence on
model memorization.

# 14. Hybrid retrieval

``` text
Semantic retrieval + BM25 lexical retrieval
             ↓
         candidate set
             ↓
          RRF/rerank
             ↓
          top-k context
```

Semantic search handles meaning; BM25 helps with exact identifiers,
terminology and codes.

# 15. Cross-encoder reranking

Bi-encoder compares separate embeddings. A cross-encoder scores a
query/document pair jointly and is generally more expensive.

Production pattern:

``` text
Vector/BM25 → Top 50 → Cross-encoder → Top 5 → LLM
```

# 16. RAG evaluation

Separate retrieval quality from generation quality.

Retrieval:

-   Recall@K
-   Precision@K
-   MRR
-   nDCG

Generation:

-   correctness
-   groundedness/faithfulness
-   relevance
-   source/citation correctness
-   human evaluation

A good final answer does not prove the retriever is good; evaluate
components independently.

# 17. MRR

For a query with the first relevant result at rank r:

``` text
RR = 1/r
MRR = average(RR across queries)
```

Rank 1 gives 1.0, rank 2 gives 0.5, rank 5 gives 0.2.

# 18. CRA project

## Medium explanation

> "I also worked on an enterprise Q&A system around Change Risk
> Assessment documents. The system combined historical data and a
> knowledge base with an agent layer that could retrieve relevant
> context, summarize information, execute SQL-based queries where
> required, and assist with downstream actions such as drafting emails.
>
> The key design consideration was separating retrieval, reasoning and
> deterministic actions. SQL execution was treated as a controlled tool
> rather than allowing arbitrary model-generated operations, and outputs
> needed validation before downstream use."

# 19. CRA architecture

``` text
User → UI/API → Agent/Orchestrator
                   ├── Knowledge Retriever
                   ├── SQL Tool
                   ├── Summarization
                   └── Email Drafting
                         ↓
                    Guardrails
                         ↓
                 Response / Draft
```

# 20. Tool safety

Do not let an LLM freely execute arbitrary database operations.

``` text
LLM → structured tool request → allowlist → validation → database
```

Where appropriate, restrict analytics tools to safe/read-only
operations.

# 21. Agent vs RAG

RAG:

``` text
retrieve context → generate answer
```

Agent:

``` text
reason → choose tool → execute → inspect → continue
```

They are complementary rather than mutually exclusive.

# 22. Agent safeguards

Production agent controls:

-   maximum steps
-   maximum retries
-   timeouts
-   typed state
-   tool allowlists
-   schema validation
-   fallback
-   human review
-   deterministic termination

# 23. ML system design framework

When asked to design a system:

``` text
1. Clarify objective
2. Users
3. Inputs/outputs
4. Scale/SLA
5. Data architecture
6. Features
7. Model
8. Training
9. Evaluation
10. Serving
11. Storage/cache
12. Monitoring
13. Failure handling
14. Security/governance
15. Cost/scaling
```

# 24. Example --- production document classification

``` text
Client → API Gateway → Upload → Blob Storage → Queue
                                      ↓
                                Processing Workers
                                      ↓
                                OCR / Extraction
                                      ↓
                                 ML Classifier
                                      ↓
                              Confidence check
                              ↙             ↘
                         auto-process      review
                              \             /
                               Result Store
```

For large documents, asynchronous processing avoids keeping HTTP
connections open for long periods.

# 25. Why queue?

Queues decouple upload from processing and provide backpressure, retries
and scalable workers.

Return a job ID and let clients poll or receive a callback/event.

# 26. Reliability

Potential failures:

-   OCR outage
-   model outage
-   storage failure
-   malformed document
-   timeout
-   downstream failure

Use timeouts, bounded retries, dead-letter queues, idempotency and
manual/fallback paths where appropriate.

# 27. Idempotency

If the same document is processed twice, the system should not
accidentally create duplicate side effects.

Use a unique job/document ID and design processing to be safe to repeat.

# 28. Caching

Potential caches include embeddings, retrieval results or safe response
caches.

Always consider staleness, authorization boundaries and whether the
request is user-specific.

# 29. API design

Possible endpoints:

``` text
POST /documents
GET  /documents/{id}
GET  /documents/{id}/status
GET  /documents/{id}/result
```

For simple synchronous inference: `POST /predict`.

# 30. Security

Production systems need authentication, authorization, encryption,
secrets management, network controls, input validation and audit
logging.

For enterprise documents, avoid putting sensitive raw content into
generic logs.

# 31. RAG system design

``` text
Client → Gateway → Agent Service → query rewrite
                         ↓
                  Hybrid retrieval
                   ↙          ↘
               Vector DB      BM25
                   \          /
                      RRF
                       ↓
                 Cross-encoder
                       ↓
                    Top-K
                       ↓
                     LLM
                       ↓
                Output validation
```

# 32. RAG latency budget

Example target p95 of 2 seconds:

``` text
API overhead       100 ms
retrieval          200 ms
reranking          300 ms
LLM               1200 ms
postprocessing     100 ms
-------------------------
total             1900 ms
```

The numbers are illustrative. The important idea is to assign budgets
and measure p50/p95/p99 per component.

# 33. Prompt injection

Treat retrieved documents as **data**, not trusted instructions.

Defenses:

-   strong instruction hierarchy
-   tool permission boundaries
-   output validation
-   least privilege
-   no secrets exposed to the model
-   separation of data and instructions
-   human approval for sensitive actions

# 34. Production RAG observability

Track request ID, retrieval latency, retrieved document IDs, reranker
scores, LLM latency, token usage, model version, prompt version and
errors. Avoid logging sensitive content unnecessarily.

# 35. Backend/system-design essentials

Know API flow, authentication vs authorization, rate limiting, timeouts,
bounded retries, idempotency, caching, load balancing and queues.

A retry is appropriate for transient failures but dangerous when the
operation is non-idempotent or retries are unbounded.

# 36. Behavioral --- tell me about yourself

Suggested 60--90 second structure:

> "I'm an ML Engineer with around four years of experience, primarily
> working on enterprise AI and financial-domain use cases. My work has
> covered document intelligence, NLP/ML pipelines and more recently RAG
> and agentic workflows. I've worked across the lifecycle from data
> processing and model development to application integration and
> production-oriented engineering. I'm now looking for an AI/ML
> engineering role where I can work on larger-scale end-to-end systems
> and deepen my production and cloud expertise."

# 37. Why Deloitte?

Keep it role-focused:

> "The role interests me because it combines data science and machine
> learning with end-to-end engineering and client-facing delivery. The
> JD covers model development, Python/SQL, distributed data processing,
> cloud/deployment and large-scale projects, which matches the direction
> I want to develop in."

# 38. Why switch?

Use:

``` text
What I learned + What I want next + Why this role fits
```

Example:

> "I've built a strong foundation in enterprise ML and AI systems in my
> current role. At this stage I want broader ownership of end-to-end ML
> systems, including deployment, cloud and large-scale engineering."

# 39. Strength

A useful technical strength:

> "One of my strengths is breaking an ambiguous AI problem into explicit
> components and evaluating each component separately rather than
> treating the whole system as a black box."

Then support it with a project example.

# 40. Weakness

A credible engineering answer:

> "Earlier I focused more heavily on the ML/AI side than infrastructure.
> I've been deliberately strengthening cloud, Docker, deployment and
> system-design knowledge so I can own more of the production
> lifecycle."

# 41. Client-facing requirements

For ambiguous requirements:

``` text
Clarify objective → identify users → define measurable output
→ document assumptions → confirm constraints → prototype → validate → iterate
```

Don't start coding before understanding the problem.

# 42. Agile

Know sprint planning, backlog, estimation, standups, retrospectives,
code review and defect management at a practical level.

# 43. Peer review

Review for:

``` text
Correctness
Readability
Tests
Performance
Security
Error handling
Maintainability
Observability
```

# 44. Full system-design sample --- document extraction

### Requirements

Input: PDF/image. Output: structured JSON. Potentially thousands/day.
Asynchronous processing. Auditability and confidence handling.

### Architecture

``` text
Client → Gateway → Upload → Blob → Queue → Workers
                                           ↓
                                          OCR
                                           ↓
                                    LLM/ML extraction
                                           ↓
                                    Schema validation
                                           ↓
                                        Database
```

### Reliability

Timeouts, bounded retries, dead-letter queue, idempotency and human
review.

### Scaling

Horizontal workers and autoscaling based on queue depth.

### Monitoring

Latency, errors, OCR success, extraction quality and drift.

### Security

Authentication, authorization, encryption, secrets management and audit
logs.

# 45. Rapid-fire questions

## Python

1.  List vs tuple?
2.  Set vs dict?
3.  Mutable vs immutable?
4.  Generator?
5.  Decorator?
6.  Context manager?
7.  Shallow vs deep copy?
8.  Big-O of dictionary lookup?
9.  Why vectorization?
10. Pandas vs Spark?

## SQL

1.  WHERE vs HAVING?
2.  INNER vs LEFT JOIN?
3.  Window function?
4.  ROW_NUMBER vs RANK?
5.  CTE?
6.  Find duplicates?
7.  Top N per group?
8.  NULL handling?
9.  SQL injection?
10. Prevent leakage in feature SQL?

## ML

1.  Bias/variance?
2.  Overfitting?
3.  Regularization?
4.  Precision/recall?
5.  ROC-AUC?
6.  PR-AUC?
7.  Cross-validation?
8.  Leakage?
9.  Drift?
10. Calibration?

## Spark

1.  Driver vs executor?
2.  Transformation vs action?
3.  Lazy evaluation?
4.  Shuffle?
5.  Partition?
6.  Broadcast join?
7.  Data skew?
8.  Repartition vs coalesce?
9.  Why avoid unnecessary UDFs?
10. Optimize a slow job?

## Cloud

1.  VM?
2.  VNet?
3.  Subnet?
4.  NSG?
5.  RBAC?
6.  Docker?
7.  Image vs container?
8.  API Gateway?
9.  Load balancer?
10. p95/p99?

## GenAI

1.  RAG?
2.  Hybrid retrieval?
3.  BM25?
4.  RRF?
5.  Cross-encoder?
6.  MRR?
7.  Hallucination?
8.  Prompt injection?
9.  Agent vs RAG?
10. Agent safeguards?

# 46. Production-connection answer pattern

When asked theory, add:

> "In production, I'd also consider..."

Examples:

-   ML model → latency, monitoring, versioning, drift
-   SQL → grain, joins, leakage, performance
-   Spark → shuffles, skew, partitioning, execution plan
-   Docker → dependency pinning, security, secrets
-   Azure → identity, networking, scaling, monitoring
-   RAG → retrieval vs generation evaluation, latency and cost

# 47. Final preparation priorities

### Tier 1

-   Python coding
-   SQL
-   ML fundamentals
-   project deep dives
-   metrics
-   leakage
-   system design

### Tier 2

-   statistics
-   PySpark
-   Docker
-   deployment
-   Azure basics
-   Databricks

### Tier 3

-   RAG evaluation
-   agent architecture
-   hybrid retrieval
-   reranking
-   LLM safeguards

### Low priority

-   Tableau/PowerBI
-   obscure algorithms
-   deep Kubernetes administration
-   deep Azure service catalog
-   memorizing research papers

# 48. Per-project checklist

For every project, prepare:

-   [ ] Problem
-   [ ] Users/stakeholders
-   [ ] Architecture
-   [ ] Data flow
-   [ ] Your exact contribution
-   [ ] Model choice
-   [ ] Retrieval choice
-   [ ] Evaluation
-   [ ] Failure modes
-   [ ] Improvements
-   [ ] Latency
-   [ ] Scalability
-   [ ] Deployment
-   [ ] Monitoring
-   [ ] Security
-   [ ] Difficult technical decision
-   [ ] Mistake/lesson
-   [ ] Future improvement

# 49. Final interview mindset

Do not aim to memorize every question. Aim to reason from:

``` text
Business problem → data → model → engineering → deployment → monitoring
```

That is the core Engineer-II mindset.
