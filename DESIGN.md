# OneSearch — design & build handoff

Status: **DESIGN ONLY — not built yet.** This file is the contract. A new opencode session
(working directory = this repo) reads it end-to-end, then builds it. Owner: SargentNg. Date: 2026-11-26.

## 1. What it is

Local, private, hybrid (keyword + vector) search over **the entire OneDrive** (rclone remote
`onedrive:`). Query from the terminal with natural language ("that HOA letter about the reserve
study"), exact strings, or filenames; get ranked file paths + snippet. No data leaves this machine
except rclone→OneDrive, which is how the drive is already accessed.

Non-goals (do not build now): full-drive local copy, web UI, ANN/HNSW libraries, video/audio
indexing, other local folders (config structure must support adding roots later; owner said
OneDrive-only for now), HEIC OCR, .msg/.ost/.pst.

**OneDrive is read-only, forever. Never write, rename, or delete anything in it.**

## 2. Locked decisions

1. **Embeddings = Ollama `nomic-embed-text:latest`** (768-dim, 8192 ctx, 274 MB, already pulled;
   `POST http://127.0.0.1:11434/api/embed` verified working, returns 768-dim). Model string is
   pinned in the `kv` table; a model change is a full re-embed migration (fresh .npy, then swap),
   never in-place. Use stdlib `urllib` for the HTTP call — no Ollama SDK.
2. **No vector DB.** Exact KNN: load all vectors (mmap'ed float32 array) + BLAS dot products.
   At this scale (≤ ~1M chunks) it's faster, simpler, and lossless vs HNSW; adds zero deps.
3. **SQLite (WAL) + one `.npy` sidecar** hold the index — schema in §7.
4. **Chunking:** ~900 chars, 120-char overlap, prefer paragraph/sentence boundaries. One chunk
   pool per extracted unit (PDF page, xlsx sheet, docx body, .eml HTML/stripped body, text file
   sections).
5. **Hybrid score** = `0.5*cos + 0.5*BM25norm` (default). `--vectors` / `--keywords` for pure
   modes. Expand top-20 chunks → top-10 unique files (best chunk wins per file).
6. **OneDrive access = rclone `onedrive:` read-only** (remote exists, OAuth working). Stream each
   file with `rclone cat` into a blank temp file — no full-drive local copy. Incremental diff from
   `rclone lsf -R --json` (fields: Path, Size, ModTime).
7. **Caps:** files > 100 MB → `skipped_size`. Extracted text < 10 chars → `no_text`.
8. **Type dispatch (tier 1):**
   - `.pdf` — PyMuPDF text layer (our 31 Archivist docs in `Family Vault/Scans` have embedded
     OCR text layers — they come out free);
   - `.docx .xlsx .pptx` — python-docx / openpyxl / python-pptx;
   - `.eml` — stdlib `email` (family mail folders exist);
   - plain text family: `.txt .md .csv .tsv .log .json .xml .htm .html .ini .toml .yaml .yml`
     (decode utf-8 → cp1252 → replace);
   - images `.jpg .jpeg .png .tif .tiff .bmp` → `ocr_candidate` (tier 1 = filename only, OCR is
     phase 2);
   - legacy `.doc .xls .ppt` → `skipped_legacy` (LibreOffice revisit is an owner decision);
   - anything else → `skipped_ext` (keep per-extension counts in `status` output).
9. **Phase-2 OCR is explicit and owner-triggered:** `onesearch index --ocr` processes ≤ 200 files
   per run, resumes across runs. `no_text` PDFs: PyMuPDF render → tesseract `--psm 6 -l eng`
   (proven in Archivist). Images: pillow → tesseract `psm 3`. OCR text lands in the same chunks
   table (unit prefix `ocr:`), query API unchanged. HEIC → `skipped_ext` for now.
10. **Fresh venv** (`.venv`, python3.12) with `pymupdf numpy python-docx openpyxl python-pptx
    pillow`; tesseract binary at `/usr/bin/tesseract` (installed). **Do not reuse Archivist's venv.**
11. **Ops follow the Archivist pattern** (see `~/.config/systemd/user/archivist*.{service,timer}`):
    systemd-user oneshot `onesearch.service` + `onesearch.timer` (every 6 h, owner adjusts),
    log → `var/onesearch.log`, single writer via `var/index.lock`, queries may run concurrently
    (WAL).
12. **Repo:** public GitHub, SSH-key-only auth, zero document data in the repo (`var/`, `.venv/`
    gitignored). Commit identity `SargentNg <24596997+SargentNg@users.noreply.github.com>`
    (already global git config). **`gh` CLI is not authenticated on this box** — owner creates the
    empty public repo in the web UI; `git push` over SSH works.

## 3. Layout

```
~/opencode/OneSearch/
  DESIGN.md  README.md  .gitignore
  .venv/               (gitignored)
  var/                 (gitignored: index.db, vectors.npy, onesearch.log, index.lock)
  onesearch/
    __init__.py    # __version__
    cli.py         # argparse: index | query | status | gc
    config.py      # OLLAMA_URL, REMOTE, ROOTS=[("onedrive","")], caps, chunk params; env overrides
    store.py       # schema, kv, status transitions, checkpoint/resume, gc
    list_od.py     # rclone lsf -R --json -> (relpath,size,mtime)
    extract.py     # ext dispatch -> [(unit,text),...]; decides status
    chunk.py       # text -> chunks (900/120)
    embed.py       # batches of 64 -> nomic-embed-text; appends vectors.npy + vecrows
    query.py       # embed q -> mmap KNN + FTS5 hybrid -> top files w/ snippets
  ops/README.md    # units, log locations, manual commands
```

## 4. CLI

```
onesearch index [--ocr] [--dry-run] [--force-file RELPATH] [--force-embed]
onesearch query "text" [-k 10] [--vectors|--keywords] [--under PREFIX]
onesearch status        # counts by status & extension, npy size, last run, error samples
onesearch gc            # purge deleted + orphan rows; rewrite npy dense when >25% gaps
```

Query output: `score  relpath  (top-chunk snippet, ~80 chars)` plus the copyable
`onedrive:<relpath>` reference for each hit.

## 5. Incremental lifecycle (`onesearch index`)

1. `rclone lsf -R --json onedrive:` → live set. On the *first* run also run
   `rclone size "onedrive:"` + file count and log the totals (fills §8; both take minutes —
   run them detached and read the file when done).
2. Diff vs `files` table: new / changed (size or mtime differs) / vanished → status `deleted`.
3. Work queue = everything not `indexed` (incl. previously `queued`, `error` with <3 strikes,
   and `deleted` reappearing). Per file: extract → chunk (replace old chunks/fts/vecrows for
   that file) → embed (batch 64) → status `indexed`. 3 failed attempts → `error` (retries next
   run; log the reason to the `files` row via a `note`-less convention: last line of the log).
4. When a full pass completes clean, `gc` runs automatically at the end.
5. The timer just re-runs `index`; it's incremental by design. `var/index.lock` (flock) prevents
   concurrent index runs; a running query under WAL is fine.

## 6. Privacy & safety

- Ollama on 127.0.0.1 only. rclone→OneDrive is the only outbound traffic (pre-existing pattern).
- Writable locations: `OneSearch/var/`, the GitHub remote, `~/.config/systemd/user/`.
  OneDrive is read-only. If in doubt during a build, stop and ask the owner.
- The Archivist system at `~/opencode/Archivist` is a read-only reference (tesseract/PSM6
  recipes, rclone quirks, unit style). Never modify it.

## 7. Storage schema (SQLite, `PRAGMA journal_mode=WAL`)

```sql
CREATE TABLE kv(k TEXT PRIMARY KEY, v TEXT);
-- keys: schema_version, model, dims, embed_calls, last_index_at, run_counter

CREATE TABLE files(id INTEGER PRIMARY KEY, relpath TEXT UNIQUE NOT NULL,
                   size INTEGER, mtime REAL, status TEXT NOT NULL);
-- status: queued|indexed|no_text|ocr_candidate|skipped_size|skipped_legacy|skipped_ext|error|deleted

CREATE TABLE chunks(id INTEGER PRIMARY KEY,
                    file_id INTEGER NOT NULL REFERENCES files(id) ON DELETE CASCADE,
                    seq INTEGER NOT NULL, unit TEXT,
                    char_start INTEGER, char_end INTEGER, text TEXT NOT NULL);

CREATE TABLE vecrows(chunk_id INTEGER PRIMARY KEY REFERENCES chunks(id) ON DELETE CASCADE,
                     row INTEGER UNIQUE);

CREATE VIRTUAL TABLE chunks_fts USING fts5(fts_text, tokenize="porter unicode61");
-- plain (non-external-content) FTS table; rowid = chunks.id;
-- fts_text = chunk.text || ' ' || relpath  (so filenames keyword-match too).
-- Insert/delete in the same transaction as the chunk row.
```

`var/vectors.npy`: float32 `[N, 768]`; **row i = the chunk whose `vecrows.row` is i**.
Query loads it with `np.load(..., mmap_mode='r')`. N grows on append; `gc` rewrites it dense and
remaps `vecrows.row` when `(N - in_use)/N > 0.25`. The .npy is derived data — if lost, rerun
`onesearch index --force-embed` (re-embeds all `indexed` files; hours, acceptable disaster path).

## 8. Budget (fill at build, step 0)

Known floor: the `Family Vault/Scans` subtree alone is ≥ 8.9 GiB / 1,720 files. Whole drive:
re-run `rclone size "onedrive:"` + `rclone lsf -R --files-only "onedrive:" | wc -l` (minutes;
detach). From those numbers estimate: indexable files, chunks ≈ files × 1–2 avg, embed rate with
nomic-embed-text on this box (expect ~50–150 chunks/s; the box has 4× RTX 3090 + RTX 3080, but
embeddings are small — don't wire in GPU management), npy size = chunks × 3 KB. If projected
npy > ~1.5 GB, drop chunk size before the first full run and note the decision in the commit.

## 9. Acceptance (done when all pass)

1. `onesearch status`: every unit-file is `indexed`/`no_text`/`ocr_candidate`/`skipped_*`;
   `error` count ≈ 0 (list any remainders with reasons).
2. Query tests against **real files** — pick 3 under `onedrive:Family Vault/Scans` (flat folder,
   the 31 recent Archivist docs all have embedded OCR text layers):
   - exact phrase from one doc → ranks #1;
   - a paraphrase of that doc's content → top 5;
   - part of a filename → #1.
   Record the actual test queries + results in `QUERY_TESTS.md` and commit it.
3. Re-run `onesearch index` → `0 new / 0 changed / 0 deleted` (idempotent).
4. Query latency < 5 s after first (warm mmap).
5. `systemctl --user status onesearch.timer` → active; one full cycle in the log.
6. Repo pushed; document content verifiably absent from the repo.
7. Nothing written outside the allowed locations.

## 10. Environment facts (verified 2026-11-26; re-verify cheap ones at build)

- Ollama: `/usr/local/bin/ollama`, 127.0.0.1:11434, running. Models include
  `nomic-embed-text:latest` (endpoint tested: 768-dim OK) plus `bge-m3`,
  `nomic-embed-text-v2-moe`, `glm-ocr`, `deepseek-ocr:3b` (future options — do not use by default).
- rclone remote `onedrive:` (OAuth) works. Quirks of this rclone build: no `size --json`;
  no `lsf --fs-only` (use `-R --files-only` or `-R --json`); no `--allow-overwrite` flag;
  `move` without a trailing-slash dest creates a dir named after the file.
- `/usr/bin/tesseract`, `/usr/bin/pdftotext`, python3.12 present. `pymupdf`+`numpy` exist in
  Archivist's venv — mirror those deps into a fresh venv instead of sharing it.
- GPUs: 4× RTX 3090 24 GB + 1× RTX 3080 10 GB; ~378 GB free on `/` (nvme). Embeddings don't need GPU mgmt.
- systemd-user pattern: model the new units on `~/.config/systemd/user/archivist*.{service,timer}`.
- Git: global identity already `SargentNg <24596997+SargentNg@users.noreply.github.com>`.

## 11. Build order

0. Inventory (size + file count) → commit `INVENTORY.md`.
1. venv + deps, `store.py` schema, `cli.py` skeleton (`status` shows empty counts).
2. `list_od.py`, `extract.py` (PDF first, then office/eml/plaintext), `chunk.py` + unit tests
   for chunking edge cases.
3. `embed.py`; smoke-test with `--force-file` on one known-good `Family Vault/Scans` doc.
4. Full `onesearch index` (detach with nohup; compare against §8 budget).
5. `query.py` + acceptance queries (§9.2).
6. systemd units + first timer cycle.
7. Phase-2 OCR with `--ocr` — only after the owner approves a 10-file trial batch.
8. `ops/README.md`, `QUERY_TESTS.md`, push.