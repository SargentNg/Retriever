# The Retriever

Local, private, hybrid (keyword + vector) semantic search over the entire OneDrive.
Embeddings via local Ollama (`nomic-embed-text`), exact KNN (numpy/BLAS), SQLite + FTS5
storage. OneDrive is treated strictly read-only and is never copied fully to disk.

CLI: `retriever index | query | status | gc`.

**Status: design handoff, not built yet.** Read `DESIGN.md` end-to-end and build per it.
The first line of a build session is: "Read DESIGN.md and build The Retriever per it."

Distinct from `~/opencode/Archivist` (the scan → classify → name → OneDrive pipeline):
separate repo, separate venv, separate services; they share nothing at runtime.