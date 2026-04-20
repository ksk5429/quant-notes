---
title: Glossary
description: Prediction-market, calibration, and K-Fish-specific terms
---

# Glossary

Terms used throughout the K-Fish documentation. Cross-references use
`→ Term`.

## Prediction markets

**Binary market** — A market that resolves to 0 (NO) or 1 (YES) at a
specified time per a written resolution source.

**CLOB** — Central Limit Order Book. Polymarket's order-matching venue
for binary outcomes. Opposite of a constant-function market maker.

**Crowd probability** — The market price of the YES share, interpreted
as a probability. The benchmark for → BSS on prediction markets. Current
K-Fish baseline: Brier 0.206 vs crowd 0.084 (crowd is winning).

**Edge (bps)** — Signed difference between our calibrated probability
and the market price, in basis points: `10000 × (p − price)`.

**Gamma** — Polymarket's read-only REST API for market metadata
(questions, slugs, resolution sources, volume).

**HIP-4** — Hyperliquid Improvement Proposal 4. Binary perpetuals on
Hyperliquid. Not live on mainnet as of 2026-04-20; K-Fish HL handlers
feature-gated to testnet.

**Oracle (UMA)** — Decentralized dispute resolution for Polymarket.
Proposers post a price; bonders dispute; the DAO settles. K-Fish tracks
all four events → ProposePrice / DisputePrice / Settle / Reset.

**Resolution** — The event that terminates a market; `outcome ∈ {0, 1}`
plus `resolved_at` timestamp.

## Scoring and calibration

**Brier score** — Quadratic scoring rule:
$\text{Brier} = \frac{1}{N} \sum_i (p_i - y_i)^2$. Lower is better.
0.25 is random on binary questions. Formally defined in → Murphy
decomposition.

**BSS (Brier Skill Score)** — Normalized improvement vs a baseline:
$\text{BSS} = 1 - B / B_{\text{ref}}$. Current K-Fish: BSS +17.6 %
vs random, −145 % vs crowd.

**Calibration** — The property that, among forecasts assigned
probability $p$, a fraction $p$ actually occur. Measured by → ECE and
the reliability term of the → Murphy decomposition.

**Climatology** — The unconditional base rate used as a calibration
baseline (usually 0.5 on uniform binary markets).

**Discrimination** — The ability to separate events from non-events.
Captured by the resolution term of the → Murphy decomposition.

**ECE (Expected Calibration Error)** — Bucketed calibration gap:
$\text{ECE} = \sum_b \frac{n_b}{N}\,|\bar{p}_b - \bar{y}_b|$. Bin count
matters; K-Fish uses 10 bins for legacy parity.

**Extremization** — Pushing an aggregate probability away from 0.5.
K-Fish uses the asymmetric formula from [Baron 2014]: more aggressive
when the swarm agrees, less when it disagrees.

**Isotonic regression** — Monotone-non-decreasing fit to calibration
data. Piecewise-constant. Trained via PAV algorithm.

**Mondrian conformal** — Conformal prediction stratified by a
category variable. K-Fish stratifies by `(category, true_class)` so
per-class coverage holds independently.

**Murphy decomposition** — $\text{Brier} = \text{reliability} -
\text{resolution} + \text{uncertainty}$ ([Murphy 1973]). Reliability:
calibration error. Resolution: discrimination. Uncertainty: variance
of $y$.

**Venn-Abers predictor** — A conformal calibrator that returns
intervals $[p_0, p_1]$ with provable coverage. K-Fish uses the inductive
variant (IVAP) from [Vovk 2015].

## Swarm

**3-Fish pre-screen** — Three personas (`inside_view`, `outside_view`,
`premortem`) sampled once each. If all land within 0.05 of 0.50, the
market is declared *unknowable* and skipped. News-free by design.

**9-persona swarm** — The nine personas that run the full Delphi:
`contrarian`, `inside_view`, `outside_view`, `premortem`,
`devils_advocate`, `quant`, `geopolitical`, `macro`, `red_team`. Defined
in `apps/kfish-core/src/kfish_core/agents/personas.py`; weighted by
category at aggregation time.

**Asymmetric extremization** — See → Extremization.

**Delphi rounds** — Iterated forecasting with opaque per-round ID
shuffle. Each persona sees anonymized prior-round peer probabilities;
terminates on convergence (absolute spread ≤ ε) or stall + low-variance.

**Persona** — A system prompt prefix that gives a base model a specific
analytical style. K-Fish uses nine; see [Swarm Architecture](swarm.md).

## Data

**ASOF JOIN** — DuckDB's temporal join returning the most-recent row
at-or-before each probe timestamp. O(n log n) vs naive O(n²).

**Calibration row** — Tuple `(decision_id, ts, prob, outcome)` feeding
→ Brier and → ECE. Exposed as the `kfish.calibration_rows` view.

**FTS (Full-Text Search)** — DuckDB extension providing BM25 over text
columns. Built lazily on `kfish.articles` via
`fts_kfish_articles.match_bm25`.

**Kiwi** — Korean morphological analyzer (`kiwipiepy`). Produces
part-of-speech-tagged tokens; K-Fish keeps `NNG/NNP/VV/VA/MAG/MAJ/SL/SN`
in `body_ko_tokens` for BM25.

**SimHash-64** — [Charikar 2002] locality-sensitive hash. K-Fish
fingerprints article title+lead; Hamming distance ≤ 4 within a 48h
window marks a duplicate.

## Operations

**Cutover gate** — See → Math-parity gate. The current ADR-0002 gate.
Renamed explicitly so it isn't mistaken for forecast parity on fresh
markets.

**Langfuse** — Open-source LLM observability. K-Fish self-hosts on
Oracle A1 (ADR-0007) so trading traces never leave in-house infra.

**Math-parity gate** — Seven consecutive UTC-day passes where the
new numpy Brier matches the legacy engine's stored Brier within 0.002
on archived retrodiction JSON.

**Papago** — Naver's machine translator. K-Fish uses it for cheap
title-only translation (~₩60/run) to populate `title_en` for BM25
queries in English.

**uv workspace** — Rust-based Python package manager. K-Fish uses it
to manage the monorepo (ADR-0006).

## K-Fish-specific

**hypekr-bot** — The private Telegram bot on Hyperliquid.
`apps/hypekr-bot/`.

**kfish-common** — Shared utilities package: DuckDB client, Langfuse
wrapper, LLM router, config. `packages/kfish-common/`.

**kfish-core** — The 9-agent swarm, calibration, markets, news,
pipeline, CLI. `apps/kfish-core/`.

**polymarket-oracle-risk** — The MIT-licensed public Bayesian risk
scorer. `polymarket-oracle-risk/`.

## R-series rules

The project rules enumerated in `CLAUDE.md`:

| Rule | Short form |
|---|---|
| R1 | Source verification — no hallucinated citations |
| R2 | Paper trading first |
| R3 | Interpretability — reasoning chain required |
| R4 | Calibration mandatory — raw LLM probs never used |
| R5 | Human-AI boundary — final approval is human |
| R6 | Grounded in literature |
| R7 | ADR discipline — immutable except for `status:` |
| R8 | License boundary — nothing public imports private |
