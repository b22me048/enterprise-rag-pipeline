# Enterprise RAG Pipeline

A production-grade **Retrieval-Augmented Generation** pipeline for querying messy enterprise PDFs -- 10-K filings, technical manuals, contracts, SOPs -- with citation-grounded answers.

Combines hybrid search (dense vectors + BM25 sparse), cross-encoder reranking, and LLM synthesis. Supports fully local execution for privacy, or OpenAI for higher-quality embeddings.

---

## Features

- **PDF ingestion** -- parses complex PDFs (tables, headers, scanned docs, 2000+ page documents) via unstructured.io with automatic strategy fallback
- **Hybrid search** -- dense vector search (semantic) fused with BM25 (exact keyword match) via weighted reciprocal-rank fusion
- **Cross-encoder reranker** -- two-stage retrieval: fast bi-encoder first, precise cross-encoder second
- **Citation-grounded answers** -- every factual claim links back to source document, page number, and section heading
- **Conversational memory** -- session-based chat history stored in PostgreSQL, with context carry-over across follow-up questions
- **Local or cloud embeddings** -- sentence-transformers (free, private, offline) or OpenAI text-embedding-3-small
- **CLI + API** -- interactive Rich-based CLI wizard, direct ingestion commands, FastAPI server for programmatic access
- **Parallel ingestion** -- multi-worker PDF processing with batch upserts and retry logic

---

## Architecture

```
                           +------------------------------+
                           |     FastAPI (port 8000)       |
                           |  /chat  /ingest  /health     |
                           +------+-----------+-----------+
                                  |           |
                   +--------------+           +--------------+
                   v                                         v
     +---------------------------+      +------------------------------+
     |     INGESTION PIPELINE     |      |      RETRIEVAL PIPELINE      |
     |  parser.py                |      |  retrieval.py                |
     |  chunker.py               |      |    (Dense + BM25)            |
     |  llm.py (embed_texts)     |      |  reranker.py                 |
     |  ingest.py                |      |    (Cross-encoder)           |
     +-------------+-------------+      |  query.py (LLM synthesis)    |
                   |                    +--------------+---------------+
                   v                                   |
         +------------------+                          |
         |     Qdrant       |<--- vector search -------+
         |   (Vector DB)    |
         +------------------+                          |
                                                       v
                                               +---------------+
                                               |  OpenAI LLM   |
                                               |  (GPT-4o-mini)|
                                               +---------------+

     +------------------+
     |   PostgreSQL     |<-- ingestion logs, chat sessions, chat messages
     |   (Relational)   |
     +------------------+
```

### Data Flow

**Ingestion:**
```
PDF file -> parser.py (unstructured.io) -> elements -> chunker.py (semantic overlap chunks)-> llm.py (embed_texts) -> Qdrant upsert (batched, with retry) -> PostgreSQL ingestion log
```

**Query:**
```
Question -> retrieval.py (hybrid_search: dense Qdrant + sparse BM25)
        -> reranker.py (CrossEncoder rerank top-K)
        -> prompts.py (build_rag_messages with citation context)
        -> llm.py (OpenAI chat completion, temperature=0.0)
        -> Answer + Citations (with conversational history carry-over)
```

---

## Quickstart

### Prerequisites

| Dependency | Version | Purpose |
|------------|---------|---------|
| Python | 3.11+ | Runtime |
| Docker + Docker Compose | Latest | Qdrant + PostgreSQL containers |
| OpenAI API Key | - | LLM answer generation |

### 1. Configure environment

```bash
cp .env.example .env
# Edit .env -- set OPENAI_API_KEY
```

### 2. Start infrastructure

```bash
docker compose up -d
```

Verify Qdrant is running at http://localhost:6333/dashboard

### 3. Install dependencies

```bash
python -m venv .venv
source .venv/bin/activate        # macOS / Linux
# .venv\Scripts\activate         # Windows

pip install -r requirements.txt
pip install -e .                 # register ragpipeline CLI commands
```

### 4. Poppler setup (Windows only)

