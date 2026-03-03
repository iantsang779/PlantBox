# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Activate the project virtual environment (required — do not use system or miniforge Python)
source ~/python_venv/bin/activate

# Install dependencies
pip install -r requirements.txt

# Start the server (single worker required — cache is in-memory and not process-safe)
uvicorn main:app --host 0.0.0.0 --port 8000 --workers 1

# Start with auto-reload during development
uvicorn main:app --reload
```

## Git workflow

After completing any change to the codebase, stage the relevant files and **ask the user for permission before committing**. Once approved, commit with a concise message in this format:

```
<type>: <short imperative summary>

[optional body explaining why, not what]
```

Common types: `feat`, `fix`, `refactor`, `chore`, `docs`.

Then ask whether to push to `origin main`.

## Architecture

This is a single-page FastAPI dashboard that queries the **EMBL-EBI public MySQL server** (`mysql-eg-publicsql.ebi.ac.uk:4157`, user `ensro`, no password) for wheat genomic variant and KASP marker data.

### Two-database design

At startup, the app discovers and connects to two separate databases on the EBI server:

| Pool | Database pattern | Used by |
|---|---|---|
| `variation_pool` | `triticum_aestivum_variation_*` | Variant/transcript queries |
| `core_pool` | `triticum_aestivum_core_*` | KASP marker queries |

Both pools are stored on `app.state` and passed explicitly into `fetch_and_join()`. Version selection picks the highest numeric suffix (e.g. `110_1 > 99_1`).

### Request flow

1. `POST /api/query` → validates inputs → `fetch_and_join(variation_pool, core_pool, ...)` in `database.py`
2. Inside `fetch_and_join`: variant query and marker query run **concurrently** via `asyncio.gather` when a `variant_id` is provided; marker query is skipped when only `transcript_id` is given
3. Results are pandas left-joined on `variant_name`, serialised to `list[dict]`, and stored in an `OrderedDict` cache under a UUID token
4. Download endpoints (`/api/download/csv/{token}`, `/api/download/excel/{token}`) do O(1) token lookup — no DB re-query

### In-memory cache

`_result_cache` in `main.py` — TTL 30 min, max 100 entries, FIFO eviction. **Not shared across processes**, hence `--workers 1` is mandatory.

### Frontend

`static/index.html` is a self-contained SPA loaded via `GET /`. It uses Tailwind CSS and Alpine.js from CDN (no build step). All state lives in the `dashboardApp()` Alpine component. Client-side sorting is computed in a `get sortedRows()` getter.

### Input validation

All user inputs pass through `validate_input()` in `database.py` before query construction: max 100 chars, regex `^[\w.\-]+$`. SQL parameters are always passed via aiomysql parameterisation (never interpolated).
