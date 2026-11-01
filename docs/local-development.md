# IntelliDocs AI — Local Development Setup

## 1. Purpose

This guide describes the intended local development setup. Commands should be verified against the current repository before use. Do not assume a step is complete merely because it is documented here.

## 2. Developer environment

The developer uses an Apple Silicon Mac (M2) and Fish shell.

Observed terminal versions:

```text
Node.js:        v26.13.0
npm:            11.20.0
Python:         3.15.2 (detected; project target is Python 3.12)
Git:            2.54.0
Docker:         29.9.2
Docker Compose: v5.6.1
```

Use Python 3.12 for this project. Do not uninstall or overwrite the system Python.

Node.js 24 LTS was recommended as a predictable LTS runtime, but the current environment has Node.js 26.13.0. Verify Next.js compatibility before changing Node versions.

## 3. Prerequisites

- Node.js and npm.
- Python 3.12.
- Git.
- Docker Desktop and Docker Compose.
- Homebrew on macOS.

Verify:

```fish
node --version
npm --version
python3.12 --version
git --version
docker --version
docker compose version
docker info
```

`docker info` should show server information. If it reports that it cannot connect, start Docker Desktop and wait for the engine to become ready.

## 4. Repository layout

Initial repository:

```text
intellidocs-ai/
├── frontend/
├── backend/
├── infrastructure/
├── docs/
├── docker-compose.yml
├── .env.example
├── .gitignore
├── README.md
└── AGENT.md
```

The Next.js app was initialized in `frontend/` with App Router and the `src/` directory. Its default starter page was successfully displayed in the browser.

Do not recreate the frontend. Inspect existing files first.

## 5. Frontend

From the repository root:

```fish
cd frontend
npm install
npm run dev
```

Open:

http://localhost:3000

Stop the development server with `Ctrl+C`.

Frontend requirements:

- Next.js App Router.
- JavaScript only.
- `.js` and `.jsx` files.
- No `.ts` or `.tsx`.
- Centralized API client.
- Backend API base URL in `frontend/.env.local`.

Example `frontend/.env.local`:

```env
NEXT_PUBLIC_API_BASE_URL=http://localhost:5000/api/v1
```

Only put browser-safe values in `NEXT_PUBLIC_` variables.

## 6. Python environment

From the repository root:

```fish
cd backend
python3.12 -m venv venv
source venv/bin/activate.fish
```

If activation syntax differs in the installed Fish version, run the virtual environment's Python directly:

```fish
./venv/bin/python --version
```

Initial dependencies may include:

```fish
pip install Flask Flask-Cors Flask-SQLAlchemy Flask-Migrate python-dotenv "psycopg[binary]" pgvector
pip freeze > requirements.txt
```

Install Celery and Redis client dependencies when background processing is implemented, rather than adding unused dependencies prematurely.

## 7. PostgreSQL with pgvector using Docker Compose

A root-level `docker-compose.yml` can initially contain:

```yaml
services:
  postgres:
    image: pgvector/pgvector:pg17
    container_name: intellidocs-postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: intellidocs
      POSTGRES_PASSWORD: change-this-local-password
      POSTGRES_DB: intellidocs
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U intellidocs -d intellidocs"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:
```

This is a local-development example. Replace the sample password and never use it in production.

Start:

```fish
docker compose up -d
docker compose ps
```

Enable pgvector:

```fish
docker exec -it intellidocs-postgres \
  psql -U intellidocs -d intellidocs \
  -c "CREATE EXTENSION IF NOT EXISTS vector;"
```

Verify:

```fish
docker exec -it intellidocs-postgres \
  psql -U intellidocs -d intellidocs \
  -c "SELECT extname FROM pg_extension;"
```

Stop containers without deleting database data:

```fish
docker compose down
```

To remove volumes, which deletes local database data, use `docker compose down -v` only when intentionally resetting the local database.

## 8. Backend environment

Create `backend/.env` locally and exclude it from Git.

Example:

```env
FLASK_ENV=development
SECRET_KEY=replace-with-a-long-random-development-secret
DATABASE_URL=postgresql+psycopg://intellidocs:change-this-local-password@localhost:5432/intellidocs
FRONTEND_ORIGIN=http://localhost:3000
STORAGE_BACKEND=local
LOCAL_STORAGE_PATH=./storage
MAX_UPLOAD_SIZE_MB=25
OPENAI_API_KEY=
ANTHROPIC_API_KEY=
```

Do not commit real credentials. The exact variables should match the implemented configuration.

## 9. Running the applications

Typical local workflow uses separate terminal tabs:

### Terminal 1 — Database

From repository root:

```fish
docker compose up -d
```

### Terminal 2 — Flask

```fish
cd backend
source venv/bin/activate.fish
python run.py
```

The intended development address is:

http://localhost:5000

### Terminal 3 — Next.js

```fish
cd frontend
npm run dev
```

The intended development address is:

http://localhost:3000

## 10. Health checks

Planned health endpoint:

```text
GET http://localhost:5000/api/v1/health
```

Example response:

```json
{
  "status": "healthy",
  "application": "IntelliDocs AI",
  "version": "1.0.0"
}
```

Once the database is integrated, add a readiness check that verifies database connectivity without exposing credentials or internal infrastructure details.

## 11. Git hygiene

The root `.gitignore` should exclude at least:

```gitignore
# Environment files
.env
.env.*
!.env.example
!.env.local.example

# Node
node_modules/
.next/
out/

# Python
venv/
.venv/
__pycache__/
*.py[cod]
.pytest_cache/
.mypy_cache/

# Local data and uploads
storage/
uploads/
*.log

# OS/editor
.DS_Store
.vscode/
.idea/
```

Review ignore patterns so required examples and documentation remain tracked.

## 12. Troubleshooting

### Docker command not found

Check Docker Desktop installation and shell PATH. On macOS, Docker Desktop commonly installs the CLI under:

```text
/Applications/Docker.app/Contents/Resources/bin
```

For Fish, if that path exists:

```fish
fish_add_path /Applications/Docker.app/Contents/Resources/bin
```

Then open Docker Desktop and verify `docker info`.

### Docker daemon unavailable

Run:

```fish
open -a Docker
docker info
```

Wait for Docker Desktop to finish starting.

### Python 3.12 not found

Install with Homebrew:

```fish
brew install python@3.12
python3.12 --version
```

Use the explicit `python3.12` interpreter to create the virtual environment.

### Database connection failure

Check:

```fish
docker compose ps
docker logs intellidocs-postgres
```

Confirm database name, username, password, port, and `DATABASE_URL` match.

### Frontend cannot reach Flask

- Confirm Flask is running.
- Confirm the API base URL.
- Confirm CORS allows `http://localhost:3000`.
- Check browser network and console errors.
- Do not put backend secrets in frontend environment variables.

## 13. Local development completion checklist

- [ ] Node and npm work.
- [ ] Python 3.12 is installed.
- [ ] Backend virtual environment is created.
- [ ] Docker Desktop engine is running.
- [ ] PostgreSQL container is healthy.
- [ ] pgvector extension is enabled.
- [ ] Flask health endpoint responds.
- [ ] Flask can connect to PostgreSQL.
- [ ] Next.js runs at port 3000.
- [ ] Frontend can call Flask.
- [ ] `.env` files and local data are ignored by Git.

## 14. Production note

This guide is for local development. Do not deploy Flask's development server, sample passwords, local filesystem storage, permissive CORS, or development secrets to production. Production deployment requires HTTPS, secret management, managed/persistent storage, database backups, secure authentication, monitoring, and a production WSGI server.
