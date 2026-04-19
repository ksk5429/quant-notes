---
title: Oracle risk analyzer
description: The public polymarket-oracle-risk package
---

# Oracle risk

The public [`polymarket-oracle-risk`][repo] package scores Polymarket markets
on manipulation-risk, from 0 (objective named source) to ~1 (pure moderator
judgment). It's the refusal-gate input for K-Fish trading and also stands
alone on PyPI as a PhD-adjacent research contribution.

## Feature vector

13 features total (blueprint §2.3), in stable column order:

| # | Feature | Signal |
|---|---|---|
| 1 | `liveness_hours` | shorter = less review time |
| 2 | `log1p_bond_usdc` | smaller bond = thinner deterrent |
| 3 | `subjectivity_score` | [0, 1] — LLM-rated |
| 4 | `resolution_source_concreteness` | 1.0 = named authoritative source |
| 5 | `hhi_uma_top10` | Herfindahl of UMA top-10 voters |
| 6 | `log1p_market_volume_usd` | attack incentive |
| 7 | `log1p_max_single_wallet_position_usd` | whale flag |
| 8 | `price_distance_from_extremes` | `min(p, 1-p)` — contested-ness |
| 9 | `hours_to_resolution` | last-minute surprise window |
| 10 | `price_moved_pct_24h` | late divergence from consensus |
| 11 | `proposer_whitelisted` | MOOv2 post-UMIP-189 flag |
| 12 | `similar_market_dispute_rate` | beta-binomial shrunk per category |
| 13 | `llm_grok_disagrees_with_market` | inter-model disagreement |

## Posterior fit

NumPyro NUTS with priors set to reflect a ~2% base rate for disputes:

$$
\beta \sim \mathcal{N}(0, I),\ \alpha \sim \mathcal{N}(-3, 2),\
\Pr(\text{disputed}\mid x) = \sigma(\alpha + \beta^\top x)
$$

For 13 features and <50 high-profile dispute observations, the posterior is
wide by design. `predict_one()` returns the posterior mean + 95% credible
interval; **callers report the interval, not the point**.

## Zelenskyy regression test

The canonical subjectivity-scorer fail mode is to let vague "consensus of
credible reporting" language through as if it were an objective source. The
Zelenskyy-suit market (May-July 2025) is the archetypal case. Every release
of `polymarket-oracle-risk` must score that text **≥ 0.75**:

```python
from polymarket_oracle_risk.subjectivity import (
    ZELENSKYY_RESOLUTION_TEXT, rule_based_score
)
assert rule_based_score(ZELENSKYY_RESOLUTION_TEXT).score >= 0.75
```

The test lives at `tests/test_subjectivity_zelenskyy.py` and gates every PR.

## Refusal gate

$$
\text{action}(p, \text{risk}) =
\begin{cases}
\textbf{refuse} & \text{risk.mean} > 0.35 \\
\textbf{refuse} & \text{risk.width} > 0.50 \\
\text{size} \times \text{cap}(\text{risk.mean}) & \text{otherwise}
\end{cases}
$$

| `risk.mean` | Position cap vs unconstrained Kelly |
|---|---|
| ≤ 0.15 | 100% |
| ≤ 0.25 | 30% |
| ≤ 0.35 | 10% |
| > 0.35 | **refuse** |

The width guard is a CVaR-style second gate — a wide posterior means we don't
trust the point estimate, so we refuse even if the mean is below the hard
threshold. This caught the Zelenskyy-class case in backtest.

## Honest limitations

- **<50 high-profile disputes** → wide posteriors
- **Managed Proposer regime change** (UMIP-189, Aug 2025) → segment training
  data; old dispute rates don't apply
- **HIP-4 has no dispute history yet** → scores there are low-confidence until
  mainnet accumulates samples
- **UMA top-10 concentration** is community-derived, not officially published
  — re-verify from the voting-v2 subgraph before using as a load-bearing
  feature.

## Where this lives

- [`ksk5429/polymarket-oracle-risk`][repo] — the public repo (MIT)
- `src/polymarket_oracle_risk/features.py` — feature construction
- `src/polymarket_oracle_risk/train.py` — NumPyro model + `FitSummary` pickle
- `src/polymarket_oracle_risk/scorer.py` — `PosteriorModel` + `RiskScore`
- `src/polymarket_oracle_risk/subjectivity.py` — rule-based + LLM path
- `src/polymarket_oracle_risk/refusal_gate.py` — the gate

[repo]: https://github.com/ksk5429/polymarket-oracle-risk