Required for `hi_res` PDF parsing strategy (scanned docs, complex layouts). The default `fast` strategy does not require Poppler.

```powershell
# Windows -- download Poppler binaries into vendor/
Invoke-WebRequest -Uri "https://github.com/oschwartz10612/poppler-windows/releases/download/v24.08.0-0/Release-24.08.0-0.zip" -OutFile poppler.zip
Expand-Archive poppler.zip -DestinationPath vendor/
Move-Item vendor/Release-24.08.0-0 vendor/poppler
Remove-Item poppler.zip
```

```bash
# macOS: brew install poppler
# Linux:  sudo apt-get install poppler-utils
```

### 5. Ingest documents

**Interactive wizard** (recommended for first use):

```bash
ragpipeline
```

Walks you through target folder, parsing strategy, worker count, and embedding provider. Runs first-time setup checks (Docker, .env, database initialization).

**Direct ingestion:**

```bash
# Basic
python -m src.ingest data/raw_pdfs/

# With options
python -m src.ingest data/raw_pdfs/ --strategy hi_res --workers 4 --recreate
python -m src.ingest doc.pdf --strategy fast --embedding-provider local
python -m src.ingest data/raw_pdfs/ --dry-run             # parse+chunk only, no DB writes
python -m src.ingest data/raw_pdfs/ --force                # re-ingest already-logged PDFs
```

### 6. Ask questions

**CLI:**

```bash
ragpipeline --query "What was the net income for 2025?"
ragpipeline -q "What are the risk factors mentioned in the 10-K?"
```

**API server:**

```bash
ragpipeline-api
# or: uvicorn src.api:app --reload
```

```bash
curl -X POST http://localhost:8000/chat \
  -H "Content-Type: application/json" \
  -d '{"question": "What was the net income for 2025?"}'
```

---

## API Server

Start with `ragpipeline-api` (or `uvicorn src.api:app --reload`). Interactive docs at http://localhost:8000/docs.

### Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/` | API info and link to /docs |
| `POST` | `/chat` | Ask a question, get citation-grounded answer |
| `POST` | `/ingest` | Trigger ingestion of a PDF file or directory |
| `GET` | `/health` | Check Qdrant, PostgreSQL, and LLM status |

### /chat

Request:

```json
{
  "question": "What was the net income for 2025?",
  "session_id": "optional-session-id"
}
```

Response:

```json
{
  "answer": "Net income for fiscal year 2025 was $72.880 billion...",
  "citations": [
    {
      "doc_name": "2026-Annual-Report-Web.pdf",
      "page_number": 130,
      "section": "Preamble",
      "text": "..."
    }
  ],
  "model": "gpt-4o-mini",
  "latency_seconds": 6.57,
  "timestamp": "2026-06-24T03:51:25.297265+00:00",
  "session_id": "94ca80fe07e5493faf903f81e2ebae05"
}
```

Conversational follow-ups with a `session_id` automatically carry context from previous turns:

```bash
# First question (establishes session)
curl -X POST http://localhost:8000/chat \
  -H "Content-Type: application/json" \
  -d '{"question": "What was net income for 2025?", "session_id": "my-session"}'

# Follow-up -- retrieves based on expanded context + previous chunks
curl -X POST http://localhost:8000/chat \
  -H "Content-Type: application/json" \
  -d '{"question": "How does that compare to 2024?", "session_id": "my-session"}'
```

### /ingest

```bash
curl -X POST http://localhost:8000/ingest \
  -H "Content-Type: application/json" \
  -d '{"target": "data/raw_pdfs/", "strategy": "fast", "workers": 4}'
```

### /health

```bash
curl http://localhost:8000/health
```

---

## Configuration

All configuration is via environment variables (`.env` file).

