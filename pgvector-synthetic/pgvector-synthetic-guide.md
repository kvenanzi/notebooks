# Technical User Guide: `pgvector-synthetic.ipynb`

This guide walks through every numbered section of the `pgvector-synthetic.ipynb` notebook and explains how to operate it end-to-end. The workflow builds a synthetic patient analytics stack on top of PostgreSQL with the `pgvector` extension, generates embeddings from either a local Ollama model or an OpenAPI-compatible hosted service, and layers semantic search plus visual analytics.

## Prerequisites
- **PostgreSQL** reachable at `localhost:6432` with a database named `patient_db` and credentials matching the connection cell (`user="kevin"`, `password="password123"`). Install the [`pgvector`](https://github.com/pgvector/pgvector) extension in this database. Run the docker-compose file in the project root for a quick install.
- **Python environment** capable of installing packages listed below (the notebook relies on pip inside the runtime).
- The notebook auto-loads environment variables from `pgvector-synthetic/.env` via `python-dotenv`.
- **Data files** `data/patients.csv` and `data/allergies.csv` present relative to the notebook.
- **Ollama** running locally on the default port `11434` with the `phi4-mini` embedding model pulled and ready (`ollama pull phi4-mini`).
- **Optional hosted embeddings** When selecting the `llm_api` backend, export `OPENAI_API_KEY` and (if needed) `OPENAI_BASE_URL`, `OPENAI_EMBED_MODEL`, `OPENAI_ORG`, plus `EMBEDDING_DIM` if the remote vector dimensionality differs.

## Step-by-step breakdown

### 1. Install dependencies
*Notebook cell 1 (`!pip install …`)*  
Installs all third-party libraries required by later sections: PostgreSQL client (`psycopg2-binary`), data wrangling (`pandas`, `numpy`), visualization (`matplotlib`, `plotly`), synthetic-data helpers (`faker`), HTTP client (`requests`), dimensionality reduction (`umap-learn`), pretty tables (`tabulate`), and assorted utilities.

### 2. Imports
*Cell 3*  
Collects the Python modules used throughout the notebook. Ensure this executes cleanly before proceeding; any missing package means the install cell did not run in the same kernel session.

### 3. Connect to PostgreSQL with pgvector
*Cell 5*  
Creates a `psycopg2` connection using the hard-coded credentials. Immediately enables the `vector` extension (`CREATE EXTENSION IF NOT EXISTS vector`). Update the connection arguments if your environment differs, then re-run the cell.

### 4. Load CSVs
*Cell 7*  
Reads `patients.csv` and `allergies.csv` into pandas dataframes. The `display()` calls preview the first few records so you can sanity-check schema alignment and data quality before loading anything into Postgres.

### 5. Create normalized tables
*Cell 9*  
Drops and recreates the `patients` and `allergies` tables. The schema keeps patient demographics, SDOH (social determinants of health) features, genetic risk fields, and care-plan metrics in `patients`, while `allergies` houses reaction-level detail keyed by `patient_id`. Dates are stored as `TEXT` for flexibility when ingesting CSV strings. Adjust column types here if your source data uses stricter formats.

### 6. Bulk-insert dataframes into Postgres
*Cell 11*  
Defines `copy_dataframe`, a helper that streams a pandas dataframe into Postgres via `COPY ... FROM STDIN` for efficient batch loads. Before loading:
- Normalizes nulls (`NaN` → `None`) and converts date columns to strings.
- Filters allergies to patient IDs present in the patients dataframe to avoid FK violations.
Running the cell populates the two tables and prints row counts so you can confirm ingestion volumes.

### 7. Join data into patient “context” paragraphs
*Cell 13*  
Runs a SQL `WITH` query that aggregates demographics, SDOH risk buckets, insurance, lifestyle flags, mortality signals, and summarized allergies per patient. The result is a dataframe `df_context` with one row per patient and a human-readable `context_text` string. This textual synopsis is the basis for embedding generation and semantic search in later steps.

### 8a. Configure embedding backend
*Cell 15*  
Chooses between the local `"ollama"` backend and the hosted `"llm_api"` option. When the hosted path is selected the cell verifies that `OPENAI_API_KEY` is present, applies optional overrides (`OPENAI_BASE_URL`, `OPENAI_EMBED_MODEL`, `OPENAI_ORG`, `EMBEDDING_DIM`), and allows you to skip the automatic smoke test via `SKIP_EMBEDDING_SMOKETEST=1`.

### 8b. Embedding helper functions
*Cell 16*  
Wraps both embedding providers behind `get_embedding`. The helper dispatches to the correct HTTP endpoint, normalizes responses to Python lists, and enforces a consistent vector length. When the smoke test is enabled it immediately calls the selected backend to surface configuration issues early.

### 9. Create and fill `patient_embeddings`
*Cell 19*  
Discovers the embedding dimensionality (using the smoke-test result or the first context embedding) before creating the table so it works with either backend. Iterates through `df_context`, calls `get_embedding` for each row, and inserts the vectors. Re-running this cell after changing context generation will refresh the embeddings in-place.

### 10. Train UMAP reducer
*Cell 21*  
Pulls all embedding vectors from the database, converts them to `float32` numpy arrays, and fits a 2D [UMAP](https://umap-learn.readthedocs.io/) reducer (`metric="cosine"`). This produces `embedding_2d`, a low-dimensional layout capturing similarity structure among patients. The fitted reducer is stored in `reducer` for reuse (e.g., projecting query embeddings).

### 11. Semantic search query
*Cell 23*  
Demonstrates a basic vector similarity query using `pgvector`’s `<=>` operator. An example query string is embedded through the selected backend, then Postgres computes cosine similarity (`1 - (embedding <=> query_vector)`) against all patients to return the top matches. Modify the query text or `LIMIT` clause to explore different cohorts.

### 12. Visualization (Matplotlib + Plotly)
*Cell 25*  
Plots the 2D UMAP embedding in Matplotlib for a quick glance and builds an interactive Plotly scatter (`hover_data` reveals patient IDs and context snippets). Use this to verify clustering behavior and to spot-check that similar patients are near one another.

### 12 (bis). Add NLP Semantic Search for Patients
*Cell 27*  
Introduces a higher-level semantic search pipeline:
- `embed_query_vector` embeds user queries through the selected backend.
- `semantic_search_fused` orchestrates filter parsing, candidate selection, cosine similarity scoring, and result explanation.
The function prints a tabulated summary including demographic context, SDOH bucket, allergy samples, similarity scores, and which optional filters matched.

### 13. Filtering + fused semantic search
*Cell 29*  
Adds deterministic filters on top of the vector search:
- `parse_filters` infers gender, SDOH buckets, and mortality windows from natural-language cues.
- `candidate_ids_from_filters` executes SQL to pre-select patient IDs satisfying those filters before cosine scoring. This hybrid approach narrows the candidate set for faster, more relevant results.

### Run queries
*Cell 31*  
Calls `semantic_search_fused` with several example prompts. Each invocation prints a ranked table in the notebook so you can inspect output quality and the explanation trail.

### Project query into UMAP space (optional)
*Cell 32*  
Takes the last query embedding (`q_emb`), transforms it via the fitted UMAP reducer, and overlays the point on the patient embedding scatter plot. This helps visualize where the query vector sits relative to existing patient clusters.

## Operating tips
- If you restart the kernel, re-run the notebook from the top so that the database connection, helper functions, and reducer object are repopulated.
- Long-running steps (embedding generation, UMAP fitting) benefit from monitoring system load; consider batching or caching if working with larger patient cohorts.
- When adapting to production data, tighten data types in the table DDL and add foreign keys/indexes appropriate for your workload.
- To swap embedding models or providers, update `EMBEDDING_BACKEND` (and related environment variables) and ensure the `VECTOR(n)` column matches the returned dimensionality.

With this guide, you can trace how each section of the notebook transforms raw CSVs into a searchable, explainable semantic layer on top of Postgres + pgvector.

## Troubleshooting
- Backend won’t switch to hosted: Ensure `.env` sets `EMBEDDING_BACKEND=llm_api` and rerun the kernel. Cell 2/4 must run so `python-dotenv` loads `.env` before cell 8a prints settings.
- Missing API key: Cell 8a raises an error if `llm_api` is selected without `OPENAI_API_KEY`. Add it to `.env` (no quotes/extra spaces) or export it in your shell.
- Remote calls are very slow: Batch multiple texts per embeddings request, use a lighter model (e.g., `text-embedding-3-small`), and cache vectors so only new/changed rows call the API. Long runtimes happen if Step 9 embeds one row at a time over the network.
- Vector length mismatch: If you see “Embedding length mismatch…”, set `EMBEDDING_DIM` in `.env` to the correct integer for your model, then rerun Step 9 to recreate the `VECTOR(n)` column.
- Dimension inference got “stuck”: When `SKIP_EMBEDDING_SMOKETEST=1`, the first produced embedding defines the size. If you change models afterward, drop/recreate `patient_embeddings` by rerunning Step 9.
- Postgres connection fails: Verify the DB is up (try `docker-compose up -d`), the host/port match `localhost:6432`, and cell 5 succeeded in `CREATE EXTENSION vector`.
- Rate limits/429s from provider: Add a small delay and retry logic, reduce batch size, or throttle requests to stay under limits.
