# Documentation Search

[![Python Version](https://img.shields.io/badge/python-3.10%20%7C%203.11%20%7C%203.12-blue?logo=python&logoColor=white)](https://www.python.org/)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.110+-009688?logo=fastapi&logoColor=white)](https://fastapi.tiangolo.com)
[![ChromaDB](https://img.shields.io/badge/ChromaDB-Vector%20Store-orange?logo=databricks&logoColor=white)](https://www.trychroma.com/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Code Style: Black](https://img.shields.io/badge/code%20style-black-000000.svg)](https://github.com/psf/black)
[![Tests](https://img.shields.io/badge/tests-pytest-blueviolet?logo=pytest&logoColor=white)](https://pytest.org/)

**Documentation Search** is a high-precision, Retrieval-Augmented Generation (RAG) service designed to index internal technical documentation, Markdown repositories, architecture decision records (ADRs), and developer guides into a vector store. It exposes both a command-line interface and a robust FastAPI web service that returns semantically synthesized answers paired with granular, inline citations, exact source line references, and deep-linked document locations.

---

## 1. Problem Statement & Target Audience

### Problem Statement
Engineering organizations routinely suffer from knowledge fragmentation. Critical operational runbooks, architecture designs, API specifications, and setup instructions are scattered across multi-repository Markdown directories, internal wikis, and PDF manuals. 

Standard lexical/keyword search fails because:
* It cannot resolve semantic intent, synonyms, or conceptual queries (e.g., searching for "rate limit configuration" misses files discussing "request throttling windows").
* Search engines point to monolithic files without pinpointing exact line numbers or sections, forcing engineers to manually scan hundreds of lines of documentation.
* Generic LLM chat interfaces hallucinate obsolete or generic patterns without grounding answers in company-specific technical conventions.

### Target Audience
* **Platform & Infrastructure Teams:** Maintaining extensive runbooks, Kubernetes manifests, and internal RFCs.
* **Software Engineering Teams:** Navigating sprawling monorepos containing nested `docs/` folders and architecture decision records (ADRs).
* **Technical Writers & Developer Relations:** Auditing documentation coverage and surfacing contextual answers to technical users.

---

## 2. Key Features

* **Header-Aware Semantic Chunking:** Ingests Markdown (`.md`) and Portable Document Format (`.pdf`) files while parsing semantic document structure (H1, H2, H3 headers). Preserves precise line number tracking (`start_line` to `end_line`) inside chunk metadata.
* **Vector Store Persistence via ChromaDB:** Manages embedding generation and collection lifecycle with on-disk ChromaDB vector storage, ensuring sub-50ms nearest-neighbor semantic retrieval.
* **Grounded Query Synthesis with Inline Citations:** Queries OpenAI LLMs using retrieved context windows, formatting synthesized markdown answers embedded with explicit citation tags (e.g., `[^1]`) mapped directly to document paths and source line intervals.
* **Strict Pydantic Validation & Filtering:** Implements strict data models across all API payloads and ingestion metadata, enforcing configurable similarity score thresholds (`min_score`) to prune low-confidence retrievals before LLM synthesis.

---

## 3. Architecture & System Design

The system runs a decoupled two-phase lifecycle: **Ingestion** and **Retrieval & Synthesis**.

```
INGESTION PIPELINE:
  [ Markdown / PDF Files ]
             │
             ▼
    [ src/core.py: Document Parser ] 
             │  (Extracts raw text, maps file paths & line numbers)
             ▼
    [ Header-Aware Chunking Engine ]
             │  (Preserves # H1 / ## H2 boundaries & line intervals)
             ▼
    [ OpenAI Embeddings ]
             │  (Converts text chunks into dense vector representations)
             ▼
    [ ChromaDB Vector Store ] ─── (Persistent disk storage)

QUERY & SYNTHESIS PIPELINE:
    [ User Query / API Request ]
             │
             ▼
    [ src/api.py: FastAPI Route /api/v1/query ]
             │  (Validates payload via src/models.py)
             ▼
    [ ChromaDB Similarity Search ]
             │  (Retrieves Top-K chunks filtered by similarity threshold)
             ▼
    [ OpenAI Synthesis Engine ]
             │  (Generates structured context answer)
             ▼
    [ Citation & Source Line Resolver ]
             │  (Attaches inline citations: document path, start_line, end_line)
             ▼
    [ Structured JSON Response ]
```

### Module Responsibilities

| Module | File | Purpose |
| :--- | :--- | :--- |
| **Data Contracts** | `src/models.py` | Defines Pydantic models for request/response payloads, document chunk metadata, query options, citation representations, and configuration schemas. |
| **Core Engine** | `src/core.py` | Contains core pipeline logic: Markdown/PDF document parsing, header-aware semantic chunking with line number tracking, ChromaDB vector store client management, and OpenAI synthesis engine with inline citation building. |
| **API Layer** | `src/api.py` | FastAPI router implementation defining API endpoints for health checking, document ingestion, vector search query synthesis, and collection metadata inspection. |
| **Entry Point & CLI** | `src/main.py` | Application entry point managing environment initialization, Uvicorn server execution, and CLI commands for batch file ingestion. |

---

## 4. Technology Stack

* **Language:** Python 3.10+
* **Web Framework:** FastAPI (ASGI via Uvicorn)
* **Vector Database:** ChromaDB
* **Orchestration:** LangChain Core
* **Data Validation:** Pydantic v2
* **LLM & Embeddings:** OpenAI (`text-embedding-3-small`, `gpt-4o-mini` / `gpt-4o`)
* **Testing:** Pytest, pytest-asyncio, pytest-mock

---

## 5. Installation & Setup

### Prerequisites
* Python 3.10 or higher
* Valid OpenAI API Key

### Step-by-Step Installation

1. **Clone the repository:**
   ```bash
   git clone https://github.com/your-org/documentation-search.git
   cd documentation-search
   ```

2. **Create and activate a virtual environment:**
   ```bash
   # Linux/macOS
   python3 -m venv .venv
   source .venv/bin/activate

   # Windows (PowerShell)
   python -m venv .venv
   .venv\Scripts\Activate.ps1
   ```

3. **Install dependencies:**
   ```bash
   pip install --upgrade pip
   pip install -r requirements.txt
   ```

---

## 6. Environment Variables Configuration

Create a local `.env` file by copying the distributed `.env.example`:

```bash
cp .env.example .env
```

Configure the environment parameters according to your deployment:

| Variable | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `OPENAI_API_KEY` | String | *Required* | Secret API key for OpenAI embedding and synthesis models. |
| `CHROMA_PERSIST_DIR` | String | `./data/chroma` | Local directory path where ChromaDB persists indices. |
| `EMBEDDING_MODEL` | String | `text-embedding-3-small` | OpenAI model identifier used for vector embeddings. |
| `SYNTHESIS_MODEL` | String | `gpt-4o-mini` | OpenAI LLM identifier used for answer synthesis. |
| `API_HOST` | String | `0.0.0.0` | Host interface for the Uvicorn web server. |
| `API_PORT` | Integer | `8000` | Port for the Uvicorn web server. |
| `DEFAULT_MIN_SCORE` | Float | `0.65` | Default minimum similarity score threshold (0.0 to 1.0). |

---

## 7. Usage: CLI & API Execution

### Starting the Web API
Run the Uvicorn server through the application entry point:

```bash
python -m src.main serve --host 0.0.0.0 --port 8000 --reload
```

The interactive OpenAPI documentation is available at `http://localhost:8000/docs`.

### CLI Document Ingestion
You can batch ingest directories containing Markdown and PDF documentation directly via the CLI:

```bash
# Ingest all docs from a local folder into ChromaDB
python -m src.main ingest --path ./docs

# Ingest a specific file
python -m src.main ingest --path ./docs/architecture/adr-004.md
```

---

### REST API Endpoints

#### 1. System Health Check
* **Endpoint:** `GET /health`
* **Response:**
  ```json
  {
    "status": "healthy",
    "timestamp": "2025-02-17T12:00:00Z"
  }
  ```

#### 2. Ingest Document Files
* **Endpoint:** `POST /api/v1/documents/ingest`
* **Payload:**
  ```json
  {
    "file_path": "./docs/networking/ingress.md",
    "collection_name": "internal-docs"
  }
  ```
* **Response:**
  ```json
  {
    "status": "success",
    "document_id": "doc_9f81a1c4",
    "chunks_indexed": 6,
    "file_path": "./docs/networking/ingress.md"
  }
  ```

#### 3. Vector Search & Answer Synthesis
* **Endpoint:** `POST /api/v1/query`
* **Payload:**
  ```json
  {
    "query": "How do we configure zero-downtime rolling deploys for the ingress controller?",
    "collection_name": "internal-docs",
    "top_k": 3,
    "min_score": 0.70
  }
  ```
* **Response:**
  ```json
  {
    "query": "How do we configure zero-downtime rolling deploys for the ingress controller?",
    "answer": "To configure zero-downtime rolling updates for the ingress controller, set `maxSurge: 25%` and `maxUnavailable: 0` inside the Deployment spec [^1]. Additionally, implement a preStop lifecycle hook with an 8-second sleep to allow upstream load balancers to unregister the terminating pod [^2].",
    "citations": [
      {
        "citation_id": 1,
        "source": "./docs/networking/ingress.md",
        "header_hierarchy": ["Ingress Controller", "Deployment Strategy"],
        "start_line": 42,
        "end_line": 56,
        "similarity_score": 0.892
      },
      {
        "citation_id": 2,
        "source": "./docs/networking/ingress.md",
        "header_hierarchy": ["Ingress Controller", "Graceful Termination"],
        "start_line": 88,
        "end_line": 95,
        "similarity_score": 0.841
      }
    ]
  }
  ```

#### 4. Collection Metadata Inspection
* **Endpoint:** `GET /api/v1/collections/stats?collection_name=internal-docs`
* **Response:**
  ```json
  {
    "collection_name": "internal-docs",
    "total_documents": 48,
    "total_chunks": 312,
    "persist_directory": "./data/chroma"
  }
  ```

---

## 8. Automated Testing

The suite uses `pytest` alongside mock adapters to validate chunking accuracy, score filtering, and API contracts without invoking live OpenAI endpoints during continuous integration.

```bash
# Run all test suites
pytest

# Run tests with verbose output and coverage
pytest -v --cov=src --cov-report=term-missing

# Run core engine tests specifically
pytest tests/test_core.py -v

# Run FastAPI endpoint contract tests
pytest tests/test_api.py -v
```

---

## 9. Project Structure

```
.
├── .env.example              # Template environment variable configuration
├── .gitignore                # Git ignore patterns for Python, virtual environments, and ChromaDB
├── requirements.txt          # Production and development Python dependencies
├── src/
│   ├── __init__.py           # Package marker for src module
│   ├── models.py             # Pydantic models for validation, citations, and requests/responses
│   ├── core.py               # Parsing, header chunking, line tracking, ChromaDB & OpenAI pipeline
│   ├── api.py                # FastAPI routes, error handlers, and lifecycle hooks
│   └── main.py               # CLI command interface and Uvicorn server entry point
└── tests/
    ├── __init__.py           # Package marker for tests module
    ├── test_core.py          # Unit tests for chunking, line tracking, and synthesis logic
    └── test_api.py           # Integration tests for FastAPI endpoints and Pydantic schemas
```

---

## 10. Roadmap & Future Enhancements

- [ ] **Cross-Encoder Re-ranking:** Introduce a secondary re-ranking stage (e.g., `bge-reranker-large`) to optimize top-k precision prior to LLM synthesis.
- [ ] **Automated Git Sync:** Webhook integration for GitHub/GitLab to automatically re-index modified Markdown files on merge events.
- [ ] **Source Document Anchoring:** Extend API responses with deep-link URI schemas compatible with VS Code, JetBrains IDEs, and GitHub web source views (e.g., `file.md#L42-L56`).
- [ ] **Hybrid Lexical Search:** Combine BM25 sparse lexical search with ChromaDB dense vector embeddings for hybrid retrieval over technical acronyms and exact error codes.

---

## 11. License

This project is licensed under the terms of the **MIT License**. See the [LICENSE](LICENSE) file for complete details.