The `pgvector-synthetic.ipynb` notebook bootstraps a synthetic patient analytics pipeline: it ingests fictional patients and allergies into PostgreSQL, normalizes tables, builds per‑patient context paragraphs, generates embeddings, and layers semantic search and visualization on top of `pgvector`.

Key points
- Dual embedding backends: run locally via Ollama (`phi4-mini` by default) or call a hosted, OpenAPI‑compatible embedding service using an API key.
- pgvector storage: embeddings are stored in Postgres and queried with cosine similarity; UMAP provides a 2D layout for quick visual inspection.
- Fused search: natural‑language filters + vector similarity to surface relevant synthetic patients.

Quick start
1) Prerequisites
   - Postgres with `pgvector` available at `localhost:6432` (use `docker-compose up -d` in this folder for a quick start).
   - A Python environment capable of installing notebook dependencies.
2) Configure environment
   - Create `pgvector-synthetic/.env` (already git‑ignored). Minimal examples:
     - Local Ollama
       EMBEDDING_BACKEND=ollama
       OLLAMA_EMBED_MODEL=phi4-mini
     - Hosted API
       EMBEDDING_BACKEND=llm_api
       OPENAI_API_KEY=YOUR_KEY
       OPENAI_BASE_URL=https://api.openai.com/v1
       OPENAI_EMBED_MODEL=text-embedding-3-small
   - The notebook auto‑loads this file via `python-dotenv`.
3) Run the notebook
   - Start Jupyter, open `pgvector-synthetic.ipynb`, and run top‑to‑bottom. Cell 8a prints the active backend and settings.

Switching backends
- Change `EMBEDDING_BACKEND` in `.env` (`ollama` or `llm_api`) and restart the kernel; you do not need to remove your API key.
- If the hosted model’s vector length differs, set `EMBEDDING_DIM` to the integer size; otherwise the notebook will infer it on first embed and size the `VECTOR(n)` column accordingly.

Performance tips (hosted backend)
- Batch multiple inputs per request to amortize network latency.
- Cache embeddings (disk or DB) and only re‑embed changed rows.
- Prefer lighter embedding models when latency matters.
