# ClauseAI — AI Contract Intelligence Platform

ClauseAI is an AI-powered contract intelligence platform that analyzes legal agreements using **RAG, embeddings, semantic search, structured LLM extraction, contract comparison, risk analysis, and policy/compliance checks**.

The system should be implemented as a production-style application rather than a simple "upload PDF → send to LLM → get summary" application.

---

# 1. Core Objective

Build a platform where a user can:

1. Upload one or more legal contracts.
2. Extract text from PDFs, including scanned PDFs through OCR.
3. Split documents into semantically meaningful chunks.
4. Generate embeddings for chunks.
5. Store vectors in Qdrant.
6. Store document and extracted-structure data in MongoDB.
7. Ask questions about uploaded contracts.
8. Receive answers grounded in retrieved contract sections.
9. Compare two contract versions semantically.
10. Detect potentially risky contractual provisions.
11. Check contracts against configurable policies.
12. Generate an explainable contract risk report.
13. Display source pages/sections supporting AI conclusions.

---

# 2. Target Technology Stack

## Frontend

* Next.js
* React
* TypeScript
* Tailwind CSS

Responsibilities:

* Authentication UI
* Document upload
* Document library
* Contract viewer
* Chat / contract Q&A
* Side-by-side contract comparison
* Risk dashboard
* Clause-level risk indicators
* Compliance/policy results
* AI-generated reports

---

## Backend

* FastAPI
* Python

Responsibilities:

* REST APIs
* Document processing
* OCR
* Chunking
* Embedding generation
* Qdrant interaction
* MongoDB interaction
* AI orchestration
* Contract comparison
* Risk analysis
* Policy evaluation

---

## AI Orchestration

* LangChain

Use LangChain for:

* Retrieval pipelines
* Prompt management
* RAG orchestration
* Structured LLM calls
* Multi-step analysis workflows
* Agent/tool orchestration where appropriate

Do NOT use LangChain merely as a wrapper around every function.

---

## LLM Gateway

* OpenRouter

OpenRouter is the model gateway.

Do NOT describe OpenRouter itself as an AI technique.

Architecture:

```text
Application
    ↓
LangChain
    ↓
OpenRouter
    ↓
Selected LLM
```

The application should keep the model provider configurable.

Do not hard-code application logic around one specific model.

---

## Vector Database

* Qdrant

Qdrant stores:

* Embeddings
* Original chunk text or chunk reference
* Document ID
* Page number
* Section number
* Clause type
* Other retrieval metadata

---

## Database

* MongoDB

MongoDB stores:

* Users
* Documents
* Document metadata
* Processing status
* Extracted clauses
* Structured contract information
* Risk assessments
* Policy definitions
* Analysis results
* Chat history
* Report metadata

Qdrant and MongoDB have different responsibilities.

```text
MongoDB
→ application/document data

Qdrant
→ semantic vector retrieval
```

---

# 3. High-Level Architecture

```text
                         ┌──────────────────────┐
                         │      Next.js UI       │
                         │ React + TypeScript    │
                         └──────────┬───────────┘
                                    │
                              REST / HTTP
                                    │
                                    ▼
                         ┌──────────────────────┐
                         │     FastAPI Backend   │
                         └──────────┬───────────┘
                                    │
                 ┌──────────────────┼──────────────────┐
                 │                  │                  │
                 ▼                  ▼                  ▼
        Document Pipeline      AI Orchestrator    Contract Engine
                 │                  │                  │
                 ▼                  ▼                  ▼
          PDF / OCR / Text       LangChain        Comparison
          Extraction             Workflows        Engine
                 │                  │                  │
                 ▼                  │                  ▼
          Chunking + Metadata      │             Semantic
                 │                  │             Comparison
                 ▼                  │
          Embedding Model          │
                 │                  │
                 ▼                  │
              Qdrant ◄─────────────┘
                 │
          Relevant Chunks
                 │
                 ▼
              RAG Context
                 │
                 ▼
             OpenRouter
                 │
                 ▼
                LLM
                 │
       ┌─────────┼─────────┐
       ▼         ▼         ▼
    Clause     Risk     Compliance
   Analysis   Analysis    Engine
       │         │         │
       └─────────┼─────────┘
                 ▼
          Structured Results
                 │
                 ▼
              MongoDB
                 │
                 ▼
             Next.js UI
```

---

# 4. Document Ingestion Pipeline

