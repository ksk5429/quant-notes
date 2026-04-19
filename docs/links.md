---
title: Related repos and upstreams
description: Where everything lives
---

# Related repos

## K-Fish repos

| Repo | Visibility | Purpose |
|---|---|---|
| [ksk5429/kfish](https://github.com/ksk5429/kfish) | private | Production uv-workspace monorepo — 9-agent swarm, calibration spine, Telegram bot |
| [ksk5429/polymarket-oracle-risk](https://github.com/ksk5429/polymarket-oracle-risk) | public, MIT | PyPI analyzer for UMA oracle manipulation-risk scoring |
| [ksk5429/quant-notes](https://github.com/ksk5429/quant-notes) | public | **This site** (MkDocs Material) |
| [ksk5429/quant](https://github.com/ksk5429/quant) | public | Earlier scratch repo (status pending — see [open questions](reviews/open-questions.md)) |
| [ksk5429/research-notes](https://github.com/ksk5429/research-notes) | public | Separate MkDocs site for offshore-wind PhD research |

## Upstreams + references

### Prediction markets

- [Polymarket Gamma API](https://docs.polymarket.com/developers/public-api)
- [Polymarket CLOB](https://docs.polymarket.com/quickstart/introduction/main)
- [py-clob-client](https://github.com/Polymarket/py-clob-client)

### UMA Optimistic Oracle

- [UMA protocol docs](https://docs.uma.xyz/)
- [UMA subgraphs repo](https://github.com/UMAprotocol/subgraphs) — canonical
  source for current Goldsky endpoint URLs
- [Goldsky public subgraphs](https://api.goldsky.com/) — API root

### Hyperliquid

- [Hyperliquid docs](https://hyperliquid.gitbook.io/hyperliquid-docs)
- [hyperliquid-python-sdk](https://github.com/hyperliquid-dex/hyperliquid-python-sdk)
- [HIP-4 proposal](https://github.com/hyperliquid-dex/rfcs/) (testnet as of
  Apr 2026)

### LLM forecasting literature

- Halawi et al. 2024 — [arXiv:2402.18563](https://arxiv.org/abs/2402.18563)
- Schoenegger et al. 2024 — Sci Adv 10 (9-persona median ensemble)
- Tian et al. 2023 — [arXiv:2305.14975](https://arxiv.org/abs/2305.14975)
  (verbalized probabilities)
- Turtel et al. 2025 — [arXiv:2505.17989](https://arxiv.org/abs/2505.17989)
- ForecastBench (Karger 2024/25) —
  [arXiv:2409.19839](https://arxiv.org/abs/2409.19839)
- Alur et al. 2025 AIA — [arXiv:2511.07678](https://arxiv.org/abs/2511.07678)

### Calibration

- Murphy 1973 — Brier decomposition
- Naeini 2015 — Expected Calibration Error
- Venn-Abers Predictors — [library](https://github.com/ivanpetej/venn-abers)
- Mondrian conformal — [crepes library](https://github.com/henrikbostrom/crepes)

### Tooling

- [uv](https://docs.astral.sh/uv/)
- [ruff](https://docs.astral.sh/ruff/)
- [MkDocs Material](https://squidfunk.github.io/mkdocs-material/)
- [release-please](https://github.com/googleapis/release-please)
- [Langfuse](https://langfuse.com/docs) — LLM observability
- [NumPyro](https://num.pyro.ai/) — Bayesian inference
