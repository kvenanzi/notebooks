# Technical User Guide: `pgvector-synthetic.ipynb`

This guide walks through every numbered section of the `pgvector-synthetic.ipynb` notebook and explains how to operate it end-to-end. The workflow builds a synthetic patient analytics stack on top of PostgreSQL with the `pgvector` extension, generates embeddings with a local Ollama model, and layers semantic search plus visual analytics.

## Prerequisites
- **PostgreSQL** reachable at `localhost:5432` with a database named `patient_db` and credentials matching the connection cell (`user="kevin"`, `password="password123"`). Install the [`pgvector`](https://github.com/pgvector/pgvector) extension in this database.
- **Python environment** capable of installing packages listed below (the notebook relies on pip inside the runtime).
- **Data files** `data/patients.csv` and `data/allergies.csv` present relative to the notebook.
- **Ollama** running locally on the default port `11434` with the `phi4-mini` embedding model pulled and ready (`ollama pull phi4-mini`).

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

### 8. Local embedding function using Ollama φ4-mini
*Cell 15*  
Defines `get_ollama_embedding`, a thin wrapper around the Ollama embeddings HTTP API. It posts the `context_text` to `http://localhost:11434/api/embeddings` using the `phi4-mini` model and validates the response. A quick smoke test ensures the service returns vectors of the expected dimensionality (3072 in this setup).

### 9. Create and fill `patient_embeddings`
*Cell 17*  
Drops/recreates the `patient_embeddings` table with columns `(patient_id, context_text, embedding VECTOR(3072))`. Iterates through `df_context`, calls the Ollama embedding endpoint for each patient, and inserts the results. Re-running this cell after changing context generation will refresh the embeddings in-place.

### 10. Train UMAP reducer
*Cell 19*  
Pulls all embedding vectors from the database, converts them to `float32` numpy arrays, and fits a 2D [UMAP](https://umap-learn.readthedocs.io/) reducer (`metric="cosine"`). This produces `embedding_2d`, a low-dimensional layout capturing similarity structure among patients. The fitted reducer is stored in `reducer` for reuse (e.g., projecting query embeddings).

### 11. Semantic search query
*Cell 21*  
Demonstrates a basic vector similarity query using `pgvector`’s `<=>` operator. An example query string is embedded with Ollama, then Postgres computes cosine similarity (`1 - (embedding <=> query_vector)`) against all patients to return the top matches. Modify the query text or `LIMIT` clause to explore different cohorts.

### 12. Visualization (Matplotlib + Plotly)
*Cell 23*  
Plots the 2D UMAP embedding in Matplotlib for a quick glance and builds an interactive Plotly scatter (`hover_data` reveals patient IDs and context snippets). Use this to verify clustering behavior and to spot-check that similar patients are near one another.

### 12 (bis). Add NLP Semantic Search for Patients
*Cell 25*  
Introduces a higher-level semantic search pipeline:
- `embed_query_ollama` embeds user queries.
- `semantic_search_fused` orchestrates filter parsing, candidate selection, cosine similarity scoring, and result explanation.
The function prints a tabulated summary including demographic context, SDOH bucket, allergy samples, similarity scores, and which optional filters matched.

### 13. Filtering + fused semantic search
*Cell 27*  
Adds deterministic filters on top of the vector search:
- `parse_filters` infers gender, SDOH buckets, and mortality windows from natural-language cues.
- `candidate_ids_from_filters` executes SQL to pre-select patient IDs satisfying those filters before cosine scoring. This hybrid approach narrows the candidate set for faster, more relevant results.

### Run queries
*Cell 29*  
Calls `semantic_search_fused` with several example prompts. Each invocation prints a ranked table in the notebook so you can inspect output quality and the explanation trail.

### Project query into UMAP space (optional)
*Cell 30*  
Takes the last query embedding (`q_emb`), transforms it via the fitted UMAP reducer, and overlays the point on the patient embedding scatter plot. This helps visualize where the query vector sits relative to existing patient clusters.

## Operating tips
- If you restart the kernel, re-run the notebook from the top so that the database connection, helper functions, and reducer object are repopulated.
- Long-running steps (embedding generation, UMAP fitting) benefit from monitoring system load; consider batching or caching if working with larger patient cohorts.
- When adapting to production data, tighten data types in the table DDL and add foreign keys/indexes appropriate for your workload.
- To swap embedding models, update the Ollama endpoint/model name and adjust the `VECTOR(n)` dimension in the schema accordingly.

With this guide, you can trace how each section of the notebook transforms raw CSVs into a searchable, explainable semantic layer on top of Postgres + pgvector.

