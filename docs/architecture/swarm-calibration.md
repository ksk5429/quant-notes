---
title: Swarm + calibration deep-dive
description: How 9 personas become a calibrated probability
---

# Swarm + calibration

## End-to-end data flow

```mermaid
flowchart TB
    market["Market<br/>(question + context)"]
    prescreen["3-Fish pre-screen<br/>skip if all near 0.50"]
    market --> prescreen
    prescreen --"not unknowable"--> delphi
    prescreen --"unknowable"--> skip[skip market]

    subgraph delphi[Delphi loop]
        direction TB
        r0[Round 0: 9 personas × 5 samples, independent]
        r0 --> agg0[median per persona + median across]
        agg0 --> anon[anonymized peer summary]
        anon --> r1[Round 1: same personas, seeing peers]
        r1 --> agg1[aggregate]
        agg1 --> converge{|Δspread| < ε?}
        converge -->|yes| out[consensus]
        converge -->|no| anon
    end

    delphi --> calib[Calibration pipeline]

    subgraph calib[Calibration pipeline]
        direction TB
        iso[Isotonic per category]
        va[Venn-Abers intervals]
        conf[Mondrian conformal<br/>α=0.10 singleton gate]
        shrink[Shrinkage ensemble<br/>p_final = α·p_llm + (1-α)·p_market]
        iso --> va --> conf --> shrink
    end

    calib --> trade[Trade decision]
```

## 9 personas in detail

| Name | Temperature | Cognitive frame | Prompt focus |
|---|---|---|---|
| `contrarian` | 0.8 | AOT | "what would have to be true for the crowd to be wrong here?" |
| `inside_view` | 0.5 | inside view | concrete actors + immediate causal chain |
| `outside_view` | 0.5 | reference-class | base rates from 2-3 historical analogs |
| `premortem` | 0.6 | error anticipation | "write the most plausible surprise-chain" |
| `devils_advocate` | 0.7 | adversarial collaboration | always argue the opposite of first impression |
| `quant` | 0.4 | structured Fermi | decompose into independent factors with probability rules |
| `geopolitical` | 0.5 | domain expertise | state actors, elections, diplomatic signalling |
| `macro` | 0.5 | domain expertise | rates, FX, liquidity regimes |
| `red_team` | 0.9 | independent audit | **runs on the opposite LLM provider**; attacks weakest assumption |

Each persona runs 5 samples per round by default, with a small per-sample
temperature jitter to broaden the within-persona spread.

## Aggregation math

Given per-persona sample probabilities $p_{i,j}$ for persona $i$, sample $j$:

1. Persona summary: $\bar p_i = \text{median}_j p_{i,j}$
2. Consensus: $\hat p = \text{median}_i \bar p_i$ (or trimmed-mean $\alpha$=0.1)
3. Dispersion: $\sigma = \text{std}_i \bar p_i$
4. Asymmetric extremization:

$$
\hat p_{\text{ext}}
= 0.5 + (\hat p - 0.5) \cdot f(\sigma),\quad
f(\sigma) = \begin{cases}
1 + a & \sigma \leq 0.05 \\
1 - \tfrac{a}{2} & \sigma \geq 0.20 \\
\text{linear} & \text{between}
\end{cases}
$$

with $a=0.5$ by default. Intuition: when the swarm agrees strongly, push
confidence outward; when it disagrees, shrink toward 0.5.

## Calibration pipeline

### 1. Isotonic per category

Low-bias, high-variance. Refuses to fit below 150 samples; falls through to
identity with $\epsilon$-clipping. Mathematically it's the optimal
monotone-non-decreasing regression of outcomes on raw probabilities.

### 2. Venn-Abers intervals

Distribution-free: for each new $p$, returns $[p_0, p_1]$ containing the true
posterior with validity guarantee. Midpoint is the point estimate, interval
width feeds the Kelly-fraction shrinkage.

### 3. Mondrian conformal gate

Per-category inductive conformal. Nonconformity score $= 1 - p_{\text{true}}$.
For each new $p$, compute $p$-values for both class hypotheses. Prediction
set at $\alpha = 0.10$:

- `{0}` — buy NO (or short YES)
- `{1}` — buy YES
- `{0, 1}` — **skip** (ambiguous at the current confidence level)

### 4. Shrinkage ensemble

$$
p_{\text{final}}
= \alpha_c \cdot p_{\text{llm}} + (1 - \alpha_c) \cdot p_{\text{market}}
$$

where $\alpha_c$ is per-category, shrunk empirical-Bayes toward a global α:

$$
\alpha_c^{\text{shrunk}}
= w_c \cdot \alpha_c^{\text{raw}} + (1 - w_c) \cdot \alpha_{\text{global}},
\quad
w_c = \frac{n_c}{n_c + k},\ k \approx 50
$$

On liquid markets we expect $\alpha \in [0.2, 0.5]$; on illiquid / novel
markets $[0.5, 0.8]$. If $\alpha \to 1$ on holdout, that's a diagnostic that
the category is genuinely inefficient (the market isn't contributing info) —
which is exactly the alpha K-Fish hunts.

## Brier decomposition

Murphy 1973:

$$
\text{Brier} = \text{reliability} - \text{resolution} + \text{uncertainty}
$$

where:

- $\text{reliability} = \frac{1}{N} \sum_k n_k (f_k - o_k)^2$ — small is good
- $\text{resolution} = \frac{1}{N} \sum_k n_k (o_k - \bar o)^2$ — large is good
- $\text{uncertainty} = \bar o (1 - \bar o)$ — data-bound

The new `kfish-core` path reproduces legacy Brier **exact to 4 decimal
places** across all three legacy runs — see
[Brier parity](../progress/brier-parity.md).

## Why the 3-Fish pre-screen matters

~10-20% of Polymarket markets are effectively unknowable within our research
horizon. If we predict on those, we waste LLM tokens and drag Brier upward
without positive EV. The pre-screen runs three cheap Fish
(`inside_view`, `outside_view`, `premortem`) once each; if all three land
within 0.05 of 0.50, we skip.

## Where this lives in code

- `apps/kfish-core/src/kfish_core/agents/personas.py` — the 9 persona specs
- `apps/kfish-core/src/kfish_core/agents/prompts.py` — `PROMPT_PERSONA_BASE_V2`, `PROMPT_RED_TEAM_V1`
- `apps/kfish-core/src/kfish_core/agents/delphi.py` — Delphi loop
- `apps/kfish-core/src/kfish_core/calibration/` — isotonic, venn-abers, conformal, ensemble, refit
- `apps/kfish-core/src/kfish_core/agents/swarm.py` — orchestrator
