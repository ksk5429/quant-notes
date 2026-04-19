---
title: Operations
description: Nightly pipeline, monitoring, backups, runbook index
---

# Operations

How K-Fish runs in production. Private infrastructure only; the public
MIT package `polymarket-oracle-risk` has no operational footprint.

## Scheduling

Primary: **GitHub Actions self-hosted runner** on an Oracle Ampere A1 (ARM64).
Workflow: `.github/workflows/nightly.yml`. Cron: `17 18 * * *` (03:17 JST
/ 18:17 UTC the prior day).

Backup: **systemd timer** at `infra/systemd/kfish-nightly.{service,timer}`.
Off by default; enable only when the self-hosted runner is offline by
touching `/opt/kfish/.enable-systemd-nightly`. Both schedulers writing
concurrently would contend on DuckDB's single-writer lock.

!!! warning "One writer at a time"
    DuckDB 1.x permits a single read-write process per database file.
    Do not enable both schedulers. The systemd unit is explicitly gated
    on a sentinel file to prevent accidents.

## What nightly does

`python -m kfish_core.pipeline.nightly` in order:

1. **Apply migrations.** `run_migrations()` from `kfish-common`. Idempotent.
2. **Ingest markets from Polymarket Gamma.** Writes `kfish.markets` and
   `kfish.prices`. Failure-tolerant: an outage degrades this step only.
3. **Ingest UMA oracle events from Goldsky.** Writes `kfish.oracle_events`.
4. **Ingest Korean news.** RSS always; Naver tier when credentials present;
   optional Papago title translation (default on, ~₩60/run).
5. **Checkpoint DuckDB.**
6. **Brier decomposition** over `kfish.calibration_rows`. Logged for
   trend tracking.
7. **Brier math-parity gate.** `scripts/brier_cutover_night.py` runs the
   ADR-0002 gate and appends a row to `data/brier_cutover_log.jsonl`.
8. **Export to Parquet.** `out/parquet/` for B2 cold + R2 warm sync.
9. **Healthchecks.io ping.** Success or `/fail`.

All steps are independently failure-tolerant. Unreachable upstream or
missing credential degrades that step only; the run moves forward. One
structured log line summarizes the result for downstream parsing.

## Backups

Two tiers, written on every successful nightly:

- **Backblaze B2 cold.** `s3://kfish-backups/warehouse/run=YYYY-MM-DD/`.
  Every run, retained indefinitely (cheap).
- **Cloudflare R2 warm.** `s3://kfish-backups/warehouse/latest/`. Most
  recent run only; egress-free for HL/Polymarket APIs.

`uv run python -m kfish_core.pipeline.export_parquet` produces the files.
Restore path is documented in [`runbooks/restore.md`](https://github.com/ksk5429/kfish/blob/main/runbooks/restore.md).

## Monitoring

| Signal | Channel | Alert trigger |
|---|---|---|
| Nightly completion | healthchecks.io | 24h silence → email |
| DuckDB size growth | Prometheus node-exporter + Grafana | >20% week-over-week |
| LLM cost spike | Langfuse dashboard | >$5 in a 24h window |
| HL positions | Custom Streamlit dashboard | P&L deviation >3σ |
| Cutover streak | `data/brier_cutover_log.jsonl` | streak resets to 0 |

Langfuse self-hosted is deployed via `infra/langfuse/docker-compose.yml`
on the Oracle A1 (ADR-0007 keeps trading traces in-house).

## Secrets

Managed via `.env` files loaded by `python-dotenv` and `pydantic-settings`:

- Workspace: `.env` at repo root (git-ignored).
- Bot: `.env` in `apps/hypekr-bot/` (encrypted Fernet envelope, per ADR-0004).
- CI: GitHub Actions encrypted secrets, injected at workflow level.

Rotation: see [`runbooks/rotate-keys.md`](https://github.com/ksk5429/kfish/blob/main/runbooks/rotate-keys.md).
Every key ships with a documented revocation path.

## Runbook index

Located in `runbooks/` (private repo):

| Runbook | Purpose |
|---|---|
| `deploy.md` | Full Oracle A1 bootstrap |
| `restore.md` | DuckDB restore from B2/R2 |
| `rotate-keys.md` | Revocation + regeneration for every secret |
| `incident.md` | Five-step incident triage template |
| `dr-log.md` | Post-incident disaster-recovery journal |
| `scraper-ops.md` | Korean news scraper operator guide (see [News Pipeline](news.md)) |
| `pypi-first-release.md` | polymarket-oracle-risk v0.1.0 release steps |
| `kr-legal-brief.md` | Private Korean legal due diligence (VAUPA, KoFIU, 특금법) |

## Cost profile

| Component | Monthly estimate |
|---|---|
| Oracle A1 (4 vCPU, 24 GB, always-free) | $0 |
| Fly.io NRT edge (scan worker) | ~$5 |
| Backblaze B2 cold storage | ~$0.50 per 100 GB |
| Cloudflare R2 warm | $0 (below free tier) |
| LLM (Claude primary + OpenAI evaluator) | $30–$80 depending on market cadence |
| Langfuse self-hosted | $0 (runs on Oracle A1) |
| Naver Developers + Papago | ₩0–₩30k (within free tier for <1000 calls/day) |

## Related

- ADRs: [ADR-0007](../architecture/adrs.md) Langfuse self-hosted,
  [ADR-0009](../architecture/adrs.md) Hybrid cloud.
- Code: `apps/kfish-core/src/kfish_core/pipeline/nightly.py`
- Schedulers: `.github/workflows/nightly.yml`, `infra/systemd/kfish-nightly.{service,timer}`