When a user uploads a document:

```text
PDF
 ↓
File Validation
 ↓
PDF Text Extraction
 ↓
OCR fallback for scanned pages
 ↓
Text Cleaning
 ↓
Section Detection
 ↓
Semantic Chunking
 ↓
Metadata Generation
 ↓
Embedding Generation
 ↓
Qdrant
 ↓
MongoDB
```

The system must NOT simply split every document into arbitrary fixed-size strings.

Chunks should preserve useful legal context.

Each chunk should have metadata similar to:

```json
{
  "document_id": "...",
  "page_number": 14,
  "section": "8.2",
  "section_title": "Termination",
  "chunk_index": 42,
  "clause_type": "termination"
}
```

---

# 5. Where Chunked Data Goes

This distinction is critical.

After chunking:

```text
Chunk
 ↓
Embedding Model
 ↓
Vector
 ↓
Qdrant
```

Store the chunk text and metadata either directly in the Qdrant payload or maintain a reference to MongoDB.

Recommended structure:

```text
Qdrant
 ├── vector
 ├── chunk_id
 ├── document_id
 ├── page_number
 ├── section
 └── metadata

MongoDB
 └── original structured document/chunk data
```

Do not lose the connection between:

```text
vector
    ↓
chunk
    ↓
page
    ↓
section
    ↓
original document
```

This is required for citations and explainability.

---

# 6. RAG Architecture

RAG must be implemented as an actual retrieval pipeline.

```text
User Question
      ↓
LangChain
      ↓
Query Embedding
      ↓
Qdrant Similarity Search
      ↓
Top-K Relevant Chunks
      ↓
Optional Metadata Filtering
      ↓
Context Construction
      ↓
Prompt
      ↓
OpenRouter
      ↓
LLM
      ↓
Structured Answer
```

The answer must be grounded in retrieved document content.

The system should return source information such as:

```text
Answer:
The agreement can be terminated with 30 days' notice.

Sources:
Page 14
Section 8.2 — Termination
```

If the required information cannot be found, the system should explicitly state that the contract does not contain sufficient information rather than inventing an answer.

---

# 7. Embeddings

Embeddings should be used for semantic retrieval and contract comparison.

Example:

```text
"Either party may terminate with 30 days notice."

        ↓

[0.021, -0.182, 0.731, ...]
```

The exact embedding provider should be configurable.

Do not hard-code embedding logic throughout the application.

Create an abstraction such as:

```python
EmbeddingService
```

Responsibilities:

* Generate embeddings
* Batch embeddings where possible
* Handle failures
* Expose a consistent interface to the rest of the application

---

# 8. Qdrant

Create a dedicated vector service.

Example responsibility:

```python
VectorStoreService
```

Responsibilities:

* Create collection
* Insert vectors
* Delete document vectors
* Search vectors
* Filter by document ID
* Filter by metadata
* Return top-K relevant chunks

Example retrieval:

```text
query
 ↓
embedding
 ↓
Qdrant
 ↓
top_k = 5
 ↓
relevant chunks
```

Never expose raw Qdrant implementation details throughout the application.

---

# 9. Structured Clause Extraction

After document ingestion, extract important contractual information.

Potential clause categories:

* Parties
* Effective date
* Term
* Termination
* Payment
* Liability
* Indemnification
* Confidentiality
* Intellectual property
* Data protection
* Governing law
* Jurisdiction
* Renewal
* Dispute resolution
* Force majeure
* SLA
* Non-compete
* Non-solicitation

The LLM should return structured output.

Example:

```json
{
  "clause_type": "liability",
  "text": "The supplier's liability shall be unlimited...",
  "page": 22,
  "risk_level": "high",
  "reason": "No monetary liability cap is specified."
}
```

Use schema validation.

Do not depend on unstructured LLM text parsing whenever structured output is possible.

---

# 10. Contract Comparison Engine

The comparison system should compare two versions semantically.

Do NOT implement this as simple string diffing.

Pipeline:

```text
Contract A
    ↓
Clause Extraction
    ↓
Embeddings
    ↓
Contract B
    ↓
Clause Extraction
    ↓
Embeddings
    ↓
Semantic Matching
    ↓
Change Classification
```

Classify changes as:

```text
UNCHANGED
MODIFIED
ADDED
REMOVED
```

The system should identify meaningful changes even when wording changes substantially.

Example:

