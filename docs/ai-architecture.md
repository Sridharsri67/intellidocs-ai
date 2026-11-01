# IntelliDocs AI — AI Architecture

## 1. Purpose

Describe the AI capabilities, provider abstraction, processing pipeline, retrieval design, and safety controls for IntelliDocs AI.

## 2. AI capabilities by phase

### Phase 1

- Document classification suggestions.
- Document summaries.
- Text embeddings.
- Semantic retrieval.
- RAG question answering.
- Verifiable source citations.

### Phase 2

- OCR-assisted understanding.
- Schema-validated structured extraction.
- Hybrid retrieval and reranking.
- Version-aware indexing.
- Evaluation datasets and quality metrics.
- Model routing and tenant usage budgets.
- Human review for uncertain or high-impact outputs.

### Phase 3

- Dedicated AI Gateway.
- Independently scalable AI and embedding workers.
- Provider-aware concurrency and quotas.
- Cost/latency policies.
- Distributed tracing and AI workload observability.

## 3. Provider abstraction

Support OpenAI and Anthropic through backend-only adapters. Keep provider-specific request/response details out of route handlers and domain services.

A conceptual internal interface may expose operations such as:

- `generate_text(...)`
- `generate_structured_output(...)`
- `create_embeddings(...)`

The exact interface should be implemented only when the first provider integration is added.

Provider selection should consider task suitability, quality, latency, cost, context limits, data handling, and availability. Do not assume every provider supports identical features.

## 4. Document processing pipeline

```text
File
 |
 v
Validation and storage
 |
 v
Text extraction
 |
 v
Normalization and page mapping
 |
 v
Chunking
 |
 +--> Classification / metadata extraction
 |
 +--> Summary generation
 |
 v
Embedding generation
 |
 v
Vector and metadata persistence
 |
 v
Searchable document
```

### Extraction

- PDF: PyMuPDF for text-based PDFs.
- DOCX: python-docx.
- TXT: controlled text decoding.
- Scanned PDF/image OCR: Phase 2 unless an MVP requirement proves otherwise.
- Spreadsheet parsing: later phase using an appropriate library such as openpyxl.

Preserve source page, section, and document-version metadata whenever available.

### Chunking

Chunking strategy must be configurable and evaluated. Store:

- Document ID and version.
- Chunk ID and sequence.
- Extracted text.
- Page/section reference.
- Character/token offsets where practical.
- Embedding model and dimension.
- Processing version.

Avoid splitting in ways that destroy important table, heading, or page context. Do not hardcode a chunk size without testing representative documents.

### Embeddings

- Generate embeddings in batches where supported.
- Record the embedding model and vector dimension.
- Do not mix incompatible embedding dimensions in the same vector column/index.
- Re-embedding may be required when the model changes.
- Use retry and rate-limit handling.
- Do not generate duplicate embeddings for the same processing version.

## 5. RAG retrieval flow

1. Authenticate user.
2. Resolve authorized documents on the server.
3. Embed the query using a compatible embedding model.
4. Retrieve relevant chunks within the authorized scope.
5. Apply metadata filters.
6. Optionally rerank results in Phase 2.
7. Construct a bounded evidence context.
8. Ask the LLM to answer only from supplied evidence.
9. Validate citations against retrieved chunk/document records.
10. Return answer and source metadata.

### Citation requirements

Each citation should resolve to a real document and source passage/page. Never trust an LLM-generated document ID or page number without validation. If evidence is insufficient, the model should say so rather than fabricate an answer.

## 6. Prompt and content security

Documents are untrusted data. A document may contain instructions intended to manipulate an AI system.

- Delimit retrieved document content as evidence, not instructions.
- Never allow document content to change system policy.
- Never let an LLM grant access to a document.
- Keep authorization outside the model.
- Validate structured outputs.
- Restrict tools and actions to explicit allowlists.
- Require human authorization for controlled workflow actions.
- Avoid sending unnecessary sensitive content to external providers.

## 7. AI gateway evolution

Phase 1 can use provider adapters inside the Flask application. Phase 3 may introduce a dedicated AI Gateway responsible for:

- Provider/model routing.
- API key and credential management.
- Tenant budgets and quotas.
- Rate limits and concurrency.
- Retry/fallback policy.
- Token and cost accounting.
- Provider latency and error metrics.
- Caching where safe and semantically valid.

Do not create a separate gateway service in Phase 1 unless a concrete requirement appears.

## 8. Evaluation

Create a small representative evaluation corpus. Track:

- Retrieval precision/recall or appropriate retrieval metrics.
- Answer groundedness.
- Citation correctness.
- Unsupported-answer rate.
- Structured extraction validity.
- Latency.
- Token usage and cost.
- Failure and retry rates.

Use deterministic tests with mocked providers for core CI. Run real model evaluations separately.

## 9. AI data lifecycle

Define what is stored for each AI result:

- Provider and model.
- Prompt/template version.
- Input document/version references.
- Output and structured metadata.
- Token usage where available.
- Created timestamp.
- Processing status and errors.

Apply retention and deletion rules consistently to summaries, embeddings, chunks, and derived data when a document is deleted or retention expires.
