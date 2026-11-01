# IntelliDocs AI — Backend Architecture

## 1. Purpose

Define the Flask/Python backend structure and conventions. The backend is a modular monolith in Phase 1, with clean boundaries that permit selected components to be extracted later.

## 2. Technology

- Python 3.12 is the selected project runtime.
- Flask with an application factory.
- Flask Blueprints for route grouping.
- Flask-SQLAlchemy / SQLAlchemy for persistence.
- Flask-Migrate / Alembic for schema migrations.
- PostgreSQL and pgvector.
- Celery with Redis when asynchronous processing is introduced.
- pytest for tests.
- Gunicorn for production WSGI serving.

Do not assume the developer's system Python is the project interpreter. Use a Python 3.12 virtual environment.

## 3. Suggested backend structure

```text
backend/
├── app/
│   ├── __init__.py
│   ├── config.py
│   ├── extensions.py
│   ├── models/
│   ├── routes/
│   ├── services/
│   ├── tasks/
│   ├── schemas/
│   ├── utils/
│   └── common/
├── migrations/
├── tests/
│   ├── unit/
│   ├── integration/
│   └── conftest.py
├── requirements.txt
└── run.py
```

Create modules as they are needed; do not create empty placeholder files for every future capability.

## 4. Application factory

`create_app()` should:

1. Create the Flask application.
2. Load configuration.
3. Initialize extensions.
4. Register Blueprints.
5. Register error handlers.
6. Configure logging and request IDs.
7. Return the application.

Avoid importing the application globally from unrelated modules. Use the application context where required.

## 5. Layer responsibilities

### Routes / Blueprints

- Parse HTTP inputs.
- Authenticate and authorize requests.
- Validate request data.
- Call service functions.
- Return consistent HTTP responses.
- Do not contain complex business logic or provider SDK calls.

### Services

- Implement business rules.
- Coordinate repositories/models and external adapters.
- Enforce domain-level invariants.
- Be testable without running the HTTP server.

### Models and persistence

- Define database entities and relationships.
- Use migrations for schema changes.
- Avoid embedding large file bytes in PostgreSQL.
- Persist storage keys, metadata, and processing state.

### Tasks

- Execute slow work outside request handlers.
- Be idempotent.
- Record progress and failures.
- Use bounded retries for transient errors.
- Avoid duplicate chunk/index creation.

### AI adapters

- Hide OpenAI/Anthropic SDK details behind a stable internal interface.
- Apply timeouts, retry policies, usage tracking, and structured-output validation.
- Keep credentials in backend configuration.
- Never expose provider keys to the browser.

## 6. API conventions

Use `/api/v1` as the API prefix.

- JSON for standard requests and responses.
- Multipart or signed-upload flow for files, depending on phase.
- Pagination for collection endpoints.
- Consistent error codes and response shapes.
- Correct HTTP status codes.
- OpenAPI documentation.
- Request IDs in logs and responses.
- No stack traces or secrets in client-facing errors.

## 7. Authentication and authorization

For Phase 1, implement secure authentication and resource ownership checks. Select one session/token strategy and document it before implementation.

For browser sessions using cookies:

- Use HttpOnly, Secure in production, and appropriate SameSite settings.
- Implement CSRF protection where required.
- Avoid storing long-lived sensitive tokens in localStorage.

For every protected operation, verify the authenticated principal's permission to access the resource. Never trust a client-supplied owner ID.

Phase 2 adds organization membership and RBAC. Tenant context must be resolved and validated server-side.

## 8. Error handling

Use centralized error handlers and a consistent format, for example:

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

Do not expose internal exceptions, SQL statements, file paths, credentials, or provider responses containing sensitive data.

## 9. Testing

Use pytest for:

- Service unit tests.
- Route tests.
- Database integration tests.
- Authentication and authorization.
- File validation.
- Extraction and chunking.
- Retrieval authorization.
- Citation validation.
- Task idempotency and retry behavior.

Mock paid AI APIs in normal test runs. Keep real-provider integration tests separate and opt-in.

## 10. Production serving

Do not use Flask's development server in production. Use Gunicorn or an appropriate production WSGI server behind a reverse proxy/load balancer.

Keep API instances stateless where possible. Store files in an external storage abstraction rather than on an ephemeral application container filesystem.

## 11. Evolution path

Keep internal module boundaries clean. Extract a worker or service only when it needs independent scaling, resource isolation, deployment, failure isolation, or ownership. Do not introduce microservices solely for architectural appearance.
