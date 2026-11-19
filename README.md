# rag-project

A Retrieval-Augmented Generation (RAG) service built step by step, from primitives up, as a learning project.

## Run locally

    uv sync
    uv run uvicorn rag.main:app --reload

## Checks

    uv run ruff check .
    uv run ruff format --check .
    uv run pytest
