Batching Embeddings: Implementation Plan

Goals
- Reduce end-to-end runtime when using the hosted `llm_api` backend by batching multiple texts per request.
- Keep the Ollama (`ollama`) path working unchanged while still benefiting from faster DB inserts.
- Preserve correctness: vector dimensions, row ordering, and idempotent writes.

Constraints & Notes
- OpenAPI-style embedding endpoints accept multiple inputs via `input: ["...", "..."]` and return `data: [{embedding: [...]}, ...]` in the same order.
- Ollama’s embeddings endpoint does not support batching; we’ll call it per-text but still batch DB inserts.
- pgvector schema remains unchanged; only the ingestion logic changes.

High-Level Changes
- Add config knobs in `.env` (optional):
  - `EMBED_BATCH_SIZE` (e.g., 64 or 128)
  - `EMBED_MAX_RETRIES` (e.g., 5)
  - `EMBED_BACKOFF_MS` (e.g., 500)
  - `EMBED_MAX_CONCURRENCY` (optional; default 1)
- Introduce a `batch_embed(texts)` helper:
  - `llm_api`: single POST with `input: texts` returns list of vectors.
  - `ollama`: loop over texts calling `get_embedding` once per text.
  - Validate every vector’s length matches the discovered dimension.
- Replace the Step 9 per-row loop with a batched pipeline:
  - Chunk `df_context` into batches of `EMBED_BATCH_SIZE`.
  - Call `batch_embed` once per chunk.
  - Insert results using `psycopg2.extras.execute_values` for speed.
  - Commit per batch, print progress.

Pseudocode
```
BATCH = int(os.getenv("EMBED_BATCH_SIZE", 64))
MAX_RETRIES = int(os.getenv("EMBED_MAX_RETRIES", 5))
BACKOFF_MS = int(os.getenv("EMBED_BACKOFF_MS", 500))

def batch_embed(texts, model=None):
    if EMBEDDING_BACKEND == 'llm_api':
        for attempt in range(MAX_RETRIES):
            try:
                headers = {...}
                payload = {"model": model or OPENAI_EMBED_MODEL, "input": texts}
                r = requests.post(f"{OPENAI_BASE_URL}/embeddings", headers=headers, json=payload, timeout=EMBEDDING_TIMEOUT)
                r.raise_for_status()
                data = r.json()["data"]
                vectors = [item["embedding"] for item in data]
                return vectors
            except HTTPError as e:
                if r.status_code in (429, 500, 502, 503, 504) and attempt < MAX_RETRIES-1:
                    sleep_ms = BACKOFF_MS * (2 ** attempt) + random_jitter()
                    time.sleep(sleep_ms/1000)
                else:
                    raise
    else:
        return [get_embedding(t, model=model) for t in texts]

# Step 9 replacement
rows = list(df_context.itertuples(index=False))
for chunk in chunked(rows, BATCH):
    texts = [row.context_text for row in chunk]
    ids   = [row.patient_id for row in chunk]
    vecs  = batch_embed(texts)
    # assert len(vecs) == len(ids)
    execute_values(cur,
      "INSERT INTO patient_embeddings (patient_id, context_text, embedding) VALUES %s",
      list(zip(ids, texts, vecs))
    )
    conn.commit()
```

Error Handling & Idempotency
- Dimension mismatch: raise early if any returned vector length differs; prompt to set `EMBEDDING_DIM`.
- Partial failures: retry whole batch on 429/5xx with exponential backoff + jitter. If a batch repeatedly fails, fall back to embedding one-by-one to isolate the offender, then continue.
- Idempotency: use `ON CONFLICT (patient_id) DO UPDATE` to support resume without truncating the table. Optional, controlled by a flag.

Performance Tweaks
- Reuse a single `requests.Session()` for connection pooling and keep-alive.
- Tune `EMBED_BATCH_SIZE` based on provider limits (tokens, payload size). Start at 64 and adjust.
- Optional concurrency: if allowed, process multiple batches in parallel with a small thread pool; guard against rate limits.
- Use `execute_values` (or `COPY`) to reduce round-trips for inserts.

Caching (Optional but Recommended)
- Add a simple cache keyed by `hash(context_text)` to skip re-embedding unchanged rows.
- Store cache in a small table (`embedding_cache(hash, vector)`) or a local JSONL/SQLite file.

Rollout Steps
- Add `.env` knobs.
- Implement `batch_embed` and replace Step 9 loop with chunked ingestion.
- Keep original Step 9 code as an alternate cell (9b: Batched embeddings) initially for easy fallback.
- Update the guide (notes on batch size, retries, limits) and add a “Batching” tip in Troubleshooting.

Verification
- Dry-run on a small subset (e.g., first 50 patients) to validate counts and dimensions.
- Measure end-to-end time before/after; expect major gains on hosted backend.
- Confirm semantic search outputs are unchanged except for natural variation from model differences.

