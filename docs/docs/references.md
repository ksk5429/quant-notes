---
title: References
description: Bibliography for every algorithmic decision in K-Fish
---

# References

Rule R6 (`CLAUDE.md`): every algorithmic decision cites a paper or a
documented OSS implementation. This page is the consolidated index.

## Scoring and calibration

**Brier, G. W.** (1950). *Verification of forecasts expressed in terms
of probability.* Monthly Weather Review, 78(1), 1–3.
[DOI](https://doi.org/10.1175/1520-0493(1950)078%3C0001:VOFEIT%3E2.0.CO;2)

**Murphy, A. H.** (1973). *A new vector partition of the probability
score.* Journal of Applied Meteorology, 12(4), 595–600.
[DOI](https://doi.org/10.1175/1520-0450(1973)012%3C0595:ANVPOT%3E2.0.CO;2)
Basis for the reliability / resolution / uncertainty decomposition used
by `decompose_brier` and `decompose_brier_sql`.

**Naeini, M. P., Cooper, G., & Hauskrecht, M.** (2015). *Obtaining
well calibrated probabilities using Bayesian binning.* AAAI.
[arXiv](https://arxiv.org/abs/1411.0411)
ECE binning convention.

**Kumar, A., Liang, P. S., & Ma, T.** (2019). *Verified uncertainty
calibration.* NeurIPS.
[arXiv](https://arxiv.org/abs/1909.10155)
Discusses ECE's bias and bin-sensitivity. K-Fish uses 10 bins for
legacy parity; acknowledged as a coarse metric.

## Conformal / quantile calibration

**Vovk, V., Petej, I., & Fedorova, V.** (2015). *Large-scale
probabilistic predictors with and without guarantees of validity.*
NeurIPS.
[arXiv](https://arxiv.org/abs/1511.00213)
Venn-Abers predictors and the inductive (IVAP) variant we use via
`venn-abers` package.

**Vovk, V.** (2022). *Conformal e-prediction.*
[arXiv](https://arxiv.org/abs/2001.00106)
Background on conformal prediction theory.

**Romano, Y., Patterson, E., & Candès, E.** (2019). *Conformalized
quantile regression.* NeurIPS.
[arXiv](https://arxiv.org/abs/1905.03222)
Mondrian-style stratified conformal — K-Fish stratifies by
`(category, true_class)`.

**Cauchois, M., Gupta, S., & Duchi, J.** (2021). *Knowing what you know:
valid and validated confidence sets in multiclass and multilabel
prediction.* JMLR.
[arXiv](https://arxiv.org/abs/2004.10181)

## Wisdom of crowds and extremization

**Galton, F.** (1907). *Vox populi.* Nature, 75, 450–451.
The original crowd-median result.

**Tetlock, P. E.** (2005). *Expert political judgment: How good is it?
How can we know?* Princeton University Press.
Foundational. K-Fish personas draw from Tetlock's "fox vs hedgehog"
typology.

**Baron, J., Mellers, B. A., Tetlock, P. E., Stone, E., & Ungar, L. H.**
(2014). *Two reasons to make aggregated probability forecasts more
extreme.* Decision Analysis, 11(2), 133–145.
[DOI](https://doi.org/10.1287/deca.2014.0293)
Basis for the asymmetric extremization formula in
`agents.aggregator.asymmetric_extremize`.

**Atanasov, P., Rescober, P., Stone, E., Swift, S., Servan-Schreiber, E.,
Tetlock, P., Ungar, L., & Mellers, B.** (2017). *Distilling the wisdom of
crowds: Prediction markets vs. prediction polls.* Management Science,
63(3), 691–706.

**Schoenegger, P., Tuminello, P., Karger, E., & Tetlock, P.** (2024).
*Wisdom of the silicon crowd: LLM ensemble prediction capabilities
rival human crowd accuracy.* Science Advances.
[arXiv](https://arxiv.org/abs/2402.19379)
Direct basis for the 9-persona LLM-ensemble design.

## Korean NLP

**Lee, M.-h.** *Kiwi: Korean morphological analyzer.*
[GitHub](https://github.com/bab2min/Kiwi) /
[kiwipiepy](https://github.com/bab2min/kiwipiepy)
The tokenizer K-Fish uses for FTS indexing.

## Similarity hashing

**Charikar, M.** (2002). *Similarity estimation techniques from
rounding algorithms.* STOC.
[ACM](https://dl.acm.org/doi/10.1145/509907.509965)
SimHash origin. K-Fish uses the 64-bit variant with Hamming ≤ 4 over a
48h window.

**Manku, G. S., Jain, A., & Sarma, A. D.** (2007). *Detecting
near-duplicates for web crawling.* WWW.
[ACM](https://dl.acm.org/doi/10.1145/1242572.1242592)
SimHash for large-scale dedup — the pattern we implement.

## Translation

**Papago NMT.**
[Papago developer docs](https://developers.naver.com/docs/papago/).
The Korean-to-English machine translation API.

## LLM clients

**Anthropic Claude.**
[API docs](https://docs.anthropic.com/).
Primary forecaster (ADR-0005).

**OpenAI GPT.**
[API docs](https://platform.openai.com/docs/).
Independent evaluator (ADR-0005).

## Data platform

**Raasveldt, M., & Mühleisen, H.** (2019). *DuckDB: An embeddable
analytical database.* SIGMOD.
[ACM](https://dl.acm.org/doi/10.1145/3299869.3320212)
The warehouse substrate.

**DuckDB Labs.** *The ASOF JOIN documentation.*
[docs](https://duckdb.org/docs/guides/sql_features/asof_join.html)

## Telegram / trading stack

**aiogram 3.** [docs](https://docs.aiogram.dev/en/latest/). Async
Telegram Bot API framework. ADR-0003 rationale.

**Hyperliquid.** [docs](https://hyperliquid.gitbook.io/hyperliquid-docs/).
Per-user agent wallet pattern (ADR-0004).

## Observability

**Langfuse.** [docs](https://langfuse.com/docs). Self-hosted LLM
tracing (ADR-0007).

## Software architecture references

**Nygard, M. T.** (2018). *Release It! Design and Deploy Production-Ready
Software* (2nd ed.). Pragmatic Bookshelf. Patterns for failure-tolerant
pipelines; K-Fish nightly uses bulkheads per-step.

**ArchitectureDecisionRecord.** *MADR 4 spec.*
[GitHub](https://github.com/adr/madr). The ADR format used in
`docs/decisions/`.

## Legal (see separate Legal section)

- **가상자산이용자보호법 (VAUPA)** — Act on the Protection of Users of
  Virtual Assets. Effective 2024-07-19.
- **특정 금융거래정보의 보고 및 이용 등에 관한 법률 (특금법)** — the
  KoFIU VASP registration regime.
- **형법 제246조, 제247조** — Criminal Code, gambling and operating
  gambling places.

Full statute citations and commentary in the private
[`runbooks/kr-legal-brief.md`](https://github.com/ksk5429/kfish/blob/main/runbooks/kr-legal-brief.md);
public-safe summary at [Korean Legal Status](../legal/summary.md).
