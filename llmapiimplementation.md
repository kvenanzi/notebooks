# Plan for Integrating OpenAPI LLM Embedding Support

## Goals
- Allow `pgvector-synthetic.ipynb` to switch between local Ollama embeddings and an external OpenAPI-compatible LLM embedding service using an API key.
- Introduce alternate workflow steps (8a, 9a, 10a, etc.) for the LLM-powered path while preserving the existing Ollama-based workflow.
- Ensure configuration, credential management, and downstream data handling support both embedding sources consistently.

## Assumptions & Constraints
- The OpenAPI provider exposes an embedding endpoint compatible with OpenAI's API schema.
- API keys will be stored securely via environment variables or notebook secrets, not hard-coded.
- Existing notebook functions (data loading, vector insertion, querying) can be adapted or conditionally invoked based on the chosen embedding source.

## Proposed Changes

### 1. Configuration & Environment Handling
- Add instructions in the notebook for setting an `OPENAI_API_KEY` (or provider-specific) environment variable, plus optional provider URL overrides.
- Provide a new configuration cell (Step 8a) that toggles between `"ollama"` and `"llm_api"` embedding backends.

### 2. Embedding Client Abstraction
- Refactor existing embedding call logic into helper functions or classes that encapsulate backend-specific details.
- Implement a new helper for calling the OpenAPI embedding endpoint (e.g., using `requests` or `openai` Python SDK) with configurable model name and headers.
- Ensure helper returns vectors in the same format (list of floats) as the Ollama implementation.

### 3. Notebook Workflow Updates
- Introduce Step 8a to collect API credentials, select backend, and instantiate the appropriate embedding client.
- Duplicate downstream steps (9a, 10a, etc.) only where behavior diverges between backends; otherwise reuse shared functions parameterized by backend choice.
- Update narrative text and markdown cells to explain both workflows and when to use each.

### 4. Data Processing Adjustments
- Audit existing steps that rely on Ollama-specific metadata or response schema; adjust to consume generalized embedding responses.
- Ensure data insertion into PostgreSQL/pgvector remains unchanged regardless of vector source.

### 5. Error Handling & Validation
- Add checks for missing API keys or failed embedding responses, with clear user guidance.
- Document rate limiting or cost considerations for LLM API usage.

### 6. Testing & Verification
- Provide notebook cells to validate connectivity to the LLM API (e.g., test embedding call) before running full pipeline.
- Confirm that vector similarity queries return results for both backends.

## Task Breakdown

### Task 1: Introduce Backend Configuration (Step 8a)
- Add markdown and code cells for selecting embedding backend and loading credentials.
- Ensure environment variables are read securely (e.g., via `os.getenv`).

### Task 2: Implement Embedding Client Abstraction
- Move existing Ollama embedding code into a reusable function.
- Add new function to call external LLM API embeddings using API key, endpoint, and model parameters.
- Provide backend selection logic that dispatches to the correct function.

### Task 3: Update Downstream Steps (9a, 10a, ...)
- Review existing steps that create embeddings, store vectors, or query them.
- Adjust code to call backend-agnostic helper functions and update markdown explanations.

### Task 4: Enhance Error Handling & Documentation
- Add validation for API key presence when `llm_api` backend is selected.
- Document usage considerations (rate limits, costs, expected response format).
- Include troubleshooting notes for common errors.

### Task 5: Testing Instructions
- Outline manual test steps to verify both Ollama and LLM API embedding workflows.
- Encourage re-running similarity queries to compare outputs.

## Open Questions
- Should the backend selection be persisted (e.g., via config file) or chosen per session?
- Are there provider-specific differences (token limits, dimensions) that require schema adjustments in PostgreSQL?
- Is there a need to support multiple LLM providers beyond the initial OpenAPI-compatible service?
