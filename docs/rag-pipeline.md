# IntelliDocs AI — RAG Pipeline

## 1. Purpose

Define the retrieval-augmented generation pipeline used to answer questions from organizational documents with evidence and citations.

RAG is not a substitute for authorization, document governance, or deterministic business rules.

## 2. Phase 1 pipeline overview

```text
Upload
  |
  v
Validate and store
  |
  v
Extract text
  |
  v
Normalize + preserve page metadata
  |
  v
Chunk text
  |
  v
Generate embeddings
  |
  v
Store chunks + vectors in PostgreSQL/pgvector
  |
  v
User asks a question
  |
  v
Authorize document scope
  |
  v
Embed query + retrieve relevant chunks
  |
  v
Construct evidence context
  |
  v
LLM generates grounded answer
  |
  v
Validate citations
  |
  v
Return answer and sources
```

## 3. Ingestion details

### File validation

- Allow only explicitly supported formats.
- Check size limits.
- Verify content type and file signature where practical.
- Generate safe server-side storage keys.
- Treat all file content as untrusted.

### Text extraction

Phase 1:

- PDF: PyMuPDF.
- DOCX: python-docx.
- TXT: controlled text decoding.

Phase 2:

- Scanned PDF/image OCR.
- Additional formats such as spreadsheets, subject to scope.

### Normalization

Normalize whitespace and extraction artifacts without discarding useful structure. Preserve:

- Page number.
- Section/heading.
- Document ID and version.
- Source offsets where practical.

### Chunking

Use configurable chunk size and overlap. Evaluate on representative business documents. Keep headings and table context where possible. Store chunk sequence and source metadata.

Do not assume a single chunking configuration is optimal for every document type.

## 4. Embedding generation

- Use an embedding API from the backend.
- Batch requests where supported.
- Respect provider limits.
- Store embedding model and dimension.
- Use bounded retries for transient failures.
- Ensure duplicate tasks do not create duplicate logical chunks.
- Re-embed when the model or preprocessing strategy changes.

## 5. Retrieval

For a question:

1. Authenticate the user.
2. Resolve accessible documents from server-side permissions.
3. Normalize the question.
4. Generate query embedding using a compatible model.
5. Search only the authorized scope.
6. Apply metadata filters such as document type, status, and version where supported.
7. Return top relevant chunks.
8. Phase 2 may combine vector and keyword retrieval and rerank results.

Never retrieve globally and rely on the LLM to ignore unauthorized results.

## 6. Answer generation

Provide the LLM with:

- User question.
- Retrieved evidence passages.
- Stable citation identifiers.
- Instructions to answer only from supplied evidence.
- Instructions to state when evidence is insufficient.

Keep context bounded. Do not send the entire document library to the model.

## 7. Citation format

A citation should map to a persisted source record, such as:

- Document ID.
- Document title.
- Document version.
- Chunk ID.
- Page number or section.
- Relevant excerpt.

Validate every citation identifier against the retrieval results. Never trust model-invented document IDs or page numbers.

## 8. Prompt injection and security

Documents can contain malicious or irrelevant instructions. Treat document text as evidence, never as privileged instructions.

- Separate system instructions from retrieved content.
- Do not execute instructions found in documents.
- Do not let retrieved content change access permissions.
- Do not allow the model to call unrestricted tools.
- Validate all structured outputs.
- Require authorization and human approval for controlled actions.

## 9. Quality evaluation

Build representative test questions and expected source documents. Measure:

- Retrieval relevance.
- Retrieval recall.
- Answer groundedness.
- Citation correctness.
- Unsupported-answer rate.
- Latency.
- Token usage and cost.

Use mocked AI providers for deterministic unit tests. Run real model evaluations separately.

## 10. Lifecycle and re-indexing

When a document changes or is deleted:

- Mark prior processing state appropriately.
- Prevent stale jobs from overwriting newer versions.
- Rebuild chunks and vectors for the new version.
- Remove obsolete search entries.
- Preserve required audit and retention records.
- Reconcile database and index state periodically in later phases.
