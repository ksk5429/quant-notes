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
canonical feed. **Six RSS feeds green at 2026-04-20; Naver API row is
gated on developer-app credentials (pending registration).**

| Source | URL | Status |
|---|---|---|
| yonhap | `yna.co.kr/rss/news.xml` | ✅ 200 |
| yonhap_econ | `yna.co.kr/rss/economy.xml` | ✅ 200 |
| hankyoreh | `hani.co.kr/rss` (no trailing slash — the old `/rss/` 308-redirects) | ✅ 200 |
| blockmedia | `blockmedia.co.kr/feed` | ✅ 200 |
| dailynk | `dailynk.com/feed/` | ✅ 200 |
| donga | `rss.donga.com/total.xml` | ✅ 200 |
| naver (API, gated on credentials) | `naveropenapi…` | ⏸ pending registration |

See [News Scraper status](../progress/news-scraper.md) for the full live run.

## Unit-test inventory

As of 2026-04-20, **138 unit tests** collected across the active
workspace (confirmed via `pytest --collect-only`):

| Package | Count |
|---|---|
| `apps/kfish-core/tests/` | 71 |
| `apps/hypekr-bot/tests/` | 36 |
| `packages/kfish-common/tests/` | 27 |
| `tests/` (repo root, e.g. `test_brier_cutover.py`) | 4 |
| **total** | **138** |

Run with:

```bash
uv run pytest apps/ packages/ tests/test_brier_cutover.py \
  --ignore=apps/kfish-core/tests/test_news_live.py \
  --ignore=apps/kfish-core/tests/test_live_apis.py -q
```

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
