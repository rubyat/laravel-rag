# rubyat/laravel-rag

[![Latest Version](https://img.shields.io/packagist/v/rubyat/laravel-rag.svg)](https://packagist.org/packages/rubyat/laravel-rag)
[![Total Downloads](https://img.shields.io/packagist/dt/rubyat/laravel-rag.svg)](https://packagist.org/packages/rubyat/laravel-rag)
[![License](https://img.shields.io/packagist/l/rubyat/laravel-rag.svg)](LICENSE)

A production-ready Retrieval-Augmented Generation (RAG) toolkit for Laravel with PostgreSQL + pgvector. Ingest documents, store embeddings, and answer questions grounded **only** in your own content — with source citations.

## Features

- 🧩 **Document ingestion** — text is chunked (configurable size/overlap), embedded, and stored.
- 🔎 **pgvector vector search** — cosine similarity over an **HNSW** index for high recall at any scale.
- 💬 **RAG pipeline** — retrieves the most relevant chunks and asks an LLM to answer from them.
- 📎 **Source citations** — every answer returns the chunks it used, with similarity scores.
- 🔌 **Driver-based** — swap embedding/chat providers; OpenAI built-in, plus deterministic `fake` drivers for offline dev and tests (no API key, no network).
- ⚙️ **Sync or queued ingestion** — inline by default, or dispatch a job for large documents.
- 🌐 **HTTP API** — ready-to-use `POST /api/rag/ingest` and `POST /api/rag/ask` endpoints.
- ✅ **Tested** — ships with a Pest suite that runs against pgvector using the fake drivers.

## Requirements

- PHP **8.3+**
- Laravel **11, 12, or 13**
- PostgreSQL with the [`pgvector`](https://github.com/pgvector/pgvector) extension (**0.5+** for HNSW)

## Installation

```bash
composer require rubyat/laravel-rag
```

Publish the config and run the migration:

```bash
php artisan vendor:publish --tag=rag-config
php artisan migrate
```

> The migration runs `CREATE EXTENSION IF NOT EXISTS vector`, which needs a database role allowed to create extensions. The [pgvector Docker image](https://hub.docker.com/r/pgvector/pgvector) (`pgvector/pgvector:pg16`) is the quickest way to get it locally.

Configure a provider (or keep the offline `fake` drivers — no network needed):

```env
OPENAI_API_KEY=sk-...
RAG_EMBEDDING_DRIVER=openai
RAG_CHAT_DRIVER=openai
```

## Quick Start

### In code

```php
use Rubyat\LaravelRag\Ingestion\DocumentIngestor;
use Rubyat\LaravelRag\Rag\RagPipeline;

// Ingest a document (chunk -> embed -> store in pgvector)
app(DocumentIngestor::class)->ingest('handbook.md', $longText);

// Ask a question grounded in the ingested chunks
$result = app(RagPipeline::class)->ask('How does pgvector search work?');

$result['answer'];     // string — grounded in your documents
$result['citations'];  // array — [{ source, chunk_index, content, score }, ...]
```

### HTTP API

Ingest:

```bash
curl -X POST http://localhost:8000/api/rag/ingest \
  -H "Content-Type: application/json" \
  -d '{"source": "handbook.md", "content": "Postgres pgvector stores embeddings and supports cosine similarity search..."}'
# => 201 { "message": "Document ingested.", "source": "handbook.md", "chunks": 1 }
```

Ask:

```bash
curl -X POST http://localhost:8000/api/rag/ask \
  -H "Content-Type: application/json" \
  -d '{"question": "How does pgvector search work?", "top_k": 4}'
# => 200
# {
#   "answer": "pgvector stores embeddings and supports cosine similarity search...",
#   "citations": [
#     { "source": "handbook.md", "chunk_index": 0, "content": "...", "score": 0.83 }
#   ]
# }
```

## Configuration

Published to `config/rag.php`. Every option is environment-driven:

| Option | Env | Default | Description |
| --- | --- | --- | --- |
| `embedding_driver` | `RAG_EMBEDDING_DRIVER` | `openai` | `openai` or `fake` |
| `chat_driver` | `RAG_CHAT_DRIVER` | `openai` | `openai` or `fake` |
| `dimensions` | `RAG_DIMENSIONS` | `1536` | Vector size; must match the embedding model |
| `chunk_size` | `RAG_CHUNK_SIZE` | `1000` | Chunk length in characters |
| `chunk_overlap` | `RAG_CHUNK_OVERLAP` | `200` | Overlap between chunks |
| `top_k` | `RAG_TOP_K` | `4` | Chunks retrieved per question |
| `queue_ingestion` | `RAG_QUEUE_INGESTION` | `false` | Dispatch ingestion to the queue (needs a worker) |
| `register_routes` | `RAG_REGISTER_ROUTES` | `true` | Auto-register the HTTP routes |
| `route_prefix` | `RAG_ROUTE_PREFIX` | `api/rag` | Prefix for the package routes |

The `fake` drivers are deterministic and require no network access — they power the test suite and let you try the full flow without API keys.

## Swappable drivers

Embedding and chat providers are resolved from config and implement small contracts, so you can plug in your own:

```php
use Rubyat\LaravelRag\Contracts\EmbeddingDriver;
use Rubyat\LaravelRag\Contracts\ChatDriver;
```

Built in: `OpenAiEmbeddingDriver` / `OpenAiChatDriver` (HTTP) and `FakeEmbeddingDriver` / `FakeChatDriver` (offline, deterministic).

## How it works

```
            ingest                                   ask
              │                                        │
              ▼                                        ▼
      DocumentChunker                          VectorRetriever
              │ chunks                                 │ query embedding
              ▼                                         ▼
      EmbeddingDriver ─── embeddings ──►  pgvector documents (HNSW, cosine)
                                                        │ top-k chunks
                                                        ▼
                                                  RagPipeline
                                                  (context + LLM)
                                                        ▼
                                              answer + citations
```

Documents are stored one row per chunk in the `documents` table (`source`, `chunk_index`, `content`, `metadata`, `embedding vector(n)`), indexed with HNSW (`vector_cosine_ops`). Retrieval orders by cosine distance (`embedding <=> :query`) and returns a similarity score in `[0, 1]`.

## Testing

The suite runs against PostgreSQL + pgvector using the deterministic `fake` drivers, so no API keys or network are required:

```bash
./vendor/bin/pest
```

## License

The MIT License (MIT). See [LICENSE](LICENSE).
