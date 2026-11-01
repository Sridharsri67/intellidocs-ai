# IntelliDocs AI — Database Design

> **Status:** Logical design for the phased product. The schema must be implemented through migrations and refined as requirements become concrete.

## 1. Database technology

- PostgreSQL is the primary relational database.
- pgvector is the initial vector-search extension.
- SQLAlchemy/Flask-SQLAlchemy is the ORM.
- Flask-Migrate/Alembic manages schema migrations.
- Store file contents in filesystem/object storage, not as large database blobs.

## 2. Phase 1 entities

### `users`

Suggested fields:

- `id` — UUID primary key.
- `email` — unique, normalized.
- `password_hash` — secure password hash.
- `is_active` — account status.
- `created_at`, `updated_at`.

### `documents`

Suggested fields:

- `id` — UUID primary key.
- `owner_id` — foreign key to users.
- `title` — display title.
- `original_filename` — sanitized display metadata.
- `file_key` — server-generated storage key.
- `mime_type`.
- `file_size`.
- `checksum` — optional integrity/deduplication aid.
- `status` — processing lifecycle.
- `created_at`, `updated_at`, `deleted_at` where soft deletion is selected.

Never use an untrusted filename as a filesystem path.

### `document_chunks`

Suggested fields:

- `id`.
- `document_id`.
- `chunk_index`.
- `text`.
- `page_number` or source location.
- `section_title` where available.
- `embedding` — pgvector vector with a dimension matching the selected embedding model.
- `embedding_model`.
- `created_at`.

Preserve source references so citations can be validated.

### `document_tags`

Suggested fields:

- `id`.
- `document_id`.
- `tag`.

A normalized tag table may be preferable if tags need organization-wide management.

### `document_summaries`

Suggested fields:

- `id`.
- `document_id`.
- `summary`.
- `provider`.
- `model`.
- `prompt_version`.
- `created_at`.

### `chat_sessions`

Suggested fields:

- `id`.
- `user_id`.
- `title`.
- `created_at`, `updated_at`.

### `chat_messages`

Suggested fields:

- `id`.
- `session_id`.
- `role`.
- `content`.
- `created_at`.

Citations may be stored in a structured JSON field or a separate normalized table, depending on query needs.

### `processing_jobs`

Suggested fields:

- `id`.
- `document_id`.
- `status`.
- `stage`.
- `attempts`.
- `idempotency_key`.
- `error_code`.
- `safe_error_message`.
- `started_at`, `finished_at`, `created_at`.

### `audit_logs`

Suggested fields:

- `id`.
- `actor_id`.
- `action`.
- `resource_type`.
- `resource_id`.
- `metadata` — carefully controlled JSON.
- `created_at`.

Do not put secrets or full sensitive document contents into audit metadata.

## 3. Phase 2 entities

Add as required:

- `organizations`
- `organization_members`
- `roles`
- `permissions`
- `role_permissions`
- `document_versions`
- `workflow_definitions`
- `workflow_instances`
- `workflow_tasks`
- `integration_connections`
- `usage_records`
- notification-related tables

Tenant-owned records should include a tenant/organization reference. Enforce membership and authorization server-side. PostgreSQL Row-Level Security may be added as defense in depth.

## 4. Relationships (conceptual)

```text
users 1---N documents
users 1---N chat_sessions
documents 1---N document_chunks
documents 1---N document_summaries
documents 1---N processing_jobs
documents 1---N document_tags
chat_sessions 1---N chat_messages
users 1---N audit_logs

Phase 2:
organizations 1---N organization_members
organizations 1---N documents
documents 1---N document_versions
workflow_instances 1---N workflow_tasks
```

Exact cardinalities and join tables should be finalized during model implementation.

## 5. Indexing guidance

Consider indexes for:

- Unique normalized user email.
- Document owner and creation time.
- Document status and creation time.
- Foreign keys used in joins.
- Processing job status and creation time.
- Chat session owner and updated time.
- Tag lookup.
- Vector similarity index after choosing embedding dimension and evaluating query patterns.

Do not add every possible index blindly. Measure query plans and write overhead.

## 6. Vector data requirements

- Select an embedding model before creating the production vector column.
- Record model identity and dimension.
- Ensure query and document vectors use compatible models.
- Plan a re-embedding strategy for model changes.
- Filter by authorization scope before or as part of vector retrieval.
- Keep document/version references on every chunk.

## 7. Deletion and retention

Deleting a document must remove or invalidate:

- Original file or storage object according to retention policy.
- Extracted chunks.
- Embeddings.
- Summaries.
- Search index entries.
- Derived references and caches where appropriate.

Phase 2 adds retention, legal hold, and version lifecycle requirements. Do not hard-delete legally retained records.

## 8. Migration policy

- Every schema change must have a migration.
- Review generated migrations before applying.
- Do not edit production schema manually.
- Test migrations against a representative database.
- Document data migrations and rollback limitations.
- Avoid destructive migrations without a backup and migration plan.
