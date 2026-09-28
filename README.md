# Northstar Knowledge Assistant

A document-grounded enterprise question-answering application. Users connect
knowledge sources, upload PDF and DOCX files, and ask questions that are answered
only from the connected documents, stored user memory, and a small set of
application reference facts.

## Features

- Multiple connected knowledge sources with connect/disconnect state
- PDF and DOCX upload with versioning
- Resumable, checkpointed ingestion
- Private object storage for uploaded files
- Vector similarity search over document chunks
- Grounded answers that refuse to answer from outside knowledge
- Verifiable source metadata attached to every answer
- Light and dark user interface with conversation history

## Architecture

```text
Browser (React + Vite)
        |
        v
FastAPI backend (Python)
        |
        +--> PostgreSQL with pgvector   (documents metadata, chunks, embeddings)
        +--> Private object storage     (uploaded files)
        +--> LLM provider               (answer generation)
```

The backend runs on a container host. The frontend is a static site and can be
hosted on any static hosting platform.

## Project layout

```text
frontend/        Static single-page application
api.py           HTTP routes, ingestion, retrieval, and answer generation
database.py      Data models and schema initialization
main.py          Local development entry point
Dockerfile       Container image definition
render.yaml      Hosting blueprint
vercel.json      Static hosting configuration
```

## Configuration

The application is configured entirely through environment variables. Values are
supplied through your hosting provider's dashboard or a local `.env` file that
is excluded from version control.

| Variable | Purpose |
| --- | --- |
| `DATABASE_URL` | Database connection used by the backend |
| `GROQ_API_KEY` | Credential for the language model provider |
| `GROQ_MODEL` | Model identifier |
| `FRONTEND_ORIGINS` | Allowed frontend origins for browser requests |
| `SUPABASE_URL` | Object storage project endpoint |
| `SUPABASE_SERVICE_ROLE_KEY` | Object storage credential for the backend |
| `SUPABASE_STORAGE_BUCKET` | Bucket that holds uploaded files |
| `MAX_UPLOAD_MB` | Maximum accepted upload size |
| `VITE_API_BASE_URL` | Backend base URL used by the frontend at build time |

A sample file with placeholder values is provided in `.env.example`.

## Local development

Create a virtual environment and install dependencies:

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

Copy the sample configuration and fill in the placeholders:

```bash
cp .env.example .env
```

Start the backend:

```bash
set -a
source .env
set +a
python3 main.py
```

Start the frontend in a second terminal:

```bash
cd frontend
npm ci
npm run dev
```

Open the local frontend URL printed by the development server.

## Database

The backend initializes its own schema on startup. It requires a PostgreSQL
database with the vector extension available, and it creates the tables,
indexes, and compatibility columns it needs.

## Object storage

Uploaded files are written to a private bucket. The backend stores only object
references in the database and downloads a temporary copy when it needs to read
a file for extraction. This keeps uploaded documents available across backend
restarts and deployments.

## Background ingestion

Uploads are accepted immediately and processed in the background. Ingestion is
checkpointed per unit, so an interrupted job can be retried and will resume from
its last saved position instead of starting over.

Concurrent operations on the same document are rejected to keep state
consistent: a document that is already processing cannot be retried or deleted
until the active job finishes.

## Grounding behavior

Answer generation is constrained by a dedicated system prompt that:

- permits only reference facts, stored user memory, and retrieved evidence
- refuses to answer when the supplied context does not contain the answer
- treats retrieved content as data rather than instructions
- requires exact preservation of names, dates, and numeric values
- reports disagreements between sources instead of inventing a resolution
- prohibits fabricated citations, versions, or quotations

Source metadata is attached to each response by the application, so the
interface can display exactly which document, version, and page supported an
answer.

## Deployment

### Backend

The included `Dockerfile` produces a self-contained image with the OCR engine
installed, so no host-level OCR installation is required. Deploy the image to a
container host and configure the variables from the table above. The service
exposes a health endpoint that can be used for platform health checks.

### Frontend

The included static hosting configuration builds the `frontend/` directory and
publishes the generated assets. Set the backend base URL as the single frontend
environment variable, then deploy. The frontend requires no database or storage
access and communicates only with the backend.

## Operational considerations

- Containerized ingestion workers are memory intensive. Hosts with at least
  1 GB of memory are recommended for reliable document processing.
- Free-tier application hosts commonly scale to zero after inactivity, which
  introduces cold-start latency and can interrupt long-running ingestion.
- Free database and storage tiers commonly pause idle projects and apply usage
  quotas.
- Language model providers commonly apply request and token rate limits.
- Background tasks are process-local. A restart during ingestion leaves the job
  in an interrupted state that can be retried from its checkpoint.

For production use, choose an always-on host with sufficient memory, and review
the quotas and retention settings of the database, storage, and model provider
you select.
