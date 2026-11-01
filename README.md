# IntelliDocs AI

**Enterprise Document Intelligence Platform**

![Status](https://img.shields.io/badge/status-Phase%201%20MVP-blue)
![Frontend](https://img.shields.io/badge/frontend-Next.js%20%7C%20React-black)
![Backend](https://img.shields.io/badge/backend-Python%20%7C%20Flask-blue)
![Database](https://img.shields.io/badge/database-PostgreSQL%20%2B%20pgvector-336791)
![AI](https://img.shields.io/badge/AI-RAG%20%7C%20OpenAI%20%7C%20Claude-7c3aed)
![Deployment](https://img.shields.io/badge/deployment-Docker-2496ED)

An AI-powered document intelligence platform that transforms
unstructured business documents into searchable, actionable, and
verifiable knowledge. Users can upload documents, process them
asynchronously, search their contents semantically, and ask questions
through a Retrieval-Augmented Generation (RAG) interface with citations
to the original sources.

> **Development status:** IntelliDocs AI is being developed
> incrementally. This README describes the target Phase 1 MVP
> architecture and development workflow. An endpoint, command, or
> capability should be considered implemented only when it exists and
> passes its corresponding tests in the repository.

------------------------------------------------------------------------

## Table of Contents

1.  [Project Overview](#1-project-overview)
2.  [Core Features](#2-core-features)
3.  [Architecture Overview](#3-architecture-overview)
    -   [High-Level System
        Architecture](#a-high-level-system-architecture)
    -   [Document Processing Workflow](#b-document-processing-workflow)
    -   [RAG Question-Answering
        Workflow](#c-rag-question-answering-workflow)
    -   [Data Model](#d-data-model)
4.  [Technology Stack](#4-technology-stack)
5.  [Phase 1 Scope](#5-phase-1-scope)
6.  [API Design](#6-api-design)
7.  [Security and Data Isolation](#7-security-and-data-isolation)
8.  [Environment Variables](#8-environment-variables)
9.  [Local Development Setup](#9-local-development-setup)
10. [Docker Development Workflow](#10-docker-development-workflow)
11. [Testing and Verification](#11-testing-and-verification)
12. [Project Directory Structure](#12-project-directory-structure)
13. [Development Roadmap](#13-development-roadmap)
14. [Engineering Principles](#14-engineering-principles)

------------------------------------------------------------------------

## 1. Project Overview

Organizations store important information across contracts, invoices,
policies, employee handbooks, technical manuals, reports, and other
documents. Finding a reliable answer often requires employees to open
multiple files, search manually, and verify the information themselves.

IntelliDocs AI aims to simplify this process by providing a single
platform to:

-   Upload and manage business documents.
-   Extract and normalize document text.
-   Classify and summarize documents with AI.
-   Split documents into searchable chunks.
-   Generate vector embeddings.
-   Search document content by meaning rather than exact keywords.
-   Ask natural-language questions across authorized documents.
-   Generate grounded answers with verifiable source citations.

### Example use case

A user uploads an employee handbook and asks:

> "What is the annual leave policy?"

The platform retrieves relevant passages from the user's accessible
documents, supplies those passages to the selected language model, and
returns an answer with document names, page references where available,
and chunk-level source references.

The answer should be traceable to retrieved source material. If the
available documents do not contain sufficient evidence, the system
should say so rather than invent an answer.

### Target users

-   Legal and compliance teams.
-   Human resources departments.
-   Finance and accounting teams.
-   Manufacturing and quality teams.
-   IT and engineering teams.
-   Organizations with large document collections.

------------------------------------------------------------------------

## 2. Core Features

  -----------------------------------------------------------------------
  Feature                 Description             Phase
  ----------------------- ----------------------- -----------------------
  Document upload         Upload supported files  1
                          and validate file type  
                          and size                

  Document storage        Store original files    1
                          separately from         
                          relational metadata     

  Text extraction         Extract text and        1
                          page-level information  
                          where supported         

  Async processing        Process documents       1
                          through Celery workers  

  Classification          Assign useful document  1
                          categories              

  Summarization           Generate document       1
                          summaries               

  Chunking                Split extracted text    1
                          into retrieval-friendly 
                          passages                

  Embeddings              Generate vector         1
                          representations of      
                          chunks                  

  Semantic search         Retrieve relevant       1
                          passages using pgvector 

  RAG chat                Answer questions using  1
                          retrieved document      
                          context                 

  Source citations        Return document and     1
                          chunk references with   
                          answers                 

  Document management     List, inspect, and      1
                          delete owned documents  

  Organizations and       Support multiple        2
  workspaces              organizations and       
                          workspaces              

  RBAC                    Granular role- and      2
                          permission-based access 

  Document versioning     Track document          2
                          revisions               

  QMS workflows           Support                 2
                          quality-management      
                          processes               

  ERP/CRM integrations    Connect enterprise      2
                          business systems        

  Distributed processing  Scale processing across 3
                          distributed workers     

  Kubernetes deployment   Orchestrate production  3
                          workloads               

  Advanced observability  Centralized metrics,    3
                          tracing, and alerting   
  -----------------------------------------------------------------------

------------------------------------------------------------------------

## 3. Architecture Overview

The Phase 1 system uses a modular application architecture. The Next.js
frontend communicates with a Flask REST API. PostgreSQL stores
application data and vectors, Redis supports caching and Celery
messaging, and Celery workers perform expensive document-processing
tasks outside the request-response cycle.

### A. High-Level System Architecture

``` mermaid
flowchart TB
    U[User] --> FE[Next.js Frontend]
    FE -->|HTTPS / REST API| API[Flask API]

    API --> AUTH[Authentication and Authorization]
    API --> DOC[Document Services]
    API --> SEARCH[Search and Chat Services]

    DOC --> DB[(PostgreSQL + pgvector)]
    SEARCH --> DB
    API --> REDIS[(Redis)]

    API -->|Enqueue processing task| REDIS
    REDIS --> WORKER[Celery Worker]

    WORKER --> EXTRACT[Text Extraction]
    EXTRACT --> CLASSIFY[Classification and Summarization]
    EXTRACT --> CHUNK[Chunking]
    CHUNK --> EMBED[Embedding Generation]
    EMBED --> DB

    WORKER --> STORAGE[(Document File Storage)]
    DOC --> STORAGE

    SEARCH --> RETRIEVE[Authorized Retrieval]
    RETRIEVE --> LLM[OpenAI API / Claude API]
    LLM --> SEARCH
    SEARCH --> FE
```

**Architectural responsibilities**

-   **Next.js:** User interface, navigation, document library, upload
    experience, search, and chat.
-   **Flask API:** Request validation, authentication, authorization,
    orchestration, and REST endpoints.
-   **Service layer:** Document, search, chat, and AI business logic.
-   **PostgreSQL:** Users, document metadata, processing state,
    extracted content, chunks, and embeddings.
-   **pgvector:** Similarity search over document embeddings.
-   **Redis:** Celery broker and application cache.
-   **Celery:** Asynchronous document-processing jobs.
-   **File storage:** Original uploaded files. Phase 1 can use a
    persistent local Docker volume; object storage can be introduced
    later.
-   **AI providers:** Backend-only integrations with OpenAI and
    Anthropic.

### B. Document Processing Workflow

``` mermaid
sequenceDiagram
    actor User
    participant UI as Next.js
    participant API as Flask API
    participant DB as PostgreSQL
    participant Redis
    participant Worker as Celery Worker
    participant Store as File Storage
    participant AI as AI Provider

    User->>UI: Select and upload document
    UI->>API: POST /api/v1/documents
    API->>API: Authenticate, authorize, validate file
    API->>Store: Save original file
    API->>DB: Create document (QUEUED)
    API->>Redis: Enqueue processing task
    API-->>UI: Return document ID and status

    Redis->>Worker: Deliver processing task
    Worker->>DB: Set status to PROCESSING
    Worker->>Store: Read original file
    Worker->>Worker: Extract and normalize text
    Worker->>AI: Classify and summarize (when configured)
    Worker->>Worker: Split text into chunks
    Worker->>AI: Generate embeddings
    Worker->>DB: Save extracted text, chunks and vectors
    Worker->>DB: Set status to COMPLETED
    UI->>API: GET /api/v1/documents/{id}
    API-->>UI: Return current processing status
```

**Processing states**

``` text
UPLOADED
   |
   v
QUEUED
   |
   v
PROCESSING
   |
   +------> COMPLETED
   |
   +------> FAILED
```

Processing must be safe to retry. A failed task should record a useful
error state without leaving a document permanently marked as processing.
Duplicate task delivery must not create duplicate chunk or embedding
records.

### C. RAG Question-Answering Workflow

``` mermaid
sequenceDiagram
    actor User
    participant UI as Next.js
    participant API as Flask API
    participant DB as PostgreSQL + pgvector
    participant LLM as OpenAI / Claude

    User->>UI: Ask a question
    UI->>API: POST /api/v1/chat
    API->>API: Authenticate user
    API->>API: Resolve authorized document scope
    API->>API: Generate query embedding
    API->>DB: Similarity search within authorized scope
    DB-->>API: Relevant chunks and source metadata
    API->>API: Build context with citation identifiers
    API->>LLM: Question + retrieved context
    LLM-->>API: Grounded answer and cited identifiers
    API->>API: Validate citations against retrieved sources
    API-->>UI: Answer, citations, and conversation data
    UI-->>User: Display answer and source references
```

**RAG principles**

1.  Convert the user's query into an embedding using the configured
    embedding model.
2.  Retrieve relevant chunks from pgvector.
3.  Apply document authorization constraints as part of retrieval, not
    after it.
4.  Pass only authorized retrieved content to the language model.
5.  Require the model to identify the supplied source references used in
    its answer.
6.  Validate returned citation identifiers against the retrieved chunk
    set.
7.  Return source metadata such as document name, page number, and chunk
    reference.
8.  Make insufficient evidence explicit.

A citation must refer to a real retrieved source. The application must
not accept arbitrary document names, page numbers, or chunk IDs
generated by the model.

### D. Data Model

The following is the proposed Phase 1 logical data model. Exact columns,
constraints, and relationships will be finalized in database migrations
as modules are implemented.

``` mermaid
erDiagram
    USERS ||--o{ DOCUMENTS : owns
    DOCUMENTS ||--o{ DOCUMENT_CHUNKS : contains
    USERS ||--o{ CONVERSATIONS : creates
    CONVERSATIONS ||--o{ CHAT_MESSAGES : contains
    CHAT_MESSAGES ||--o{ MESSAGE_CITATIONS : references
    DOCUMENT_CHUNKS ||--o{ MESSAGE_CITATIONS : cited_by

    USERS {
        uuid id PK
        string email
        string password_hash
        datetime created_at
    }

    DOCUMENTS {
        uuid id PK
        uuid owner_id FK
        string filename
        string storage_key
        string mime_type
        string status
        string category
        text summary
        datetime created_at
    }

    DOCUMENT_CHUNKS {
        uuid id PK
        uuid document_id FK
        integer chunk_index
        integer page_number
        text content
        vector embedding
    }

    CONVERSATIONS {
        uuid id PK
        uuid user_id FK
        string title
        datetime created_at
    }

    CHAT_MESSAGES {
        uuid id PK
        uuid conversation_id FK
        string role
        text content
        datetime created_at
    }

    MESSAGE_CITATIONS {
        uuid id PK
        uuid message_id FK
        uuid chunk_id FK
    }
```

**Data-model notes**

-   `storage_key` identifies the stored original file; it should not
    expose a filesystem path to clients.
-   `status` records the asynchronous processing lifecycle.
-   `page_number` is nullable when the extraction format does not
    provide reliable page information.
-   Embedding dimensions must match the configured embedding model.
-   Conversations and messages are associated with an authenticated
    user.
-   Ownership and authorization checks must be applied to document,
    chunk, conversation, and citation access.
-   The diagram is conceptual and does not replace the actual migration
    definitions.

------------------------------------------------------------------------

## 4. Technology Stack

  -----------------------------------------------------------------------
  Layer                   Technology              Responsibility
  ----------------------- ----------------------- -----------------------
  Frontend                Next.js                 Application framework
                                                  and routing

  UI                      React                   Interactive user
                                                  interface

  Frontend language       JavaScript              `.js` and `.jsx` files
                                                  only

  Backend                 Python + Flask          REST API and
                                                  application services

  Database                PostgreSQL              Relational data
                                                  persistence

  Vector search           pgvector                Embedding storage and
                                                  similarity search

  Cache                   Redis                   Caching

  Message broker          Redis                   Celery task messaging

  Background jobs         Celery                  Asynchronous document
                                                  processing

  AI provider             OpenAI API              Embeddings and
                                                  language-model
                                                  capabilities

  AI provider             Anthropic Claude API    Alternative
                                                  language-model provider

  Containerization        Docker                  Consistent development
                                                  and deployment

  Orchestration           Kubernetes              Reserved for Phase 3
  -----------------------------------------------------------------------

### Backend implementation conventions

The backend will use a modular Flask application structure with:

-   SQLAlchemy for database access.
-   Flask-Migrate/Alembic for database migrations.
-   A PostgreSQL driver compatible with the selected SQLAlchemy version.
-   A pgvector integration compatible with PostgreSQL and the selected
    embedding dimensions.
-   A dedicated Celery application and task modules.
-   Provider adapters for OpenAI and Anthropic.

These are implementation conventions for the project; dependencies and
versions will be pinned in the backend dependency file.

### Frontend implementation conventions

-   JavaScript only; do not introduce TypeScript.
-   Use `.js` and `.jsx` source files.
-   Keep API communication in a dedicated frontend service layer.
-   Keep reusable UI components separate from feature-specific
    components.
-   Never expose private AI provider keys in browser code.
-   Use environment variables prefixed with `NEXT_PUBLIC_` only for
    values intentionally exposed to the browser.

------------------------------------------------------------------------

## 5. Phase 1 Scope

Phase 1 focuses on a complete, usable MVP rather than a distributed
enterprise architecture.

### Included in Phase 1

-   User authentication and document ownership.
-   Document upload, validation, storage, and deletion.
-   Text extraction for supported document formats.
-   Asynchronous document processing with Celery.
-   Classification and summarization.
-   Text chunking and embedding generation.
-   pgvector semantic search.
-   RAG-based chat.
-   Verifiable source citations.
-   Basic document library and processing status UI.
-   Docker-based local environment.
-   Automated tests for critical workflows.

### Initial document formats

  Format   Phase 1 handling
  -------- -----------------------------------------------------
  PDF      Text extraction and page references where available
  DOCX     Paragraph and table text extraction where supported
  TXT      Plain-text extraction

Scanned PDFs require OCR and should be treated as a separately scoped
capability unless OCR is explicitly added to the Phase 1 implementation.

### Deliberately excluded from Phase 1

-   Kafka.
-   Kubernetes.
-   Large microservices architecture.
-   Distributed vector databases.
-   Enterprise organization hierarchy.
-   Advanced multi-tenant administration.
-   Complex approval and QMS workflows.
-   ERP/CRM integrations.

------------------------------------------------------------------------

## 6. API Design

The Flask backend will expose versioned REST endpoints under `/api/v1`.

The following is the proposed API contract. Request and response schemas
will be formalized alongside each endpoint implementation.

### Authentication

  Method   Endpoint                  Purpose
  -------- ------------------------- --------------------------------------
  POST     `/api/v1/auth/register`   Register a user
  POST     `/api/v1/auth/login`      Authenticate a user
  GET      `/api/v1/auth/me`         Get the authenticated user's profile

### Documents

  -------------------------------------------------------------------------------------------
  Method                  Endpoint                                    Purpose
  ----------------------- ------------------------------------------- -----------------------
  POST                    `/api/v1/documents`                         Upload a document

  GET                     `/api/v1/documents`                         List the current user's
                                                                      documents

  GET                     `/api/v1/documents/{document_id}`           Get document metadata
                                                                      and status

  DELETE                  `/api/v1/documents/{document_id}`           Delete an owned
                                                                      document

  GET                     `/api/v1/documents/{document_id}/summary`   Retrieve document
                                                                      summary
  -------------------------------------------------------------------------------------------

### Search and chat

  -------------------------------------------------------------------------------------------
  Method                  Endpoint                                    Purpose
  ----------------------- ------------------------------------------- -----------------------
  POST                    `/api/v1/search`                            Search authorized
                                                                      document content

  POST                    `/api/v1/chat`                              Ask a question using
                                                                      RAG

  GET                     `/api/v1/conversations`                     List the user's
                                                                      conversations

  GET                     `/api/v1/conversations/{conversation_id}`   Retrieve an owned
                                                                      conversation
  -------------------------------------------------------------------------------------------

### System

  Method   Endpoint           Purpose
  -------- ------------------ ----------------------------
  GET      `/api/v1/health`   Application health check
  GET      `/api/v1/ready`    Dependency readiness check

### Example: Upload a document

``` http
POST /api/v1/documents
Authorization: Bearer <access_token>
Content-Type: multipart/form-data
```

Form field:

``` text
file: company-policy.pdf
```

Illustrative response:

``` json
{
  "id": "document-uuid",
  "filename": "company-policy.pdf",
  "status": "QUEUED",
  "message": "Document accepted for processing."
}
```

### Example: Semantic search

``` http
POST /api/v1/search
Authorization: Bearer <access_token>
Content-Type: application/json
```

Request:

``` json
{
  "query": "What is the remote work policy?",
  "limit": 5
}
```

Illustrative response:

``` json
{
  "query": "What is the remote work policy?",
  "results": [
    {
      "document_id": "document-uuid",
      "document_name": "employee-handbook.pdf",
      "chunk_id": "chunk-uuid",
      "page_number": 12,
      "content": "Employees may work remotely subject to manager approval.",
      "similarity": 0.89
    }
  ]
}
```

The example similarity value and content are illustrative, not measured
system output.

### Example: RAG chat

``` http
POST /api/v1/chat
Authorization: Bearer <access_token>
Content-Type: application/json
```

Request:

``` json
{
  "question": "What is the remote work policy?",
  "conversation_id": null
}
```

Illustrative response:

``` json
{
  "conversation_id": "conversation-uuid",
  "answer": "Remote work is permitted subject to manager approval.",
  "citations": [
    {
      "document_id": "document-uuid",
      "document_name": "employee-handbook.pdf",
      "chunk_id": "chunk-uuid",
      "page_number": 12
    }
  ]
}
```

The backend must validate that each returned citation belongs to the
authorized retrieved context.

------------------------------------------------------------------------

## 7. Security and Data Isolation

Document content can contain confidential business information.
Authorization must be enforced throughout the complete request and
processing lifecycle.

### Phase 1 security controls

-   Hash passwords using a suitable password-hashing algorithm.
-   Authenticate protected API requests.
-   Validate uploaded file extensions, MIME types, and actual file
    content where practical.
-   Enforce configurable upload size limits.
-   Generate storage keys rather than trusting user-supplied filenames
    as paths.
-   Store secrets in environment variables.
-   Keep original documents outside publicly served frontend
    directories.
-   Restrict document access to its owner.
-   Apply ownership filters to semantic retrieval queries.
-   Authorize access before document text is sent to an AI provider.
-   Validate citation IDs against retrieved authorized chunks.
-   Avoid logging document contents, credentials, tokens, or provider
    secrets.
-   Use parameterized database operations through the data-access layer.
-   Apply safe error handling without returning stack traces to clients.

### Authorization rule

A user must never retrieve another user's document through:

-   Document metadata endpoints.
-   Direct document download or access.
-   Semantic search.
-   RAG chat.
-   Conversation history.
-   Citation references.

**Important:** Filtering unauthorized results after vector retrieval is
not sufficient. Authorization constraints must be part of the database
retrieval operation so unauthorized content is never supplied to the
model.

### AI data handling

AI provider calls happen only in the Python backend. Provider selection
and credentials are controlled by backend configuration. The application
should send only the minimum authorized context needed to answer a
request.

------------------------------------------------------------------------

## 8. Environment Variables

Never commit real credentials, API keys, database passwords, or
production secrets to source control. Keep `.env.example` files as
templates.

The exact variables may evolve as implementation modules are added.

### Backend --- `backend/.env`

  ---------------------------------------------------------------------------------------------------------------------------
  Variable                               Required Description      Example
  ------------------------- --------------------- ---------------- ----------------------------------------------------------
  `FLASK_ENV`                                 Yes Application      `development`
                                                  environment      

  `SECRET_KEY`                                Yes Flask            `<strong-random-secret>`
                                                  application      
                                                  secret           

  `DATABASE_URL`                              Yes PostgreSQL       `postgresql+psycopg://user:password@db:5432/intellidocs`
                                                  connection URL   

  `REDIS_URL`                                 Yes Redis connection `redis://redis:6379/0`
                                                  URL              

  `CELERY_BROKER_URL`                         Yes Celery broker    `redis://redis:6379/1`
                                                  URL              

  `CELERY_RESULT_BACKEND`                     Yes Celery result    `redis://redis:6379/2`
                                                  backend          

  `JWT_SECRET_KEY`                            Yes Authentication   `<strong-random-secret>`
                                                  token signing    
                                                  secret           

  `FRONTEND_URL`                              Yes Allowed frontend `http://localhost:3000`
                                                  origin           

  `UPLOAD_DIR`                                Yes Persistent       `/app/storage/uploads`
                                                  upload directory 

  `MAX_CONTENT_LENGTH`                        Yes Maximum upload   `20971520`
                                                  size in bytes    

  `OPENAI_API_KEY`                     For OpenAI OpenAI API       `<openai-api-key>`
                                                  credential       

  `ANTHROPIC_API_KEY`                  For Claude Anthropic API    `<anthropic-api-key>`
                                                  credential       

  `AI_DEFAULT_PROVIDER`                       Yes Default LLM      `openai`
                                                  provider         

  `EMBEDDING_MODEL`                           Yes Configured       `<embedding-model>`
                                                  embedding model  

  `CHAT_MODEL`                                Yes Configured chat  `<chat-model>`
                                                  model            
  ---------------------------------------------------------------------------------------------------------------------------

Model names should be selected and pinned through configuration.
Embedding model changes may require re-embedding existing document
chunks if vector dimensions or embedding spaces differ.

### Frontend --- `frontend/.env.local`

  -----------------------------------------------------------------------------------------------
  Variable                             Required Description      Example
  ----------------------- --------------------- ---------------- --------------------------------
  `NEXT_PUBLIC_API_URL`                     Yes Public base URL  `http://localhost:5000/api/v1`
                                                of the Flask API 

  -----------------------------------------------------------------------------------------------

Only non-secret configuration belongs in `NEXT_PUBLIC_` variables.

### Example backend environment template

``` env
FLASK_ENV=development
SECRET_KEY=replace-with-a-random-secret
DATABASE_URL=postgresql+psycopg://intellidocs:intellidocs_dev_password@db:5432/intellidocs
REDIS_URL=redis://redis:6379/0
CELERY_BROKER_URL=redis://redis:6379/1
CELERY_RESULT_BACKEND=redis://redis:6379/2
JWT_SECRET_KEY=replace-with-a-different-random-secret
FRONTEND_URL=http://localhost:3000
UPLOAD_DIR=/app/storage/uploads
MAX_CONTENT_LENGTH=20971520

OPENAI_API_KEY=
ANTHROPIC_API_KEY=
AI_DEFAULT_PROVIDER=openai
EMBEDDING_MODEL=
CHAT_MODEL=
```

Use different, securely generated secrets outside local development. Do
not use the illustrative development database password in production.

------------------------------------------------------------------------

## 9. Local Development Setup

### Prerequisites

Install the following tools:

-   Git.
-   Node.js LTS and npm.
-   Python version supported by the backend dependency lock or
    requirements file.
-   Docker Desktop with Docker Compose.
-   Access to a PostgreSQL instance with pgvector support, if running
    services outside Docker.
-   An API key for the AI provider used during AI integration testing.

### Step 1: Clone the repository

``` bash
git clone <YOUR_INTELLIDOCS_REPOSITORY_URL>
cd intellidocs-ai
```

### Step 2: Configure environment files

``` bash
cp backend/.env.example backend/.env
cp frontend/.env.example frontend/.env.local
```

Update the environment variables with local configuration and provider
credentials as needed.

### Step 3: Start infrastructure and application services

Once the Docker Compose configuration is implemented:

``` bash
docker compose up --build
```

The intended local service endpoints are:

  Service            URL
  ------------------ -------------------------
  Next.js frontend   `http://localhost:3000`
  Flask API          `http://localhost:5000`
  PostgreSQL         `localhost:5432`
  Redis              `localhost:6379`

### Step 4: Apply database migrations

Run the migration command from the backend container after the migration
tooling is configured:

``` bash
docker compose exec backend flask db upgrade
```

### Step 5: Verify application health

``` bash
curl http://localhost:5000/api/v1/health
```

Use the readiness endpoint to verify required dependencies:

``` bash
curl http://localhost:5000/api/v1/ready
```

These commands describe the intended Phase 1 workflow and become
executable when the corresponding application entry points, Compose
services, and Flask CLI commands are present.

------------------------------------------------------------------------

## 10. Docker Development Workflow

Docker Compose will manage the initial Phase 1 services.

### Intended services

  Service      Responsibility
  ------------ -----------------------------------
  `frontend`   Next.js web application
  `backend`    Flask REST API
  `worker`     Celery document-processing worker
  `db`         PostgreSQL with pgvector
  `redis`      Cache and Celery broker

### Common Docker commands

Start services:

``` bash
docker compose up -d --build
```

List service status:

``` bash
docker compose ps
```

View logs:

``` bash
docker compose logs -f
```

View logs for a specific service:

``` bash
docker compose logs -f backend
docker compose logs -f worker
```

Execute a command inside the backend container:

``` bash
docker compose exec backend flask db upgrade
```

Stop services:

``` bash
docker compose down
```

Stop services and remove named volumes (this deletes persisted local
database and file data):

``` bash
docker compose down -v
```

Do not use `down -v` unless you intentionally want to delete local
persistent data.

------------------------------------------------------------------------

## 11. Testing and Verification

Tests should be added alongside each feature. The backend test suite
will use pytest; frontend tests will be selected to fit the implemented
Next.js components and workflows.

### Backend test coverage

  Area               Verification
  ------------------ -----------------------------------------------------
  Authentication     Registration, login, invalid credentials
  Authorization      User cannot access another user's documents
  Upload             Valid file, unsupported file, oversized file
  Extraction         PDF, DOCX, and TXT extraction
  Processing         Queued, processing, completed, and failed states
  Celery             Task execution, retry, and idempotency
  Chunking           Chunk boundaries and metadata
  Embeddings         Correct dimensions and persistence
  Semantic search    Relevant result retrieval
  RAG                Answer grounded in retrieved content
  Citations          Every citation maps to a retrieved authorized chunk
  Failure handling   Provider errors and worker failures

### Frontend test coverage

-   Upload form validation.
-   Document list and status rendering.
-   Processing and error states.
-   Search results.
-   Chat message rendering.
-   Citation navigation.
-   Authentication and protected routes.

### End-to-end MVP acceptance criteria

The Phase 1 MVP is considered functionally complete when:

1.  A user can register and authenticate.
2.  The user can upload a supported document.
3.  The API stores the original file and returns a document ID.
4.  A Celery worker processes the document asynchronously.
5.  Extracted text, chunks, and embeddings are persisted.
6.  The document reaches a visible completed or failed state.
7.  The user can perform semantic search over their authorized
    documents.
8.  The user can ask a natural-language question about those documents.
9.  The response includes citations that map to real retrieved source
    chunks.
10. A user cannot retrieve another user's documents or content through
    search or chat.

------------------------------------------------------------------------

## 12. Project Directory Structure

The following is the planned modular repository structure. Directories
and modules should be created as their corresponding functionality is
implemented.

``` text
intellidocs-ai/
│
├── frontend/
│   ├── public/                         # Static assets
│   ├── src/
│   │   ├── app/                        # Next.js routes and layouts
│   │   ├── components/                 # Shared UI components
│   │   ├── features/
│   │   │   ├── auth/                   # Authentication UI
│   │   │   ├── documents/              # Upload and document library
│   │   │   ├── search/                 # Semantic search UI
│   │   │   └── chat/                   # RAG chat UI
│   │   ├── hooks/                      # Reusable React hooks
│   │   ├── lib/                        # Shared frontend utilities
│   │   └── services/                   # API client and endpoint functions
│   ├── .env.example
│   ├── Dockerfile
│   └── package.json
│
├── backend/
│   ├── app/
│   │   ├── api/
│   │   │   └── v1/                     # Versioned API routes
│   │   ├── ai/                         # Provider adapters and prompts
│   │   ├── core/                       # Configuration, security, logging
│   │   ├── models/                     # SQLAlchemy models
│   │   ├── repositories/               # Database access operations
│   │   ├── schemas/                    # Request and response validation
│   │   ├── services/
│   │   │   ├── documents/              # Document business logic
│   │   │   ├── extraction/             # File text extraction
│   │   │   ├── embeddings/             # Embedding generation
│   │   │   ├── search/                 # Semantic retrieval
│   │   │   └── chat/                   # RAG orchestration
│   │   ├── tasks/                      # Celery tasks
│   │   ├── utils/                      # Shared backend utilities
│   │   ├── __init__.py                 # Flask application factory
│   │   └── celery_app.py               # Celery application setup
│   ├── migrations/                     # Alembic / Flask-Migrate files
│   ├── tests/
│   │   ├── unit/
│   │   ├── integration/
│   │   └── conftest.py
│   ├── .env.example
│   ├── Dockerfile
│   └── requirements.txt
│
├── infrastructure/
│   └── docker/                         # Supporting Docker configuration
│
├── docs/
│   ├── architecture/                   # Architecture and design decisions
│   ├── api/                            # API documentation
│   └── ai/                             # RAG and embedding documentation
│
├── storage/                            # Local development storage; gitignored
│
├── .env.example
├── .gitignore
├── docker-compose.yml
└── README.md
```

### Backend layering rule

The backend follows this request flow:

``` text
API Route
   |
   v
Schema Validation
   |
   v
Authentication / Authorization
   |
   v
Service Layer
   |
   v
Repository / External Provider
   |
   v
Database or AI Provider
```

Route handlers should not contain document-processing logic,
vector-search implementation, or direct LLM orchestration.

------------------------------------------------------------------------

## 13. Development Roadmap

IntelliDocs AI has exactly three development phases.

### Phase 1 --- Basic Working MVP

**Objective:** Deliver a complete document upload, asynchronous
processing, semantic search, and cited RAG workflow.

-   Repository and Docker foundation.
-   Flask application and database migrations.
-   Authentication and document ownership.
-   Document upload and storage.
-   Text extraction and preprocessing.
-   Celery task processing.
-   Classification and summarization.
-   Chunking and embedding generation.
-   pgvector semantic search.
-   RAG chat and source citations.
-   Document library and frontend integration.
-   Unit, integration, and end-to-end testing.

### Phase 2 --- Enterprise-Ready Multi-Tenant Application

**Objective:** Support organizations, teams, permissions, and enterprise
document workflows.

-   Organization and workspace management.
-   Tenant-aware data isolation.
-   RBAC and granular permissions.
-   Document versioning and revision history.
-   Audit logging.
-   QMS workflows.
-   ERP and CRM integrations.
-   Enterprise administration and security controls.

### Phase 3 --- Distributed, Horizontally Scalable Platform

**Objective:** Scale processing and serving workloads across distributed
infrastructure.

-   Kubernetes orchestration.
-   Horizontally scalable API and workers.
-   Distributed processing architecture.
-   Advanced message infrastructure where justified.
-   Scalable object storage and retrieval.
-   Production observability, metrics, logs, and tracing.
-   High availability, recovery, and fault tolerance.
-   Capacity and performance optimization.

**Phase boundary:** Kafka, Kubernetes, and a large microservices
architecture are not part of the Phase 1 implementation.

------------------------------------------------------------------------

## 14. Engineering Principles

1.  **Incremental development:** Implement and verify one module at a
    time.
2.  **Complete implementations:** Deliver executable modules with
    configuration, tests, and run instructions.
3.  **Fixed stack:** Next.js, React, JavaScript, Python, Flask,
    PostgreSQL, Redis, Celery, and pgvector.
4.  **No TypeScript:** Frontend source files use `.js` and `.jsx`.
5.  **Modular monolith first:** Keep Phase 1 straightforward and
    maintainable.
6.  **Separation of concerns:** Keep business logic outside Flask route
    handlers.
7.  **Asynchronous processing:** Use Celery for expensive
    document-processing tasks.
8.  **Secure retrieval:** Enforce authorization before retrieving
    document content for AI processing.
9.  **Provider isolation:** Keep OpenAI and Claude integrations in the
    Python backend behind clear interfaces.
10. **Configuration through environment variables:** Do not hardcode
    secrets or environment-specific values.
11. **Database migrations:** Track schema changes through migrations.
12. **Test with features:** Add unit and integration tests alongside
    implementation.
13. **Traceable AI output:** Ground answers in retrieved content and
    validate source citations.
14. **Avoid premature complexity:** Introduce distributed infrastructure
    only when the relevant phase requires it.

------------------------------------------------------------------------

## Project Goal

IntelliDocs AI aims to transform organizational documents into
searchable, actionable, and verifiable knowledge.

The first milestone is a working Phase 1 MVP where an authenticated user
can upload documents, have them processed asynchronously, search their
contents semantically, and ask questions that receive grounded answers
with verifiable citations to the original sources.
