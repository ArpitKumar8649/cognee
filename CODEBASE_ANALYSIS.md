# Cognee Codebase Analysis

## Overview

[Cognee](https://github.com/topoteretes/cognee) is a Python framework designed to build "AI memory" by combining vector stores, knowledge graphs, and Large Language Models (LLMs). It enriches LLM context with a semantic layer for better understanding and complex reasoning. By turning documents into a graph/vector knowledge base, it provides persistent, learning, and agentic memory for applications.

The framework provides an API for operations like `remember` (adding data to the knowledge base), `recall` (querying the knowledge base), `improve` (managing memory feedback/weights), and `forget` (removing data). It supports multiple database adapters, including various graph (Neo4j, Ladybug/Kuzu, Postgres, Neptune) and vector (PGVector, LanceDB, Qdrant) databases.

## Repository Structure

The repository is modularly organized. Below are the key directories and their purposes:

- **`cognee/`**: The core Python library.
  - **`api/`**: Contains the FastAPI application and versioned routers (e.g., `v1/add.py`, `v1/search/search.py`, `v1/session.py`, `v1/cognify.py`).
  - **`cli/`**: Command-line interface definitions and logic invoked via `cognee-cli`.
  - **`infrastructure/`**: Database adapters, LLM provider integrations (OpenAI, Anthropic, Gemini, Mistral, Ollama), embeddings, file storage adapters, and tokenizer utilities.
  - **`modules/`**: High-level domain logic.
    - `chunking/`: Splitting documents into smaller text chunks (TextChunker, LangchainChunker).
    - `pipelines/`: Orchestrates sequences of tasks for data processing, including telemetry and incremental loading.
    - `retrieval/`: Different search algorithms, e.g., Lexical (BM25), Graph Completion, Natural Language, and Agentic retrievers.
    - `search/`: Defines `SearchType`s and entrypoints for handling user queries (`get_retriever_output`).
  - **`tasks/`**: Reusable steps used in pipelines. Examples include `memify` (tracking frequency/feedback), `documents` (extracting chunks), and `graph` (generating knowledge graphs).
  - **`eval_framework/`**: Evaluation utilities.
  - **`shared/`**: Utilities like logging, data models, and configurations.
- **`cognee-mcp/`**: A Model Context Protocol (MCP) server that exposes Cognee as MCP tools (using stdio or HTTP/SSE transports). It allows LLM agents to interact directly with the Cognee memory store.
- **`cognee-frontend/`**: A Next.js web application providing a local UI for visualizing graphs, managing memory, and interacting with the system.
- **`distributed/`**: Scripts and utilities for running Cognee in distributed setups (Modal, Fly.io, etc.).
- **`alembic/`**: Contains database migration scripts for relational backends.
- **`examples/` & `notebooks/`**: Usage examples, Jupyter notebooks for tutorials and demonstrations of different search types and use cases.

## Core Features and Data Flow

### 1. Ingestion and `cognify`
The ingestion process starts by adding data (`cognee.add()` / `cognee.remember()`). The data undergoes a pipeline (`cognify`) where tasks are executed sequentially:
- **Chunking:** Raw documents are split into `DocumentChunk` objects (handled in `cognee/tasks/documents/`).
- **Graph Extraction:** Using an LLM, entities and relationships are extracted from chunks to build a knowledge graph.
- **Embedding:** Chunks and graph entities are embedded into vectors using the configured embedding engine (handled in `cognee/infrastructure/databases/vector/embeddings/`).
- **Storage:** Metadata is saved in a relational database, chunks and vectors in a vector store, and entities/relationships in a graph database.

### 2. Retrieval and Search
When querying (`cognee.recall()` / `/search`), Cognee utilizes different `SearchType` algorithms defined in `cognee/modules/search/types/SearchType.py`:
- **CHUNKS & CHUNKS_LEXICAL:** Pure vector similarity or BM25 keyword matching (`BM25ChunksRetriever`).
- **GRAPH_COMPLETION:** Uses the graph to find connected entities and context, then generates a completion via LLM.
- **AGENTIC_COMPLETION:** An intelligent agent mode that uses tools/skills to iteratively answer complex queries.
- **FEELING_LUCKY:** Intelligently auto-selects the best search strategy based on the query.

### 3. Agentic Memory (`memify`)
Cognee can track how often data is used. The `memify` tasks (`cognee/tasks/memify/`) update graph elements (nodes and edges) with **frequency weights** and **feedback weights**. If a graph component is often retrieved to answer questions successfully, its weight increases, optimizing future retrievals. This mechanism is central to building persistent, self-improving agents.

### 4. Telemetry and Observability
The codebase includes built-in tracing (`cognee/modules/observability/`) using OpenTelemetry. It tracks `Pipeline Run Started`, `Task Completed`, span durations, and error rates, which is crucial for monitoring ingestion and search pipelines in production environments.

## Conclusion

Cognee acts as an all-in-one "memory infrastructure" for LLMs, handling the complexities of chunking, embedding, graphing, and retrieving data seamlessly. Its modular architecture permits developers to swap out underlying technologies (e.g., using Kuzu vs Neo4j, OpenAI vs local Ollama) while maintaining a consistent memory layer.
