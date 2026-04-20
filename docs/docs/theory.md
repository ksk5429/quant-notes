---
title: Theory and Math
description: Brier score, Murphy decomposition, extremization, calibration metrics
---

# Theory and Math

This document states the scoring rules and calibration metrics that the K-Fish
stack optimizes against, and explains why each one is used. Every formula has a
corresponding implementation in
[`apps/kfish-core/src/kfish_core/calibration/brier.py`](https://github.com/ksk5429/kfish/blob/main/apps/kfish-core/src/kfish_core/calibration/brier.py).

## Brier score

Given a set of probabilistic forecasts $f_i \in [0, 1]$ for binary outcomes
$o_i \in \{0, 1\}$, the Brier score [Brier 1950] is the mean squared error

$$
B \;=\; \frac{1}{N} \sum_{i=1}^{N} (f_i - o_i)^2 \;\in\; [0, 1].
$$

Perfect foresight gives $B = 0$. Always predicting $0.5$ gives $B = 0.25$.
Random guessing on a balanced dataset gives $B \approx 0.33$. The K-Fish
baseline is $B = 0.206$ on 200 retrodicted Polymarket questions
([CLAUDE.md](https://github.com/ksk5429/kfish/blob/main/CLAUDE.md)).

The computation lives in
[`brier_score`](https://github.com/ksk5429/kfish/blob/main/apps/kfish-core/src/kfish_core/calibration/brier.py).

## Murphy decomposition

[Murphy 1973] showed Brier decomposes cleanly when forecasts are grouped into
$K$ bins with $n_k$ forecasts per bin, mean forecast $\bar f_k$, empirical
frequency $\bar o_k$, and overall base rate $\bar o$:

$$
B \;=\; \underbrace{\frac{1}{N}\sum_k n_k (\bar f_k - \bar o_k)^2}_{\text{reliability}}
\;-\; \underbrace{\frac{1}{N}\sum_k n_k (\bar o_k - \bar o)^2}_{\text{resolution}}
\;+\; \underbrace{\bar o (1 - \bar o)}_{\text{uncertainty}}.
$$

Three orthogonal quantities:

| Component | Interpretation | Direction |
|---|---|---|
| **Reliability** | Distance between stated confidence and observed frequency within each bin. Captures calibration error. | Minimize |
| **Resolution** | How far bin frequencies spread from the base rate. Captures discrimination — the ability to separate "will happen" from "won't happen". | Maximize |
| **Uncertainty** | Variance of the outcome, fixed by the dataset. | Not controllable |

Implemented twice — numpy version in
[`decompose_brier`](https://github.com/ksk5429/kfish/blob/main/apps/kfish-core/src/kfish_core/calibration/brier.py)
and SQL version in `BRIER_DECOMPOSITION_SQL`
([same file](https://github.com/ksk5429/kfish/blob/main/apps/kfish-core/src/kfish_core/calibration/brier.py)) so
the warehouse and the test suite compute identical numbers.

!!! note "Why both?"
    Re-implementing the decomposition in DuckDB SQL means every
    `refit_calibrators` run (thousands of rows) reports the same number as the
    unit test on a 20-row fixture. Divergence between the two is a bug signal.

## Why Brier, not log-loss or CRPS

Three scoring rules are candidates for binary forecasts:

| Rule | Formula | Strictly proper? | Bounded? | Behavior near 0/1 |
|---|---|---|---|---|
| Brier | $(f - o)^2$ | yes | $[0, 1]$ | finite |
| Log-loss | $-[o \log f + (1 - o) \log (1 - f)]$ | yes | $[0, \infty)$ | diverges |
| CRPS | integral form — reduces to Brier for binary | yes | $[0, 1]$ | finite |

Log-loss penalizes overconfidence at 0 or 1 with infinite loss. For LLM-sampled
probabilities, a single $p = 0.999$ on a resolved-NO market would dominate the
mean even across hundreds of markets. Brier keeps every outlier bounded, which
matters when the probability generator is stochastic and occasionally extreme.

CRPS reduces to Brier in the binary case, so there is nothing to gain. For
continuous outcomes the choice would matter; binary prediction markets make it
moot.

## Calibration vs discrimination

The Murphy decomposition makes the distinction formal:

- **Calibration** is reliability: when a forecaster says $0.70$, the long-run
  frequency should be $0.70$. Low reliability term $\Rightarrow$ well
  calibrated.
- **Discrimination** is resolution: across bins, do frequencies actually differ
  from the base rate? High resolution $\Rightarrow$ the forecaster
  distinguishes events.

A forecaster that always outputs the base rate is perfectly calibrated
(reliability $= 0$) but has zero resolution, so its Brier equals uncertainty.
A useful forecaster sacrifices a little calibration for a lot of resolution.

## Brier skill score vs climatology

Skill is relative to a naïve climatology forecaster that always reports the
base rate $\bar o$. That forecaster has $B_{\text{clim}} = \bar o (1 - \bar o)
= $ uncertainty. The skill score is

$$
\mathrm{BSS} \;=\; 1 - \frac{B}{B_{\text{clim}}}
\;=\; 1 - \frac{B}{\bar o (1 - \bar o)}.
$$

$\mathrm{BSS} > 0$ beats climatology; $\mathrm{BSS} = 1$ is perfection;
negative means worse than the base rate. K-Fish currently sits at
$\mathrm{BSS} \approx +0.176$ against random, but has not yet beaten the crowd
($B_{\text{crowd}} \approx 0.084$).

Computed as
[`BrierDecomposition.skill_score`](https://github.com/ksk5429/kfish/blob/main/apps/kfish-core/src/kfish_core/calibration/brier.py).

## Expected Calibration Error

ECE [Naeini 2015] is a single-number summary of the reliability diagram. Bin
forecasts into $K$ equal-width bins; compute per-bin mean confidence
$\mathrm{conf}(B_k)$ and accuracy $\mathrm{acc}(B_k)$; take weighted mean
deviation:

$$
\mathrm{ECE} \;=\; \sum_{k=1}^{K} \frac{|B_k|}{N} \,
\bigl| \mathrm{acc}(B_k) - \mathrm{conf}(B_k) \bigr|.
$$

Default $K = 15$ in
[`compute_ece`](https://github.com/ksk5429/kfish/blob/main/apps/kfish-core/src/kfish_core/calibration/brier.py).

!!! warning "Bin count sensitivity"
    ECE with $K = 10$ and ECE with $K = 30$ on the same data can differ by a
    factor of two. [Kumar 2019] showed ECE is biased downward for small $K$.
    We pin $K = 15$ so successive refits are comparable; report raw bin
    contents in the reliability diagram rather than letting $K$ hide them.

## Why extremization

LLM probability distributions are biased toward $0.5$. [Galton 1907] and
[Tetlock 2005] showed that mechanically sharpening a crowd's consensus — pushing
the mean away from the prior — often improves Brier, provided the forecasters
are not catastrophically miscalibrated.

[Baron et al. 2014] derived the asymmetric extremization formula: push harder
when forecasters agree, pull back toward $0.5$ when they disagree. K-Fish uses
the dispersion-sensitive variant

$$
f = \begin{cases}
1 + a & \text{if } s \le 0.05 \\
1 - 0.5 a & \text{if } s \ge 0.20 \\
\text{linear interp} & \text{otherwise}
\end{cases}
\qquad
p' = 0.5 + (p - 0.5) \cdot f,
$$

where $s$ is the standard deviation of per-persona probabilities and
$a = 0.5$ is the aggressiveness hyperparameter. Implemented as
[`asymmetric_extremize`](https://github.com/ksk5429/kfish/blob/main/apps/kfish-core/src/kfish_core/agents/aggregator.py).

!!! tip "When extremization hurts"
    On unknowable markets (true $P = 0.5$), sharpening amplifies noise. The
    3-Fish pre-screen ([swarm.md](swarm.md)) skips these before the swarm even
    runs, so extremization never sees them.

## References

- Baron J, Mellers BA, Tetlock PE, et al. (2014). *Two reasons to make aggregated probability forecasts more extreme.* Decision Analysis 11(2).
- Brier GW (1950). *Verification of forecasts expressed in terms of probability.* Monthly Weather Review 78(1): 1–3.
- Galton F (1907). *Vox populi.* Nature 75: 450–451.
- Kumar A, Liang P, Ma T (2019). *Verified uncertainty calibration.* NeurIPS.
- Murphy AH (1973). *A new vector partition of the probability score.* Journal of Applied Meteorology 12(4): 595–600.
- Naeini MP, Cooper G, Hauskrecht M (2015). *Obtaining well calibrated probabilities using Bayesian binning.* AAAI.
- Tetlock PE (2005). *Expert political judgment.* Princeton University Press.
