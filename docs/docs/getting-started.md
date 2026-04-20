---
title: Getting Started
description: Install, migrate, and run a first retrodiction
---

# Getting started

The minimum-viable path from a cold clone to a working forecast.

## 1. Prerequisites

| Tool | Version | Purpose |
|---|---|---|
| Python | `>=3.12, <3.13` | Runtime |
| uv | `>=0.5` | Workspace manager |
| DuckDB | `>=1.0` | Warehouse (embedded, no server) |
| Git | any | VCS |

Optional for full features:

| Credential | Needed for |
|---|---|
| `ANTHROPIC_API_KEY` | LLM swarm (primary model) |
| `OPENAI_API_KEY` | Independent evaluator (ADR-0005) |
| `NAVER_CLIENT_ID` / `NAVER_CLIENT_SECRET` | Naver News API tier |
| `NCP_PAPAGO_CLIENT_ID` / `NCP_PAPAGO_CLIENT_SECRET` | Translation |
| `LANGFUSE_HOST` / keys | Self-hosted trace collector (ADR-0007) |

**None of these are required to boot.** RSS-only ingestion, migrations, and
retrodiction against existing DuckDB data run with zero credentials.

## 2. Install

```bash
git clone git@github.com:ksk5429/kfish.git
cd kfish
uv sync --all-packages
```

On Windows, shell environments that default to `cp949` can mangle the
console output from `uv sync`:

```bash
PYTHONUTF8=1 PYTHONIOENCODING=utf-8 uv sync --all-packages
```

## 3. Migrate the warehouse

```bash
uv run kfish-migrate
```

Flyway-style: files in `migrations/` named `V000N__*.sql` run in order and
are tracked in `kfish.schema_history`. Idempotent on re-run.

## 4. Run the legacy retrodiction baseline

```bash
uv run --package kfish-core kfish-retrodict --n 30 --samples 3 --rounds 1
```

Expected Brier on a healthy install against the archived market set:
**≈0.206** (matches the 200-market legacy baseline within noise). See
[Validation](validation.md) for the current cutover gate.

## 5. Run a live market scan

!!! warning "Live API calls"
    This makes real HTTP calls to Polymarket Gamma and consumes LLM
    tokens. Default config screens for volume ≥ $100k and routes ≤ 5
    news snippets per market.

```bash
uv run --package kfish-core kfish-scan --min-volume 100000 --limit 10
```

## 6. Run one news-ingestion pass

```bash
# Plan only — no HTTP calls, no DB writes, no credentials needed.
uv run --package kfish-core kfish-news --dry-run

# Warehouse stats — requires prior `kfish-migrate` and, ideally, at
# least one real ingestion run so the tables aren't empty.
uv run --package kfish-core kfish-news --stats

# Full pass including cheap title translation (Papago, ~₩60/run):
uv run --package kfish-core kfish-news
```

The scraper is documented in full at [News Pipeline](news.md) and operated
via [`runbooks/scraper-ops.md`](https://github.com/ksk5429/kfish/blob/main/runbooks/scraper-ops.md).

## 7. Verify everything

```bash
# Unit tests (138 as of 2026-04-20):
uv run pytest apps/ packages/

# Ruff + pyright:
uv run ruff check apps/ packages/
uv run pyright
```

## Troubleshooting

| Symptom | Fix |
|---|---|
| `DuckDBPyConnection` "different configuration" error | A writer is already open; close it with `close_duckdb()` or restart the Python process |
| `match_bm25 does not exist` | The FTS extension didn't load — `INSTALL fts; LOAD fts;` in a DuckDB shell |
| `ImportError: No module named 'kfish_common'` | You're outside a `uv run` context; prefix commands with `uv run` |
| Cp949 glyphs in output | Set `PYTHONUTF8=1 PYTHONIOENCODING=utf-8` before `uv` commands on Windows |

## Next steps

- [Theory and Math](theory.md) — *why* Brier + Murphy + extremization.
- [Swarm Architecture](swarm.md) — *how* the 9 personas produce a forecast.
- [Operations](operations.md) — nightly cron, backups, monitoring.