| Variable | Default | Description |
|----------|---------|-------------|
| `OPENAI_API_KEY` | - | OpenAI API key (required) |
| `LLM_MODEL` | `gpt-4o-mini` | OpenAI model for answer generation |
| `EMBEDDING_PROVIDER` | `local` | `openai` or `local` |
| `EMBEDDING_MODEL_OPENAI` | `text-embedding-3-small` | OpenAI embedding model |
| `EMBEDDING_MODEL_LOCAL` | `sentence-transformers/all-mpnet-base-v2` | Local sentence-transformers model |
| `EMBEDDING_DIM_OPENAI` | `1536` | OpenAI embedding dimensions |
| `EMBEDDING_DIM_LOCAL` | `768` | Local embedding dimensions |
| `RERANKER_MODEL` | `cross-encoder/ms-marco-MiniLM-L-6-v2` | Cross-encoder reranker model |
| `PARSER_STRATEGY` | `fast` | `fast`, `hi_res`, `auto`, or `ocr_only` |
| `CHUNK_SIZE` | `512` | Target tokens per chunk |
| `CHUNK_OVERLAP` | `50` | Token overlap between chunks |
| `MIN_CHUNK_SIZE` | `128` | Minimum tokens to emit a chunk |
| `QDRANT_URL` | `http://localhost:6333` | Qdrant HTTP endpoint |
| `QDRANT_COLLECTION_NAME` | `enterprise_docs` | Qdrant collection name |
| `DATABASE_URL` | - | PostgreSQL connection string (auto-built from `POSTGRES_*` vars) |
| `RETRIEVAL_TOP_K` | `20` | Initial retrieval count |
| `RERANK_TOP_K` | `5` | Final reranked count |
| `DENSE_SPARSE_WEIGHT` | `0.7` | Hybrid search fusion weight (0=sparse only, 1=dense only) |
| `API_HOST` | `0.0.0.0` | API server bind address |
| `API_PORT` | `8000` | API server port |

### Embedding Providers

| Provider | Model | Dimensions | Cost | Privacy |
|----------|-------|------------|------|---------|
| **OpenAI** | `text-embedding-3-small` | 1536 | ~$0.02/1M tokens | Data sent to OpenAI |
| **Local** | `all-mpnet-base-v2` | 768 | Free | Fully offline |

The CLI wizard saves your preference to `~/.ragpipeline/config.env` and reuses it for subsequent queries.

---

## Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| **Vendor binaries** | Poppler shipped in `vendor/` for zero system dependencies on Windows |
| **PostgreSQL + Qdrant** | Vector DBs are poor at OLTP; Postgres handles logs, metadata, chat memory |
| **Hybrid search** | Dense handles semantics, BM25 catches exact matches (tickers, part numbers, clauses) |
| **Cross-encoder reranker** | Bi-encoders are fast but fuzzy; cross-encoder narrows top-K for precision |
| **Citation-grounded prompts** | Every answer includes source doc + page + section, preventing hallucination |
| **Conversational carry-over** | Previous-turn chunks merge into follow-up retrieval for context continuity |
| **Batching + retry** | tenacity backoff for embedding APIs, 3-retry loop for Qdrant upserts |
| **Lazy settings proxy** | Hot-swappable config without restart (`get_settings.cache_clear()`) |

---

## Performance

### Optimizations Applied

| Bottleneck | Fix | Impact |
|------------|-----|--------|
| Token counting called 5000+ times per PDF | Pre-computed `token_count` on ParsedElement + LRU cache on `count_tokens()` | ~93% reduction in calls, 2-5x faster chunking |
| Default parser strategy was `hi_res` (10-100x slower) | Changed default to `fast` | Significantly faster on digital PDFs |
| Oversized Qdrant upsert batches | Batch splitting by byte size (target: ~25MB per batch) | Prevents 400 errors on large documents |

### Known Limits

- Qdrant has a 33MB per-request payload limit -- batch splitting targets 25MB to stay safely under
- BM25 index is built per-session from Qdrant data (no persistence yet)
- Large documents (2000+ pages) can produce 500k+ tokens and take significant time to embed locally

---

## Tests

```bash
pytest tests/ -v
```

13 tests covering chunking (empty, single, multi-element, long text, tables), context formatting, RAG messages with/without history, citation deduplication, tokenization, and PDF path discovery.

---
