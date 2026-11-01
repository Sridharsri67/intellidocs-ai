# IntelliDocs AI — API Reference (Planned Contract)

> **Status:** This document defines the intended Phase 1 API contract. An endpoint is not implemented merely because it appears here. Update implementation status as routes are built and tested.

## 1. API conventions

Base path: `/api/v1`

- Transport: HTTP(S).
- Standard payloads: JSON.
- File upload: multipart form data in the initial MVP.
- Authentication: protected endpoints require the selected backend authentication mechanism.
- All resource operations must enforce authorization.
- Collections should support pagination.
- Long-running processing returns a job/document identifier; clients initially poll for status.

## 2. Standard response shape

Success example:

```json
{
  "data": {
    "id": "resource-id"
  },
  "meta": {
    "request_id": "request-id"
  }
}
```

Error example:

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "The uploaded file type is not supported",
    "details": {}
  },
  "meta": {
    "request_id": "request-id"
  }
}
```

## 3. Health

### `GET /health`

Purpose: basic application health check. This endpoint may be public and should not reveal secrets or infrastructure details.

Example response:

```json
{
  "status": "healthy",
  "application": "IntelliDocs AI",
  "version": "1.0.0"
}
```

A separate readiness endpoint may check dependencies once database and worker infrastructure are configured.

## 4. Authentication

| Method | Endpoint | Purpose |
|---|---|---|
| POST | `/auth/register` | Create account |
| POST | `/auth/login` | Authenticate |
| POST | `/auth/logout` | End session |
| GET | `/auth/me` | Get current user |

Request/response fields must be finalized with the selected authentication mechanism. Never return password hashes.

## 5. Documents

| Method | Endpoint | Purpose |
|---|---|---|
| GET | `/documents` | List authorized documents |
| POST | `/documents` | Upload a document |
| GET | `/documents/{id}` | Get metadata and processing state |
| GET | `/documents/{id}/download` | Download authorized file |
| DELETE | `/documents/{id}` | Delete document and derived searchable data |

### Upload

Initial MVP: `multipart/form-data` with a file and optional title/tags.

Expected behavior:

1. Authenticate user.
2. Validate file size and supported format.
3. Generate a safe server-side storage key.
4. Store the file.
5. Create document and processing-job records.
6. Enqueue background processing when task infrastructure is available.
7. Return an accepted response with document/job identifiers.

Do not accept a client-provided storage path.

## 6. Processing jobs

| Method | Endpoint | Purpose |
|---|---|---|
| GET | `/jobs/{id}` | Get job status and safe progress information |

Suggested states:

- `queued`
- `processing`
- `completed`
- `failed`

Detailed pipeline stages may include validation, extraction, chunking, embedding, metadata generation, and indexing.

## 7. Search

### `POST /search`

Example request:

```json
{
  "query": "latest equipment calibration procedure",
  "limit": 10
}
```

Example response shape:

```json
{
  "data": {
    "results": [
      {
        "document_id": "document-id",
        "chunk_id": "chunk-id",
        "title": "Equipment Calibration SOP",
        "excerpt": "Relevant source passage...",
        "page_number": 4,
        "score": 0.87
      }
    ]
  },
  "meta": {
    "request_id": "request-id"
  }
}
```

Scores are illustrative. Actual scoring and fields must match the retrieval implementation. Search must enforce authorization before returning results.

## 8. Chat

| Method | Endpoint | Purpose |
|---|---|---|
| POST | `/chat/sessions` | Create chat session |
| GET | `/chat/sessions` | List user's sessions |
| GET | `/chat/sessions/{id}/messages` | Get session messages |
| POST | `/chat/sessions/{id}/messages` | Ask a question |

Example request:

```json
{
  "message": "What are the calibration requirements?"
}
```

Response should contain:

- Assistant answer.
- Validated citations.
- Session/message identifiers.
- Safe metadata such as model name where appropriate.

Do not expose hidden prompts, provider credentials, or internal chain-of-thought.

## 9. HTTP status conventions

| Status | Meaning |
|---|---|
| 200 | Successful read/update |
| 201 | Resource created |
| 202 | Asynchronous work accepted |
| 204 | Successful deletion with no response body |
| 400 | Malformed request |
| 401 | Missing or invalid authentication |
| 403 | Authenticated but not permitted |
| 404 | Resource not found or intentionally concealed |
| 409 | Conflict or state/version conflict |
| 413 | Payload too large |
| 415 | Unsupported media type |
| 422 | Semantic validation error, if used consistently |
| 429 | Rate limit exceeded |
| 500 | Unexpected server error |
| 503 | Dependency unavailable |

## 10. API security requirements

- Authenticate protected endpoints.
- Enforce resource ownership and later tenant membership.
- Validate every input.
- Paginate collection endpoints.
- Apply upload size and file-type limits.
- Use timeouts for provider and integration calls.
- Avoid leaking whether another tenant owns a resource.
- Do not trust user IDs or tenant IDs from the browser.
- Add request IDs and structured logs.
- Document actual endpoints with OpenAPI.
- Keep this document synchronized with implemented behavior.
