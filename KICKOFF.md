# KICKOFF — build-session brief (The Retriever)

You are a fresh session with no prior context. Everything you need is in this repo.
Owner: SargentNg. Dated 2026-11-26.

## Mission

Build **The Retriever**: a local, private, hybrid (keyword + vector) semantic search over
**the entire OneDrive** (rclone remote `onedrive:`). Read `DESIGN.md` end-to-end FIRST — it
locks every decision (embeddings, chunking, storage schema, CLI, systemd, acceptance
criteria). Don't relitigate those decisions without the owner.

## Current state

- Design committed & pushed to `SargentNg/TheRetriever` (branch `main`).
- **No code exists yet**: no `.venv`, no `var/`, no systemd units. Build order is DESIGN §11.
- Full-drive inventory numbers are pending — gather them as step 0 below.

## Environment (verified 2026-11-26; full list in DESIGN §10)

- Ollama on `127.0.0.1:11434`, running; `nomic-embed-text:latest` pulled (768-dim,
  `POST /api/embed` tested OK). Use it — don't substitute other models.
- rclone remote `onedrive:` authed & working. This rclone build's quirks: no `size --json`;
  no `lsf --fs-only` (use `-R --files-only` or `-R --json`); no `--allow-overwrite` flag;
  `move` without a trailing-slash dest creates a dir named after the file.
- python3.12, `/usr/bin/tesseract`, `/usr/bin/pdftotext` installed. GPUs: 4× RTX 3090 +
  1× RTX 3080 (embeddings need no GPU management). ~378 GB free on `/`.
- Systemd-user unit style: copy from `~/.config/systemd/user/archivist*.{service,timer}`.
- Git: identity already global (`SargentNg <24596997+SargentNg@users.noreply.github.com>`);
  push over SSH (`git push`). The `gh` CLI is **not** authed on this box — don't use it.

## Hard rules

1. **OneDrive is read-only. Forever.** Never write, rename, or delete anything in it.
2. `~/opencode/Archivist` (the sibling scan pipeline) is a **read-only reference** —
   tesseract/PSM6 recipes, rclone quirks, unit style. Never modify it or its venv.
3. **Zero document content in the repo.** `var/` and `.venv/` are gitignored. Only code,
   docs, `INVENTORY.md`, `QUERY_TESTS.md` get committed.
4. When in doubt, stop and ask the owner. Don't guess on anything touching OneDrive.

## First move — inventory (step 0, it takes minutes, so detach it)

```bash
mkdir -p var
nohup bash -c 'rclone lsf -R --files-only "onedrive:" > var/od-listing.txt && rclone size "onedrive:" > var/INVENTORY.txt 2>&1 && wc -l < var/od-listing.txt >> var/INVENTORY.txt' >/dev/null 2>&1 &
```

When it finishes, `var/od-listing.txt` holds the complete file list (gitignored cache —
reuse it for sanity checks; `list_od.py` may consume it instead of re-listing). Derive an
extension histogram from it (last dot-segment, casefolded, counts sorted desc, top 25) and
commit an `INVENTORY.md` with: total size, file count, top extensions, and your
indexable-file estimate (feeds DESIGN §8 budget).

## Then

Follow DESIGN §11 in order (venv + store + CLI scaffold → `list_od`/`extract`/`chunk` →
`embed` + single-file smoke test → full index → `query` → systemd units → viewer-facing
docs → push). Phase-2 OCR (`--ocr`) only after the owner approves a 10-file trial batch.
Acceptance checklist: DESIGN §9 — do not call it done until every item passes.

## Owner decisions already made

- Repo name: `TheRetriever` (created; tracking `origin/main`).
- Scope: **OneDrive only** for now; other local folders later via config roots.
- Timer default: every 6 h (owner may adjust).
- Embedding model, storage layout, CLI shape: locked in DESIGN §2 — don't change silently.