```text
Version A:
Liability capped at $1M.

Version B:
Liability shall be unlimited.

        ↓

Semantic Change

        ↓

Risk Impact: HIGH
```

---

# 11. Risk Engine

The risk engine should combine multiple signals.

Do NOT make the entire risk score a random LLM-generated number.

Use a hybrid approach:

```text
             ┌────────────────────┐
             │ Clause Information  │
             └─────────┬──────────┘
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
       Rules       LLM Analysis   Anomaly
       Engine                       Detection
          │            │            │
          └────────────┼────────────┘
                       ▼
                  Risk Engine
                       │
                       ▼
                 Risk Severity
```

Risk levels:

```text
LOW
MEDIUM
HIGH
CRITICAL
```

Possible risk signals:

* Unlimited liability
* Missing termination clause
* Unfavorable termination terms
* Automatic renewal
* Long payment periods
* Broad indemnification
* Unusual jurisdiction
* Data protection obligations
* Excessive penalties
* Missing SLA
* Significant version changes

Every risk result should contain:

```text
Risk level
Reason
Evidence
Page
Section
Relevant clause
```

---

# 12. Compliance / Policy Engine

Users should be able to define configurable contract policies.

Example:

```text
Policy:
Liability must not exceed $1M

Contract:
Unlimited liability

Result:
FAIL

Risk:
HIGH

Evidence:
Section 12.3
```

Another example:

```text
Policy:
Termination notice >= 60 days

Contract:
90 days

Result:
PASS
```

Policy evaluation should use deterministic rules whenever possible.

Use the LLM for extracting and interpreting contractual language, but do not use an LLM when a simple deterministic comparison is sufficient.

---

# 13. Agentic Workflow

The system can use specialized analysis stages.

```text
                    Contract
                       │
                       ▼
              Analysis Orchestrator
                       │
       ┌───────────────┼────────────────┐
       ▼               ▼                ▼
 Clause Analysis    Risk Analysis   Compliance
       │               │                │
       └───────────────┼────────────────┘
                       ▼
                 Synthesis Step
                       │
                       ▼
                Final Report
```

Possible specialized components:

### Clause Analysis

Extract:

* clauses
* obligations
* parties
* dates
* monetary values

### Risk Analysis

Identify:

* risky provisions
* anomalies
* unusual terms
* risk explanations

### Compliance Analysis

Evaluate:

* company policies
* contractual thresholds
* required clauses
* policy violations

### Synthesis

Combine outputs into:

* executive summary
* risk report
* key findings
* recommended areas for human review

The system should remain human-in-the-loop.

It is a decision-support system, not an autonomous legal decision maker.

---

# 14. OpenRouter Integration

OpenRouter is the LLM gateway.

Architecture:

```text
LangChain
    ↓
OpenRouter
    ↓
Configured Model
```

Create an abstraction:

```python
LLMService
```

The model should be configurable through environment variables.

Example:

```env
OPENROUTER_API_KEY=
OPENROUTER_MODEL=
```

Do not hard-code API keys.

Do not hard-code one specific model into business logic.

The application should be able to switch models without rewriting the analysis pipeline.

---

# 15. MongoDB Data Model

Suggested collections:

```text
users
documents
document_chunks
clauses
analyses
risk_assessments
policies
comparisons
chat_sessions
chat_messages
reports
```

Example document:

```json
{
  "_id": "...",
  "filename": "agreement.pdf",
  "status": "processed",
  "uploaded_at": "...",
  "page_count": 32,
  "processing_version": "1.0"
}
```

---

# 16. API Structure

Use clear REST endpoints.

Suggested APIs:

```text
POST   /api/documents/upload
GET    /api/documents
GET    /api/documents/{id}
DELETE /api/documents/{id}

POST   /api/documents/{id}/process

POST   /api/chat
GET    /api/chat/{id}/history

POST   /api/contracts/compare

POST   /api/contracts/{id}/analyze

GET    /api/contracts/{id}/risks

POST   /api/policies
GET    /api/policies
POST   /api/contracts/{id}/compliance

GET    /api/reports/{id}
```

Keep API logic separate from AI/business logic.

---

# 17. Error Handling

The system should gracefully handle:

* Invalid PDFs
* Empty documents
* OCR failures
* Unsupported file formats
* Embedding API failures
* Qdrant unavailable
* MongoDB unavailable
* LLM timeout
* LLM malformed output
* Rate limits
* Missing retrieved context

