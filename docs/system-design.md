# IntelliDocs AI — System Design

## 1. Purpose

This document describes the target system architecture for IntelliDocs AI, an enterprise document intelligence platform. It is an architectural plan, not a statement that every component has already been implemented.

## 2. Product scope

IntelliDocs AI helps organizations upload, manage, classify, search, understand, review, and route business documents. Initial document types include PDFs, DOCX files, and TXT files. Later releases can add scanned documents, spreadsheets, and additional formats.

Core capabilities:

- Document upload, storage, metadata, preview, download, and deletion.
- Text extraction, normalization, chunking, and indexing.
- AI classification, summaries, and structured metadata extraction.
- Semantic search and retrieval-augmented generation (RAG).
- Source citations linked to documents and pages/passages.
- Organization management, permissions, versions, and workflows in later phases.
- Integrations with QMS, ERP, CRM, and external DMS platforms.

## 3. Architectural principles

1. Start with a modular monolith, not a fleet of microservices.
2. Keep frontend, API routes, domain services, persistence, document processing, and AI provider adapters separate.
3. Run long-running extraction and AI work asynchronously.
4. Enforce authorization before retrieval, download, and AI processing.
5. Treat uploaded files and extracted content as untrusted input.
6. Make processing idempotent and observable.
7. Keep provider-specific code behind adapters.
8. Introduce distributed infrastructure only when a measured requirement justifies it.
9. Preserve source metadata throughout extraction, chunking, retrieval, and citation.
10. Document significant architectural decisions in `docs/adr/`.

## 4. Phase 1 logical architecture

```text
Next.js (JavaScript)
       |
       | HTTPS / REST / JSON
       v
Flask application (application factory + Blueprints)
       |
       +--> Authentication and user services
       +--> Document management services
       +--> Search and retrieval services
       +--> AI provider adapters
       +--> Processing-job management
       |
       +--> PostgreSQL + pgvector
       +--> Local development file storage
       +--> Celery workers + Redis (introduced with async processing)
```

The frontend must use JavaScript `.js` and `.jsx` files only. TypeScript `.ts` and `.tsx` files are not permitted.

## 5. Phase evolution

### Phase 1 — Basic working MVP

- One Next.js application.
- One Flask application using Blueprints and service modules.
- PostgreSQL as the primary database.
- pgvector for embeddings.
- Local file storage for development.
- Celery and Redis for background processing.
- OpenAI and/or Anthropic APIs called only by the backend.
- Polling for job status initially.

### Phase 2 — Enterprise-ready application

- Organization and membership model.
- Tenant-scoped authorization and RBAC.
- Document versions and controlled lifecycle.
- OCR, hybrid search, reranking, and structured extraction.
- QMS review/approval workflows.
- Initial external integrations.
- Object storage such as S3-compatible storage.
- Better observability, backups, and staging deployment.

### Phase 3 — Distributed platform

Potential additions, only when justified:

- API gateway and load balancing.
- Kafka event backbone.
- Independently scalable Python workers/services.
- AI gateway and provider routing.
- Kubernetes and Helm.
- OpenTelemetry, Prometheus, and Grafana.
- Qdrant and/or OpenSearch if benchmarks justify them.
- Database partitioning or tenant sharding when measured requirements demand it.

## 6. Key data flow: document ingestion

1. Authenticated user submits a document.
2. API validates file size, extension, MIME type, and ownership context.
3. File is stored outside the relational database.
4. Document and processing-job records are created.
5. A background task extracts text.
6. Text is normalized and chunked while preserving page/section metadata.
7. Embeddings are generated in batches.
8. Chunks and vectors are persisted.
9. AI metadata and summary are generated and stored.
10. Document status becomes completed or failed.
11. Frontend retrieves status and displays the result.

For later distributed ingestion, replace direct task dispatch with durable events and object-storage notifications as appropriate.

## 7. Key data flow: RAG question answering

1. Authenticate the user.
2. Resolve the user's authorized document scope on the server.
3. Normalize and embed the query.
4. Retrieve relevant chunks only from authorized documents.
5. Optionally rerank retrieved passages.
6. Send the question and bounded evidence to the selected LLM.
7. Validate returned citation references against retrieved records.
8. Persist the conversation and return answer plus citations.

The LLM must never decide whether a user is authorized to access a document.

## 8. Reliability and scaling

Phase 1 should already use:

- Request timeouts.
- File-size limits.
- Structured logs and request IDs.
- Background jobs for long tasks.
- Idempotent processing keys.
- Database transactions for state changes.
- Retry handling for transient provider failures.
- Clear job states and failure messages.

Phase 3 may add queue backpressure, dead-letter queues, replay, consumer groups, autoscaling, and reconciliation.

## 9. Non-goals for the MVP

- Kubernetes.
- Kafka.
- Microservices for every module.
- Multi-region deployment.
- Claims of million concurrent users.
- Autonomous approval of controlled documents.
- Universal integrations with every ERP/QMS/CRM product.

## 10. Architecture status

This is the intended architecture. Update this document when implementation decisions materially change, and record significant changes in an ADR.
