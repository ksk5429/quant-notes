---
title: Architecture Decision Records
description: The 10 load-bearing decisions behind K-Fish
---

# Architecture Decision Records

All ADRs live in the private repo at
[`docs/decisions/`](https://github.com/ksk5429/kfish/tree/main/docs/decisions).
The summaries below keep the public discussion surface readable without
exposing trading alpha.

| # | Title | Gist | Status |
|---|---|---|---|
| [0001](#adr-0001) | Hybrid repo structure | private monorepo + public analyzer, no submodules | accepted |
| [0002](#adr-0002) | DuckDB calibration spine | ASOF JOIN + Parquet interop; 7-night Brier-parity gate | accepted |
| [0003](#adr-0003) | aiogram 3 over python-telegram-bot | async/FSM/router first-class | accepted |
| [0004](#adr-0004) | Per-user agent wallets | user signs `ApproveAgent`; bot key can't withdraw | accepted |
| [0005](#adr-0005) | Claude primary + OpenAI fallback | red-team persona runs on opposite provider | accepted |
| [0006](#adr-0006) | uv over Poetry / pip-tools | single lockfile, workspace-native | accepted |
| [0007](#adr-0007) | Self-hosted Langfuse | trading data stays in-house | accepted |
| [0008](#adr-0008) | Isotonic + Venn-Abers calibration | distribution-free intervals for Kelly shrinkage | accepted |
| [0009](#adr-0009) | Oracle A1 + Fly.io NRT | always-on + Seoul-edge latency | accepted |
| [0010](#adr-0010) | Trunk-based + signed commits | solo-dev workflow with SSH-signed + `ci-pass` gate | accepted |

## ADR-0001 — Hybrid repo structure { #adr-0001 }

**Decision.** Private `ksk5429/kfish` uv-workspace monorepo containing
`kfish-common`, `kfish-core`, and `hypekr-bot`; standalone public
`ksk5429/polymarket-oracle-risk` for the PyPI package. No submodules.

**Why.** GitHub visibility is all-or-nothing per repo. Pure monorepo leaks
alpha; pure polyrepo adds Dependabot toll × 3. Hybrid aligns the boundary of
git visibility with the actual public/private boundary of the work.

**Gotcha.** Small code duplication between the private Polymarket client and
the public one is accepted — see the public repo's README for the drift
mitigation.

## ADR-0002 — DuckDB as the calibration spine { #adr-0002 }

**Decision.** All observations land in `data/warehouse/kfish.duckdb`.
`ASOF JOIN` is the key feature: every forecast is joined to the market price
at the instant it was made.

**Cutover gate.** Legacy `src/` stays authoritative until **7 consecutive
nightly runs** show Brier within 0.002 of the legacy engine. First night was
[green on 2026-04-19](../progress/brier-parity.md).

## ADR-0003 — aiogram 3 { #adr-0003 }

**Decision.** `aiogram >= 3.27` for the Telegram bot. Router + FSM + HTTPX
first-class; cleaner interop with asyncio than python-telegram-bot's v20
rewrite.

## ADR-0004 — Per-user agent wallets { #adr-0004 }

**Decision.** Each Telegram user gets a **fresh agent wallet** (not
main wallet). User signs one `ApproveAgent` message in their main wallet via
WalletConnect in the Mini App; the bot's key for that user can place orders
but cannot withdraw.

**Encryption model.** Two-layer envelope: a per-user HKDF-derived Fernet key
inside a process-wide master Fernet key. Rotating the master re-wraps every
wallet without regenerating any of them. Plaintext private keys never touch
disk and are zeroed on every scope exit.

**Revoke path.** 15-minute target from detection to full cohort rotation:
`approveAgent(0x00…, revoke)` → mint new → user re-approves in Mini App.

## ADR-0005 — Claude primary + OpenAI independent evaluator { #adr-0005 }

**Decision.** `ClaudeClient` is the default for all personas except
`red_team`, which routes to `OpenAIClient` to break shared-training blind
spots. Both adapters emit Langfuse spans with `prompt_version` metadata.

## ADR-0006 — uv over Poetry { #adr-0006 }

**Decision.** `uv >= 0.11` with workspace members, `.python-version`, and
`hatch-vcs` for `devN` versioning on every commit.

## ADR-0007 — Self-hosted Langfuse { #adr-0007 }

**Decision.** Docker-compose stack on the always-on box (Oracle A1 or Hetzner
CX32). `LANGFUSE_HOST` defaults to `localhost:3000`. Trading data never
leaves our tenancy.

## ADR-0008 — Isotonic + Venn-Abers calibration { #adr-0008 }

**Decision.** Post-hoc calibration stack per market category:

1. Isotonic (low-bias, high-variance; needs n ≥ 150)
2. Venn-Abers (distribution-free intervals)
3. Mondrian conformal at α=0.10 — singleton-required-to-trade gate
4. Shrinkage ensemble `α · p_llm + (1 - α) · p_market` with empirical-Bayes α

Refit every 50 trades per category; atomic pickle swap.

## ADR-0009 — Hybrid cloud { #adr-0009 }

**Decision.** Always-on (Langfuse, Prefect, scraper, nightly runner) on
Oracle Cloud Ampere A1 free tier; bot on Fly.io NRT (Tokyo) for ~30 ms edge
latency to Seoul. Tailscale between them.

## ADR-0010 — Trunk-based + signed commits { #adr-0010 }

**Decision.** Trunk-based development; no Gitflow. Branch protection via a
modern **Ruleset** (not classic protection) with:

- required signatures (SSH, 1Password-backed)
- required linear history
- required status check `ci-pass`
- pull-request flow with 0 required reviewers (solo-dev)
- squash merge only, auto-delete branch

Ruleset JSON lives at
`.github/ruleset-protect-main.json`; applied to both `kfish` and
`polymarket-oracle-risk` on 2026-04-19.
