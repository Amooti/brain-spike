# Second Brain — Claude Code Project

## What this is

A personal RAG-based second brain for indexing and querying coding session notes.
The primary use case is storing Claude Code session summaries so past decisions,
rationale, and patterns are retrievable by natural language query.

## Stack

- **Embeddings**: Ollama + `nomic-embed-text` (local, no API cost)
- **Vector DB**: ChromaDB (local, file-based, stored in `./chroma_db/`)
- **LLM**: Claude API (via `anthropic` Python SDK) — used at query time only
- **Orchestration**: LlamaIndex

## Repo structure

```
/sessions          ← markdown session notes (source of truth)
  /example         ← example notes to validate the pipeline
/ingestion         ← scripts to chunk, embed, and store notes
/query             ← CLI query tool
/chroma_db         ← local vector store (gitignored)
CLAUDE.md          ← you are here
README.md
requirements.txt
.gitignore
```

## Session note format

Every file in `/sessions` follows this template:

```markdown
# [Short title]
Date: YYYY-MM-DD
Project:
Tags: [comma-separated, e.g. maven, cicd, bitbucket]

## Problem
What were you trying to solve?

## What you tried
Brief notes on approaches and dead ends.

## Decision / outcome
What did you land on?

## Rationale
Why this over alternatives?

## Open questions
What is still unresolved?

## Key snippets
Any code worth preserving.
```

Chunking strategy: split by markdown heading (`##`), with metadata
(`filename`, `project`, `date`, `tags`) attached to each chunk.

## What needs to be built

### 1. `ingestion/ingest.py`

- Walk `/sessions` recursively, find all `.md` files
- Parse frontmatter-style metadata from the top block (Date, Project, Tags)
- Split each file into chunks by `##` heading
- Embed each chunk using Ollama (`nomic-embed-text`) via LlamaIndex
- Store in ChromaDB collection named `sessions`
- Skip files that haven't changed since last ingestion (use file mtime or a
  simple manifest file `ingestion/manifest.json`)

### 2. `query/query.py`

- Accept a natural language query as a CLI argument
- Embed the query using the same Ollama model
- Retrieve top-5 chunks from ChromaDB
- Send retrieved chunks + query to Claude API with a focused system prompt
- Print the answer with source filenames cited

### 3. `ingestion/summarize.py` (stretch goal)

- Accept a path to a Claude Code JSONL transcript
  (`~/.claude/projects/<project>/<session>.jsonl`)
- Parse the JSONL, extract user/assistant turns (skip tool noise)
- Send to Claude API with a prompt asking for a session note in the template
  format above
- Write the output to `/sessions/<project>/<date>-<slug>.md`
- This is the automation bridge from Claude Code sessions → second brain

## Environment setup

Requires:
- Python 3.11+
- Ollama installed and running locally with `nomic-embed-text` pulled
- `ANTHROPIC_API_KEY` environment variable set

Install Ollama: https://ollama.com/download
Pull the model: `ollama pull nomic-embed-text`

Install Python deps:
```bash
pip install -r requirements.txt
```

## Running

```bash
# Ingest all session notes
python ingestion/ingest.py

# Query
python query/query.py "how did I solve the Maven cache invalidation problem?"

# Summarize a Claude Code session transcript (stretch goal)
python ingestion/summarize.py ~/.claude/projects/myproject/abc123.jsonl
```

## Design decisions & rationale

- **Ollama over OpenAI embeddings**: zero ongoing cost, runs offline, good
  quality for this use case with `nomic-embed-text`
- **ChromaDB over Pinecone/Qdrant**: no infra, file-based, fits a personal tool
- **Markdown files as source of truth**: human-readable, git-friendly, editable
  without tooling
- **LlamaIndex for orchestration**: best RAG primitives in Python, good Ollama
  and ChromaDB integrations
- **Claude API only at query time**: keeps costs negligible (a few cents/month
  for personal use)
- **Heading-based chunking**: session notes have clear `##` sections; splitting
  on these keeps semantic units intact better than fixed-size chunking

## Owner context

Wes — experienced Java developer, comfortable with Python for tooling.
Background in distributed systems and web scraping. Active projects:
- Maven monorepo CI/CD with Bitbucket Pipelines (gitflow-incremental-builder)
- Retro game price tracker (`amooti.app`)
