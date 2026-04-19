---
title: Validation
description: Retrodiction baseline, math-parity cutover, live-API tests
---

# Validation

How we prove the stack works — or admit it doesn't yet.

## Baseline: retrodiction on resolved markets

`apps/kfish-core/src/kfish_core/pipeline/retrodict.py` loads every
`(market, outcome)` pair from `kfish.markets` ⋈ `kfish.outcomes`, runs
the full swarm on each, and records:

| Metric | Current baseline (N=200) |
|---|---|
| Brier | **0.206** |
| Accuracy | 69.0 % (138/200) |
| ECE (10 bins) | ≈0.043 |
| BSS vs random (0.25) | +17.6 % |
| BSS vs crowd (0.084) | **−145 %** (does not yet beat crowd) |

Brier-best personas: `contrarian` (0.199), `inside_view` (0.206),
`premortem` (0.207). The baseline stays authoritative through Phase 1;
the new workspace is additive.

!!! note "Retrodiction is news-free on purpose"
    Injecting today's `kfish.articles` into a market that resolved
    yesterday would be future-leak and poison the Brier comparison.
    `scan.py` uses news context; `retrodict.py` explicitly does not.
    Point-in-time news retrieval (`published_at < resolved_at`) is a
    planned feature.

## Math-parity cutover gate (ADR-0002)

**What the gate checks.** That the numpy Brier decomposition in
`kfish-core` matches the legacy engine's stored Brier on the same
archived `retro_v2_*.json` retrodiction runs, within 0.002. Seven
consecutive UTC-day passes clear the gate.

**What the gate does NOT check.** Forecast equivalence on *fresh*
markets. Both engines ingest the same archived JSON and recompute Brier
off the stored probabilities; parity just means the numpy implementation
reproduces the legacy implementation. A future live-fork gate (pending
ADR-0011) will run both engines on the same newly-resolved markets and
compare Brier on resolution — that is the real cutover gate.

The gate lives at `scripts/brier_cutover_night.py` and appends
single-line JSON records to `data/brier_cutover_log.jsonl`. Each record:

```json
{"date": "2026-04-19", "pass": true, "recorded_at": "2026-04-19T23:04:08+00:00"}
```

Streak counting walks the log from the tail; any `pass=false` resets to
zero. The script is idempotent per UTC day, atomic (`tmp` + `replace`),
and path-traversal-guarded on `--log`.

### Current streak

| Day | UTC date | Pass |
|---|---|---|
| 1 | 2026-04-19 | ✅ |

History is in [`data/brier_cutover_log.jsonl`](https://github.com/ksk5429/kfish/blob/main/data/brier_cutover_log.jsonl).

## Live-API tests

`apps/kfish-core/tests/test_live_apis.py` is marked `@pytest.mark.network`
and runs on demand against real endpoints:

- **Polymarket Gamma** — list active markets, min volume filter, pagination.
- **UMA Goldsky OOv2** — fetch optimistic price requests, per-stage
  timestamps, USDC bond decoding.
- **Managed OOv2** — same shape, different project ID (`clus2fnd…`).

Invocation:

```bash
uv run pytest -m network apps/kfish-core/tests/test_live_apis.py -q
```

All four live tests were green at the most recent probe
([live-validation](../progress/live-validation.md), 2026-04-19).

## Live RSS coverage

`apps/kfish-core/tests/test_news_live.py` (`-m network`) probes every
canonical feed. Seven green at 2026-04-20:

| Source | URL | Status |
|---|---|---|
| yonhap | `/RSS/economy.xml` | ✅ 200 |
| yonhap_econ | `/RSS/news.xml` | ✅ 200 |
| hankyoreh | `/rss` (no trailing slash — the old `/rss/` 308-redirects) | ✅ 200 |
| blockmedia | `/feed` | ✅ 200 |
| dailynk | `/feed` | ✅ 200 |
| donga | `/rss/…` | ✅ 200 |
| naver (gated on credentials) | api | ✅ 200 when keys present |

See [News Scraper status](../progress/news-scraper.md) for the full live run.

## Unit-test inventory

As of 2026-04-20, **138 unit tests** on the active workspace:

| Module | Count |
|---|---|
| `test_agents.py` | 8 |
| `test_brier.py` | 5 |
| `test_brier_sql.py` | 3 |
| `test_conformal.py` | 7 |
| `test_ensemble.py` | 6 |
| `test_ingest_articles.py` | 7 |
| `test_isotonic.py` | 5 |
| `test_markets.py` | 4 |
| `test_news.py` | 5 |
| `test_news_search.py` | 7 |
| `test_market_input_prompt.py` | 4 |
| `test_pipeline_ingest.py` | 5 |
| `test_brier_cutover.py` | 4 |
| `test_refit.py` | 3 |
| *(kfish-common)* | ~65 |

The single pre-existing failure in `tests/unit/test_calibration_v2.py`
is in the legacy `src/prediction/` engine, not the new workspace;
documented in the punch-list commit.

## Langfuse smoke

`scripts/langfuse_smoke.py` emits one `generation` span to the configured
Langfuse host and exits 0 only when the trace flushes end-to-end. Runs
in the weekly CI cron and gates deploys.

## What's not yet validated

- **Forecast parity on fresh markets** — requires the live-fork gate
  described above.
- **Paper-trading P&L** — disabled by rule R2 until Brier < 0.12 on a
  fresh (non-retrodicted) 50-market slate. Live capital requires explicit
  human approval.
- **Translation quality** — title translations are accepted as-is from
  Papago; no back-translation BLEU.

## Related

- Retrodiction code: `apps/kfish-core/src/kfish_core/pipeline/retrodict.py`
- Math-parity gate: `scripts/brier_cutover_night.py`, `scripts/brier_parity.py`
- ADRs: ADR-0002 (cutover), ADR-0005 (dual-model evaluator)
