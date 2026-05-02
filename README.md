# RAG Starter: Postgres + pgvector + FastAPI

A clean, honest starter for building a Retrieval-Augmented Generation (RAG) system on
PostgreSQL with the `pgvector` extension. It is meant as a reference implementation you
can read in an afternoon and extend, not a turnkey production system.

## What's in here

- **Postgres + pgvector** via Docker Compose
- **Ingestion script** that chunks documents, generates embeddings, and bulk-inserts
- **FastAPI server** with `/ingest`, `/search`, and `/ask` endpoints
- **Switchable embeddings**: OpenAI (`text-embedding-3-small`) or local
  `sentence-transformers` (`all-MiniLM-L6-v2`). Pick per-deployment via env var.
- **HNSW index** on the embedding column with sane defaults
- **Hybrid search** (vector + Postgres full-text) with a weighted combiner
- **Tests** for chunking, the embedding adapter, and the search endpoints (using a
  test container)

## Honest scaling notes

This starter will comfortably handle the low millions of chunks on a single Postgres
instance with HNSW. Past that, you start running into real problems that this code
does *not* solve for you:

- **HNSW index build time** grows roughly linearly with row count and is memory-hungry.
  At 100M+ rows, you'll want to build the index with `maintenance_work_mem` cranked up,
  build it concurrently, and possibly partition.
- **Recall vs. latency tradeoffs** with HNSW depend on `m` and `ef_construction` at
  build time and `hnsw.ef_search` at query time. The defaults here (`m=16`,
  `ef_construction=64`) are fine for development; tune them on your real data.
- **Storage**: pgvector stores `vector(N)` as 4 bytes per dimension. 1536-dim OpenAI
  embeddings are ~6 KB per row before indexing. Plan capacity accordingly.
- **Embedding model drift**: if you change embedding models, every existing vector in
  the table is now in a different space than your queries. The schema includes an
  `embedding_model` column so you can manage migrations explicitly.
- **The `/ask` endpoint calls an LLM**. Costs scale linearly with query volume and
  context size. Cache aggressively in production.

If you need to search hundreds of millions of documents with subsecond latency, you are
not going to get there by copy-pasting this repo. You'll need real capacity planning,
sharding strategy, and load testing on representative data.

## Quick start

```bash
# 1. Copy env template and fill in (at minimum, set EMBEDDING_PROVIDER)
cp .env.example .env

# 2. Bring up Postgres
docker compose up -d db

# 3. Install Python deps (Python 3.11+)
pip install -r requirements.txt

# 4. Initialize the schema
python -m scripts.init_db

# 5. Ingest some sample documents
python -m scripts.ingest_sample

# 6. Run the API
uvicorn app.main:app --reload --port 8000

# 7. Try it
curl -X POST http://localhost:8000/search \
  -H 'Content-Type: application/json' \
  -d '{"query": "how does pgvector work", "top_k": 5}'
```

## Configuration

All configuration is via environment variables. See `.env.example` for the full list.
Key ones:

| Variable | Default | Notes |
|---|---|---|
| `EMBEDDING_PROVIDER` | `local` | `openai` or `local` |
| `OPENAI_API_KEY` | — | Required if provider is `openai` or you use `/ask` |
| `OPENAI_EMBEDDING_MODEL` | `text-embedding-3-small` | 1536 dims by default |
| `LOCAL_EMBEDDING_MODEL` | `all-MiniLM-L6-v2` | 384 dims |
| `LLM_MODEL` | `gpt-4o-mini` | Used by `/ask` |
| `DATABASE_URL` | `postgresql://rag:rag@localhost:5432/rag` | |
| `EMBEDDING_DIM` | `384` | **Must match your provider's output dim** |

The `EMBEDDING_DIM` setting is load-bearing: the schema defines `embedding vector(N)`
where `N` comes from this value. If you switch providers, you need to recreate the
table or use a separate column.

## Project layout

```
app/
  main.py            FastAPI app and routes
  config.py          Settings loaded from env
  db.py              psycopg connection pool
  embeddings.py      Provider-agnostic embedding interface
  chunking.py        Token-aware text chunking
  rag.py             Retrieval + (optional) rerank + generation
  schemas.py         Pydantic request/response models
scripts/
  init_db.py         Apply sql/schema.sql
  ingest_sample.py   Index a few sample docs to play with
  ingest_files.py    Index a directory of .txt/.md files
sql/
  schema.sql         Tables, indexes, extensions
tests/
  test_chunking.py
  test_embeddings.py
  test_api.py
docker/
  Dockerfile         For the API service
docker-compose.yml   Postgres + API
```

## What this is *not*

- Not a managed service. You run it.
- Not multi-tenant. Add row-level security if you need that.
- Not a vector DB benchmark. Numbers depend entirely on your data, hardware, and
  index parameters; measure on your own workload.
- Not optimized for billions of rows. See "Honest scaling notes" above.

## License

MIT. Use it however you want.
