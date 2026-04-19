---
title: Build log
description: What shipped when
---

# Build log

Reverse-chronological entries. Each entry links the commit + a short why.

## 2026-04-19 — Live-API validation + Brier-parity + quant-notes

- Probed live Polymarket Gamma + UMA Goldsky OOv2 + Managed OOv2 — all three
  reachable; Goldsky project id rotated (`clcxirpj…` → `clus2fnd…`), updated
  in `config.py` and `.env.example`.
- Rewrote `UMAGoldskyClient` to use the current `OptimisticPriceRequest`
  entity with per-stage timestamps. Added `bond_usdc` property that handles
  the 6-decimal USDC fixed-point.
- Added `tests/test_live_apis.py` under `-m network` — runs on demand against
  the real endpoints. All 4 live tests green.
- Wrote `scripts/brier_parity.py` — validates the new `kfish-core`
  calibration math against legacy retrodiction JSON. Brier parity **exact to
  4 decimal places** on all three archived legacy runs (N=100, 200, 30).
  First of the 7-night ADR-0002 cutover gate is green.
- Wrote `scripts/hip4_watch.py` — polls HL mainnet + testnet `/info` meta for
  HIP-4 symbols; exit code 2 fires when mainnet goes live.
- Wrote `scripts/bootstrap_live.py` — one-command launcher for
  repos + subtree split + branch protection + PyPI environment. All three
  irreversible steps gated on `--confirm`.
- Created `ksk5429/kfish` (private) + `ksk5429/polymarket-oracle-risk`
  (public, MIT). Subtree-split the oracle package into its own history +
  pushed. Applied `protect-main` ruleset with required signatures, linear
  history, and `ci-pass` status-check gate on both repos.
- Created GitHub Environment `pypi` on the public repo. Local
  `uv build && twine check --strict` both pass. Runbook
  `runbooks/pypi-first-release.md` describes the one remaining browser-only
  step (pending-publisher registration on pypi.org).
- Wrote `scripts/langfuse_smoke.py` — emits one `generation` span to the
  configured Langfuse host; returns exit 0 only when the trace flushes.
- Built this site (`quant-notes`) and pushed to `ksk5429/quant-notes`.

## 2026-04-19 — Production build

- Full workspace implementation per blueprint. 123 tests green, ruff clean,
  uv.lock committed. See the [feat commit e804d6e][prod-commit].
- Wire-level: `kfish-common` DuckDB + ASOF helpers + Claude adapter with
  prompt caching + OpenAI independent evaluator.
- 9-persona swarm + Delphi multi-round + asymmetric extremization.
- Isotonic + Venn-Abers + Mondrian conformal + empirical-Bayes shrinkage
  ensemble. Atomic `CalibratorBundle` pickle for hot-swap.
- Polymarket Gamma + CLOB + UMA Goldsky + Kiwi tokenizer + SimHash dedup.
- hypekr-bot: Fernet-encrypted two-layer wallet envelope, aiogram 3 with
  Korean aliases, FastAPI Mini App with HMAC-signed nonces + full ECDSA
  signature verification on `ApproveAgent`, pure-functional order
  construction with builder-fee.
- polymarket-oracle-risk: NumPyro NUTS posterior fit, out-of-time backtest,
  Streamlit dashboard, CLI with `score` / `subjectivity` / `demo` subcommands.

## 2026-04-19 — Scaffold

- Initial hybrid-monorepo scaffold (pre-build): ADRs, runbooks, CI workflows,
  migrations, dbt project, Langfuse docker-compose, systemd units.

[prod-commit]: https://github.com/ksk5429/kfish/commit/e804d6e
