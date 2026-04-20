---
title: K-Fish Quant Notes
description: Development notes and progress log for the K-Fish prediction-market engine
---

# K-Fish Quant Notes

> Companion site for **K-Fish** — a swarm-intelligence prediction engine for
> prediction markets. This site tracks architecture, progress, reviews, and
> collaborator-facing discussion. Production code lives in the private
> [ksk5429/kfish][kfish] monorepo; the public risk analyzer lives at
> [ksk5429/polymarket-oracle-risk][oracle].

!!! abstract "Quick facts"
    - **Baseline**: Brier **0.1799** on N=100, **0.2026** on N=200 resolved Polymarket markets (legacy engine)
    - **Parity**: new kfish-core calibration matches legacy Brier **exact to 4 decimal places** across all three archived runs
    - **Public endpoints live**: Polymarket Gamma + UMA OOv2 + Managed OOv2 (validated April 2026)
    - **Stack**: 9-persona Delphi swarm · isotonic + Venn-Abers + Mondrian conformal · DuckDB ASOF spine · Claude + OpenAI routing
    - **138 tests** pass in the active workspace; `ruff` and `pyright` clean

## Why two sites

The research-notes site at <https://ksk5429.github.io/research-notes> is for
the academic / offshore-wind PhD work. K-Fish is a separate agenda —
high-frequency prediction-market forecasting with a distinct team dynamic, a
public package to distribute, and a different cadence of reviews. Mixing the
two diluted both. This site exists so collaborators can comment, review, and
discuss without having to page through PhD content.

## Canonical documents

Both primary planning documents are preserved verbatim and permalinkable:

- [**90-Day Build Guide**](blueprints/90-day-build-guide.md) — strategic plan
  for the 13-week sprint, covering the calibration spine, oracle analyzer,
  Korean Telegram bot, and Korean news pipeline.
- [**Three-Repo GitHub Blueprint**](blueprints/three-repo-blueprint.md) — the
  structural plan: private uv-workspace monorepo plus public MIT-licensed
  analyzer, with the full CI/CD / signing / release-please / SLSA-2 setup.

## Where things are

=== "Architecture"

    - [Architecture overview](architecture/overview.md)
    - [ADRs](architecture/adrs.md) — 10 decisions including DuckDB spine,
      agent-wallet custody model, Claude+OpenAI split, trunk-based signed
      commits.
    - [Swarm + Calibration](architecture/swarm-calibration.md)
    - [Oracle Risk](architecture/oracle-risk.md)
    - [Telegram Bot](architecture/hypekr-bot.md)

=== "Progress"

    - [Build log](progress/build-log.md) — what shipped when
    - [Brier parity](progress/brier-parity.md) — ADR-0002 cutover evidence
    - [Live-API validation](progress/live-validation.md) — Polymarket Gamma,
      UMA OOv2, Managed OOv2 smoke results
    - [HIP-4 watch](progress/hip4-watch.md) — mainnet launch monitor

=== "Review"

    - [Self-review template](reviews/template.md)
    - [Open questions](reviews/open-questions.md) — currently-ambiguous design
      choices to discuss with collaborators.

=== "Links"

    - [Related repos](links.md)

## Collaborating

**Start here:** [how to collaborate →](collaborate.md) — the three-lane
model (Discussion / Issue / PR), good-first-topics, what's in scope, and
how reviewers get credit.

Quick links:

- 💬 [Open a Discussion](https://github.com/ksk5429/quant-notes/discussions)
  (questions, ideas, reviews)
- 🐛 [File an issue](https://github.com/ksk5429/quant-notes/issues/new/choose)
  (correction / review-request / site-bug)
- ✏️ Edit this page via the pencil icon at the top-right

!!! warning "Korean legal status"
    The Korean Telegram bot is blocked from live trading pending counsel
    review. See the [legal summary](legal/summary.md) for what's pending
    and which firms to talk to.

For private trading alpha, the DMs or the private
[kfish](https://github.com/ksk5429/kfish) repo's issue tracker are the
places — never post prompts or position-sizing logic here.

[kfish]: https://github.com/ksk5429/kfish
[oracle]: https://github.com/ksk5429/polymarket-oracle-risk
