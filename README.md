# second-brain

A personal RAG pipeline for indexing coding session notes and querying them
with natural language.

## Quick start

1. Install Ollama: https://ollama.com/download
2. Pull the embedding model: `ollama pull nomic-embed-text`
3. Install Python deps: `pip install -r requirements.txt`
4. Set your API key: `export ANTHROPIC_API_KEY=your_key_here`
5. Ingest notes: `python ingestion/ingest.py`
6. Query: `python query/query.py "your question here"`

## Adding session notes

Drop a `.md` file into `/sessions/<project>/` using the template in `CLAUDE.md`.
Re-run `python ingestion/ingest.py` to pick it up.

## Automating from Claude Code

See `ingestion/summarize.py` — point it at a Claude Code JSONL transcript and
it will generate a structured session note automatically.