LLM failures should not crash the entire application.

Implement:

* retries where appropriate
* timeouts
* structured errors
* logging
* validation

---

# 18. Security

Implement basic production-style security.

Requirements:

* Never expose API keys to the frontend
* Store secrets in environment variables
* Validate uploaded files
* Restrict file types
* Limit file size
* Validate API requests
* Sanitize user-provided metadata
* Ensure users can only access their own documents
* Do not expose raw internal errors to users

---

# 19. Frontend Pages

Recommended UI:

```text
/
├── Dashboard
├── Documents
├── Upload
├── Contract Viewer
├── Contract Q&A
├── Compare Contracts
├── Risk Dashboard
├── Compliance
└── Reports
```

Contract viewer should display:

```text
┌──────────────────────┬─────────────────────────┐
│ Contract             │ AI Analysis             │
│                      │                         │
│ Section 8.2          │ Risk: HIGH              │
│                      │                         │
│ Termination clause   │ Why?                    │
│                      │ Unlimited liability...  │
│                      │                         │
│                      │ Source: Page 14         │
└──────────────────────┴─────────────────────────┘
```

---

# 20. AI Explainability

Every important AI-generated conclusion should have evidence.

Bad:

```text
Risk = HIGH
```

Good:

```text
Risk = HIGH

Reason:
The agreement does not impose a monetary cap on supplier liability.

Evidence:
Section 12.3
Page 22
```

The system should make it easy for the user to navigate back to the original clause.

---

# 21. Evaluation

Do not claim that the AI works well without evaluating it.

Create a small evaluation dataset containing contracts and expected answers.

Measure:

### Retrieval

* Recall@K
* Precision@K

### Extraction

* Precision
* Recall
* F1

### Q&A

* Groundedness
* Citation accuracy
* Answer correctness

### Risk detection

* Precision
* Recall
* False-positive rate

The goal is to demonstrate that the system is evaluated rather than judged only by subjective output quality.

---

# 22. Important Engineering Rules

1. Do not send the entire document to the LLM for every question.
2. Use RAG for document Q&A.
3. Keep embeddings separate from LLM generation.
4. Qdrant is for vector retrieval.
5. MongoDB is for application/document data.
6. LangChain orchestrates AI workflows; it is not the vector database.
7. OpenRouter is the model gateway.
8. Use structured output for clause extraction.
9. Use deterministic rules for deterministic policy checks.
10. Every risk result should have evidence.
11. Never expose API keys in Next.js.
12. Keep AI services modular.
13. Do not hard-code a specific LLM throughout the application.
14. Handle LLM failures and malformed outputs.
15. Keep humans in the loop for legal/risk decisions.

---

# 23. Final Target Flow

The final implementation should support this complete flow:

```text
USER
 │
 ▼
Next.js
 │
 ▼
FastAPI
 │
 ▼
Document Processing
 │
 ├── PDF Extraction
 ├── OCR
 ├── Cleaning
 └── Semantic Chunking
 │
 ▼
Embedding Model
 │
 ▼
Qdrant
 │
 │
 ├──────────────────────────────┐
 │                              │
 ▼                              ▼
RAG Query                  Contract Analysis
 │                              │
 ▼                              ├── Clause Extraction
LangChain                       ├── Comparison
 │                              ├── Risk Analysis
 ▼                              └── Compliance
Qdrant Retrieval
 │
 ▼
Relevant Context
 │
 ▼
OpenRouter
 │
 ▼
LLM
 │
 ▼
Structured Output
 │
 ├── Answer
 ├── Sources
 ├── Risks
 ├── Compliance
 └── Recommendations for Human Review
 │
 ▼
MongoDB
 │
 ▼
Next.js Dashboard
```

# 24. Implementation Priority

Implement in this order:

### Phase 1

Document upload + PDF extraction + MongoDB.

### Phase 2

Chunking + embeddings + Qdrant.

### Phase 3

RAG + LangChain + OpenRouter.

### Phase 4

Structured clause extraction.

### Phase 5

Semantic contract comparison.

### Phase 6

Risk engine + policy/compliance engine.

### Phase 7

Agentic analysis workflow.

### Phase 8

Risk dashboard + citations + reports.

### Phase 9

Evaluation + error handling + security + performance improvements.

Do not implement all components as fake placeholders. Each feature should have a working backend flow and a corresponding frontend experience.
