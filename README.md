# RAG Desk

A small, authenticated document Q&A app. The Go server stores uploaded plain-text files and passages in PostgreSQL/pgvector, retrieves relevant passages for each question, then asks a configured model to answer from that context. The model does not train on or automatically learn the stored files.

## Run with a hosted model

1. Copy `.env.example` to `.env`.
2. Set unique values for `SESSION_SECRET`, `ADMIN_PASSWORD`, and `DB_PASSWORD`; add a hosted provider key to `LLM_API_KEY`.
3. Start the stack:

   ```sh
   docker compose up --build
   ```

4. Open `http://localhost:8080/login` and sign in with `ADMIN_USERNAME` and `ADMIN_PASSWORD`.
5. Upload `.txt` files in Library, then ask questions from Ask.

The database is reachable only over Compose's private network and uses a persistent volume. The app waits for Postgres's health check before starting.

## Run with local models (Ollama)

Local mode keeps document text and questions on the Compose network instead of sending them to a hosted model provider:

1. In `.env`, set:

   ```dotenv
   LLM_PROVIDER=ollama
   LLM_BASE_URL=http://ollama:11434
   EMBEDDING_MODEL=nomic-embed-text
   CHAT_MODEL=llama3.2
   EMBEDDING_DIMENSIONS=768
   ```

2. Start the local-model profile:

   ```sh
   docker compose --profile local-model up --build -d
   docker compose exec ollama ollama pull nomic-embed-text
   docker compose exec ollama ollama pull llama3.2
   ```

The model files persist in the `ollamadata` volume. Choose models and dimensions that are available for your hardware. The Ollama defaults above are examples and may need more memory than a small development machine has.

### Changing model providers after uploading

Every stored passage and question embedding must have the same dimension and come from a compatible embedding model. Changing provider, embedding model, or `EMBEDDING_DIMENSIONS` does not automatically regenerate existing vectors. Back up the database and re-upload/reindex the documents before asking questions with the new embedding configuration. The schema accepts varying pgvector dimensions so both hosted and local configurations can be used; retrieval uses exact similarity scans rather than a dimension-specific ANN index.

## Reliability and security notes

- Uploads are limited by `MAX_UPLOAD_BYTES` (maximum configurable value: 100 MiB), checked on the server, validated as UTF-8 text, and stored transactionally.
- Replacement embeddings are generated before the database transaction; an indexing or save failure leaves the existing document and passages unchanged.
- Forms require authentication and a same-origin `Origin`/`Referer`; session cookies are HTTP-only, SameSite=Lax, time-limited, and Secure in production.
- Production mode rejects default/placeholder passwords and short session secrets. Terminate HTTPS at a trusted reverse proxy and set `APP_ENV=production`; configure database TLS with `DB_SSLMODE`.
- The app sends only retrieved excerpts and the question to the selected model. The model receives no SQL access.
- The similarity cutoff intentionally returns an “insufficient information” response when the best match is weak. Tune it with a real evaluation set before changing it.
- Compose publishes the web app, not the database. Put remote deployments behind TLS and the intended network access controls.

**Rotate any model API key that was previously placed in a local `.env` file or shared.** Keep `.env` out of source control; it is ignored by `.gitignore`.

## Configuration

| Variable | Purpose |
|---|---|
| `APP_ENV`, `APP_PORT` | Runtime environment and HTTP listener |
| `SESSION_SECRET`, `ADMIN_USERNAME`, `ADMIN_PASSWORD` | Admin authentication |
| `DB_HOST`, `DB_PORT`, `DB_USER`, `DB_PASSWORD`, `DB_NAME`, `DB_SSLMODE` | PostgreSQL connection |
| `LLM_PROVIDER` | `openai` (OpenAI-compatible API) or `ollama` |
| `LLM_BASE_URL`, `LLM_API_KEY` | Model API endpoint and optional hosted API key |
| `EMBEDDING_MODEL`, `CHAT_MODEL` | Selected embedding and answer models |
| `EMBEDDING_DIMENSIONS` | Expected vector length (hosted default: 1536; Ollama example: 768) |
| `MAX_UPLOAD_BYTES` | Per-file limit, from 1 byte to 100 MiB |

## Tests

Run the app tests with:

```sh
go test ./...
```

Tests cover upload validation and persistence, atomic replacement failure, retrieval/source attribution, insufficient-context behavior, session expiry, same-origin checks, chunk boundaries, and Ollama/provider error handling.
