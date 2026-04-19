---
title: Documentation
description: Comprehensive documentation of the K-Fish prediction-market forecasting stack
---

# K-Fish documentation

This section is the complete reference for the K-Fish stack — the theory, the
architecture, the data model, the operations, and the validation plan. It
complements the reverse-chronological [build log](../progress/build-log.md)
and the [architectural decision records](../architecture/adrs.md).

!!! note "Scope"
    K-Fish is compounding infrastructure for calibrated prediction-market
    forecasting. The current working baseline is Brier **0.206** on a
    200-market retrodiction (does not yet beat crowd at 0.084). These
    documents describe the machinery that will be iterated against that
    baseline, not an already-solved system.

## Reading path

| If you want to… | Start here |
|---|---|
| Run the code for the first time | [Getting Started](getting-started.md) |
| Understand *why* Brier / extremization / calibration | [Theory and Math](theory.md) |
| Understand *how* forecasts are produced | [Swarm Architecture](swarm.md) |
| Trust the probabilities | [Calibration Stack](calibration.md) |
| Understand the news pipeline | [News Pipeline](news.md) |
| Query the warehouse | [Data Model](data-model.md) |
| Operate the stack | [Operations](operations.md) |
| Audit forecast quality | [Validation](validation.md) |
| Read/modify prompts | [Prompt Engineering](prompt-engineering.md) |
| Look up a term | [Glossary](glossary.md) |
| Find the papers | [References](references.md) |

## Three primary products

1. **DuckDB + Langfuse calibration spine** — every forecast, price, news
   item, and oracle event joins on the same UTC timeline. Core package:
   `packages/kfish-common/`. App: `apps/kfish-core/`.
2. **polymarket-oracle-risk** — MIT-licensed, public, Bayesian risk scorer
   for UMA Optimistic Oracle resolutions. Target: PyPI `v0.1.0`.
3. **hypekr-bot** — private Korean Telegram bot on Hyperliquid with
   per-user agent wallets; builder-fee revenue; HIP-4 handlers feature-gated.

The three are coupled only through the DuckDB warehouse and the shared
`kfish-common` utilities; ADR-0001 and ADR-0008 specify the boundary.

## Writing style

- Code references use `[file.py:symbol](link)` pointing into the private
  monorepo where applicable, or into this docs site's source.
- Every algorithmic decision cites a paper or a documented OSS
  implementation (Rule R6 in `CLAUDE.md`).
- Mathematical notation uses MathJax: display equations as `$$ … $$`,
  inline as `$…$`.
- Admonitions flag invariants (`!!! warning`), references (`!!! note`),
  and operator-facing caveats (`!!! tip`).
