# K-Fish compounding infrastructure: a 90-day build guide

**Ship three things in parallel**: a DuckDB+Langfuse calibration spine that every other system plugs into, a `polymarket-oracle-risk` analyzer that turns the Zelenskyy-suit-class disaster into a feature, and a Korean-first Telegram bot on Hyperliquid that earns builder-fee revenue while HIP-4 moves from testnet to mainnet. None of these blocks on the PhD. The calibration spine is the single highest-leverage investment — it turns every other project into a compounding data asset. The oracle analyzer is publishable open-source credibility. The Korean bot is the only one with a short path to MRR. The 90-day plan below sequences them so that Weeks 1-4 build shared infrastructure used by all three, Weeks 5-8 ship the bot MVP and the oracle analyzer v1, and Weeks 9-13 harden calibration and launch publicly.

**Time budget assumption**: ~15 hours/week (PhD-adjacent). Everything below is scoped to that. If you get more time, HIP-4 mainnet work moves earlier; if less, drop the Next.js companion page entirely and keep the Telegram bot as the only end-user surface.

---

## Part 1 — DuckDB calibration spine + Korean news scraper + hybrid deployment

### Why this is the core investment

Every other system reads and writes to one DuckDB file (`kfish.duckdb`) and logs every LLM call to Langfuse. Trade logs, news, agent outputs, oracle events, and risk scores all share a schema and timeline. That single fact is what makes calibration possible: you can ASOF-join any forecast to the market price at the moment it was made, to the news article that moved the 9-agent swarm, to the oracle event that eventually settled it.

### 1.1 DuckDB schema — production DDL

Store everything as **immutable append** with `TIMESTAMPTZ` in UTC; render KST only at display time. DuckDB treats foreign keys as advisory — enforce integrity in application code and via dbt tests.

```sql
-- kfish.duckdb — core tables
INSTALL fts; LOAD fts; INSTALL json; LOAD json;
CREATE SCHEMA IF NOT EXISTS kfish;

CREATE TABLE kfish.venues (
  venue_id VARCHAR PRIMARY KEY, display_name VARCHAR NOT NULL,
  base_url VARCHAR, fee_bps INTEGER, created_at TIMESTAMPTZ DEFAULT now());

CREATE TABLE kfish.markets (
  market_id VARCHAR PRIMARY KEY, venue_id VARCHAR NOT NULL REFERENCES kfish.venues(venue_id),
  native_id VARCHAR NOT NULL, slug VARCHAR, question TEXT NOT NULL,
  category VARCHAR, subcategory VARCHAR, resolution_source TEXT,
  opened_at TIMESTAMPTZ, closes_at TIMESTAMPTZ, resolved_at TIMESTAMPTZ,
  resolution_value DOUBLE, meta JSON, UNIQUE(venue_id, native_id));

CREATE TABLE kfish.prices (
  market_id VARCHAR NOT NULL REFERENCES kfish.markets(market_id),
  ts TIMESTAMPTZ NOT NULL, outcome VARCHAR NOT NULL,
  price DOUBLE NOT NULL, best_bid DOUBLE, best_ask DOUBLE,
  volume_24h DOUBLE, liquidity DOUBLE, source VARCHAR);
CREATE INDEX idx_prices_mid_ts ON kfish.prices(market_id, ts);

CREATE TABLE kfish.decisions (
  decision_id UUID PRIMARY KEY DEFAULT uuid(),
  market_id VARCHAR NOT NULL REFERENCES kfish.markets(market_id),
  ts TIMESTAMPTZ NOT NULL, horizon_hours DOUBLE,
  ensemble_prob DOUBLE NOT NULL, ensemble_method VARCHAR NOT NULL,
  ensemble_conf DOUBLE, market_price_at_ts DOUBLE,
  edge_bps INTEGER, kelly_frac DOUBLE, traded BOOLEAN DEFAULT FALSE,
  trade_size_usd DOUBLE, prompt_version VARCHAR, git_sha VARCHAR, notes TEXT);
CREATE INDEX idx_decisions_mid_ts ON kfish.decisions(market_id, ts);

CREATE TABLE kfish.agent_outputs (
  decision_id UUID NOT NULL REFERENCES kfish.decisions(decision_id),
  agent_name VARCHAR NOT NULL, agent_version VARCHAR NOT NULL,
  prob DOUBLE NOT NULL, confidence DOUBLE, reasoning TEXT,
  input_tokens INTEGER, output_tokens INTEGER, cost_usd DOUBLE,
  latency_ms INTEGER, langfuse_trace_id VARCHAR, model VARCHAR,
  PRIMARY KEY (decision_id, agent_name));

CREATE TABLE kfish.outcomes (
  market_id VARCHAR PRIMARY KEY REFERENCES kfish.markets(market_id),
  resolved_at TIMESTAMPTZ NOT NULL, outcome DOUBLE NOT NULL,
  pnl_if_held DOUBLE, notes TEXT);

CREATE TABLE kfish.articles (
  article_id UUID PRIMARY KEY DEFAULT uuid(), source VARCHAR NOT NULL,
  url VARCHAR UNIQUE NOT NULL, title_ko TEXT, title_en TEXT,
  body_ko TEXT, body_en TEXT, published_at TIMESTAMPTZ,
  fetched_at TIMESTAMPTZ DEFAULT now(), simhash BIGINT,
  embedding FLOAT[1024], meta JSON);
```

**The ASOF JOIN is the feature that makes this stack worth using.** Classic pattern for "what was the market price when we made each forecast":

```sql
SELECT d.decision_id, d.ts AS decided_at, d.ensemble_prob,
       p.price AS market_price, d.ensemble_prob - p.price AS edge
FROM kfish.decisions d
ASOF LEFT JOIN kfish.prices p
  ON d.market_id = p.market_id AND d.ts >= p.ts;
```

### 1.2 Parquet partitioning + git versioning

Never commit the `.duckdb` binary. Snapshot nightly to partitioned Parquet, commit that directory, and let DVC point to bulk blobs in Backblaze B2 or Cloudflare R2:

```sql
COPY (SELECT d.*, m.venue_id,
             date_part('year', d.ts) AS year,
             date_part('month', d.ts) AS month
      FROM kfish.decisions d JOIN kfish.markets m USING (market_id))
TO 'data/decisions'
(FORMAT PARQUET, PARTITION_BY (venue_id, year, month),
 OVERWRITE_OR_IGNORE, COMPRESSION ZSTD);
```

Backups follow 3-2-1 via `restic → B2` ($0.30/mo for 50 GB), with an `rclone sync` warm tier to Cloudflare R2 (zero egress) and a local borg copy. Every scheduled job pings `healthchecks.io` on success — when a heartbeat is missed, it pages you.

### 1.3 Brier decomposition in SQL (Murphy 1973)

This is the calibration diagnostic that every other part reads. One query produces reliability / resolution / uncertainty and reconstructs Brier — run it nightly via dbt:

```sql
WITH bins AS (
  SELECT floor(ensemble_prob*20)/20.0 AS p_bin, ensemble_prob, outcome,
         count(*)     OVER () AS n_total,
         avg(outcome) OVER () AS o_bar
  FROM kfish.calibration_rows),
per_bin AS (
  SELECT p_bin, count(*) AS n_k, avg(ensemble_prob) AS f_k,
         avg(outcome) AS o_k, any_value(n_total) AS N,
         any_value(o_bar) AS o_bar FROM bins GROUP BY p_bin)
SELECT sum(n_k*(f_k-o_k)*(f_k-o_k))/any_value(N)       AS reliability,
       sum(n_k*(o_k-o_bar)*(o_k-o_bar))/any_value(N)   AS resolution,
       any_value(o_bar)*(1-any_value(o_bar))           AS uncertainty
FROM per_bin;
```

Target: `ECE < 0.05`, `reliability / brier < 0.10`. When reliability drifts up, refit isotonic (Part 4.4).

### 1.4 Korean news scraper — source reality check

| Source | Access | Difficulty | Notes (Apr 2026) |
|---|---|---|---|
| Naver News Open API | JSON API, 25k calls/day free tier — **verify at developers.naver.com tonight** | Easy | Developer portal has been migrating some APIs to paid NCP. Biggest build risk. |
| Yonhap 연합뉴스 | RSS | Easy | Authoritative on macro + DPRK |
| Hankyoreh | Rich RSS | Easy | Left-of-centre balance |
| Chosun | Partial RSS | Medium | Respect paywall paths |
| Blockmedia | RSS | Easy | **Essential for KR crypto** |
| Daily NK | RSS | Easy | Unique DPRK signal |
| Dispatch | JS-heavy | Hard | Low prediction-market signal — deprioritize |
| DC Inside crypto/politics galleries | HTML, anti-bot | Hard | Sentiment noise, not source of record; ≤1 req/s |

**Tokenizer**: `kiwipiepy 0.22.x` wins. Single pip wheel, no Java, Py3.12 support, 86.7% disambiguation (Kiwi paper, KJDH 2024) vs 50-70% for deep-learning analyzers. Skip KoNLPy-Mecab on Windows — painful; use it only inside Docker if you ever need it.

**Translation router**: Papago (NCP, ~$0.015/1k chars) for bulk first-pass title+lead, Claude Sonnet for the ~100/day scoring-relevant articles. Target ~$330/month total for 1000 articles/day. All-DeepL ($1150/mo) and all-Papago ($675/mo) are strictly dominated.

**Dedup**: two-layer. SimHash(64-bit) on normalized title+lead stored as `BIGINT` with SQL `bit_count(simhash # ?) < 5`. Then BGE-m3 embeddings (1024-d, native Korean, 8k context) with cosine > 0.92 within a 48h window. `dragonkue/bge-m3-ko` slightly beats base BGE on KR benchmarks if you have VRAM.

**Langfuse telemetry** — v3 with the `@observe` decorator creates one trace per article, a generation per LLM call, and attaches `prompt_version` metadata so you can filter A/B tests in the UI:

```python
@observe(as_type="generation", name="claude-extract-signal")
def extract_signal(title, body, *, prompt_version):
    lfc = get_client()
    lfc.update_current_generation(
        model="claude-sonnet-4-5-20250929",
        metadata={"prompt_version": prompt_version, "source_lang": "ko"},
        input={"title": title, "body": body[:4000]})
    resp = anthropic_client.messages.create(
        model="claude-sonnet-4-5-20250929", max_tokens=800,
        system=PROMPT_EXTRACT_SIGNAL_V3,
        messages=[{"role":"user","content": f"TITLE:{title}\n\n{body}"}])
    lfc.update_current_generation(
        output=resp.content[0].text,
        usage_details={"input": resp.usage.input_tokens,
                       "output": resp.usage.output_tokens})
    return parse_json(resp.content[0].text)
```

**DuckDB FTS for Korean** has one gotcha: the built-in Snowball stemmers don't speak Korean. Pre-tokenize with Kiwi into a whitespace-joined column and index with `stemmer='none'`. FTS does not auto-update on INSERT — rebuild nightly.

### 1.5 Prompt library skeleton (versioned constants)

Three prompts cover 80% of Korean news processing. Each is a module constant with an explicit `_V<n>` suffix passed into Langfuse metadata:

- `PROMPT_EXTRACT_SIGNAL_V3` returns strict JSON with `summary_en`, `entities`, `topics`, `sentiment ∈ [-1,1]`, `novelty`, `market_relevance[]`, `time_horizon_hours`, `credibility.source_tier ∈ {A,B,C,D}`. Treat Yonhap as A; DC Inside as C/D.
- `PROMPT_TRANSLATE_V2` preserves honorific register, maps canonical English names (금융위 → FSC), and keeps direct quotations verbatim.
- `PROMPT_RED_TEAM_V1` returns `{counterargument, plausibility, suggested_adjustment_bps}` as the 9th-agent adversarial check.

### 1.6 Repo layout (`github.com/ksk5429/kfish`, private)

```
kfish/
├── .github/workflows/   nightly-snapshot.yml, ci.yml, backfill.yml
├── docker/              Dockerfile, docker-compose.yml (langfuse+scraper+prefect)
├── migrations/          V0001__init.sql, V0002__add_articles_fts.sql
├── dbt_kfish/           models/{staging, calibration, marts}
├── kfish/               config, db, sources/{naver,yonhap,…}, nlp/, agents/,
│                        prompts/, ensemble.py, trade.py, markets/, tasks.py
├── streamlit_app/       Home.py + pages/{Calibration,Decisions,News}.py
├── obsidian/            vault (journals)
├── tests/               pytest + hypothesis
├── scripts/             migrate.py, snapshot.py, restore.py
├── pyproject.toml  uv.lock  .python-version (3.12)  .env.example
```

### 1.7 Tonight on Windows 11 — bootstrap commands

```powershell
# PowerShell as admin
wsl --install -d Ubuntu-24.04
```

```bash
# inside WSL
curl -LsSf https://astral.sh/uv/install.sh | sh && exec $SHELL
uv python install 3.12
mkdir -p ~/code && cd ~/code
git clone git@github.com:ksk5429/kfish.git && cd kfish
echo "3.12" > .python-version
uv venv && uv sync
uv run python -c "import duckdb, kiwipiepy, langfuse; print('ok')"
uv run python scripts/migrate.py

# self-hosted Langfuse on the same machine
git clone https://github.com/langfuse/langfuse.git ~/code/langfuse
cd ~/code/langfuse && docker compose up -d
```

### 1.8 Deployment economics

| Component | Pick | Monthly cost |
|---|---|---|
| VPS (Langfuse + Prefect + scraper + DuckDB on one box) | **Oracle Cloud Ampere A1 (4 OCPU / 24 GB free)**; fallback **Hetzner CX32 ~€7** | $0-10 |
| Claude API (targeted hybrid translation + agents) | Sonnet 4.5 with batch 50% off + prompt caching 90% off | $80-200 |
| Papago (bulk KR→EN) | Naver Cloud Platform AI·NAVER API | $200-250 (verify tier) |
| B2 backup + R2 warm tier + domain + CF | — | $2-3 |
| healthchecks.io + UptimeRobot + Grafana Cloud | free tiers | $0 |
| GitHub Actions (daily dbt + restic + nightly snapshot) | < 2000 min/mo cap | $0 |
| **Total realistic** | | **~$290-465/mo** |

At a $10-100K bankroll that's 0.3-3%/month in infra — acceptable if the calibrated edge is real. Biggest knob is Papago; cutting to title+lead-only and reserving Claude for the 50 hottest articles drops total to ~$150/mo.

**Prefect 3 self-hosted on the VPS handles minute-level jobs; GitHub Actions handles daily ones.** Scheduled Actions on private repos pause after 60 days of no pushes and can be 10-30 min late — never use them for the trading loop.

**Secrets**: SOPS + age for git-committed encrypted config, Doppler free tier for runtime injection on the VPS, GitHub Actions secrets for CI. Never commit unencrypted `.env`.

### 1.9 Honest failure modes

Naver News Open API quota is unverified — confirm before committing architecture; fallback to RSS covers ~70% of signal. Oracle Free Tier provisioning is notoriously hard in popular regions. DuckDB FKs are advisory in v1.x. GitHub Actions cron is fragile for private-repo always-on use. ARM wheels for Kiwi/DuckDB/BGE-m3 exist as of April 2026 but verify with `uv sync` on the actual instance.

---

## Part 2 — The oracle manipulation risk analyzer

### 2.1 The April 2026 oracle landscape — what actually happened

**UMA Optimistic Oracle (Polymarket's resolver)**. Managed OOV2 since UMIP-189 (August 12, 2025) restricts proposing to ~37 whitelisted addresses (Risk Labs + Polymarket operators); **disputes remain permissionless**. Default liveness is 2 hours, extended to 48 hours for high-stakes political markets. Minimum bond tracks `store.computeFinalFee` — historically ~$750 USDC for routine Polymarket proposals, higher for political markets. DVM voting: 24h commit + 24h reveal with ~0.1% UMA slash per incorrect vote.

The five disputes that matter for feature engineering:

- **Zelenskyy suit market (May 22 – July 8, 2025)** — the canonical governance-attack case study. $200-242M total volume. Resolution text "consensus of credible reporting" — no named source. At the June 24 NATO Hague summit, >40 outlets described Zelenskyy's outfit as a suit. UMA resolved **NO** on "insufficient consensus." ~23M UMA (~$25M) cast for NO; community analysis alleged 4 holders controlling >40% of UMA supply. Derek Guy (@dieworkwear) held a $3.6M NO position. This market would have scored **0.45-0.55** in our framework → hard refuse.
- **Ukraine rare-earth minerals deal (March 2025)** — $7M. No deal happened; market resolved YES after a large UMA holder cast ~5M UMA across three wallets (~25% of votes). Polymarket confirmed the result was "against our clarification" but refused refunds. **First public governance-attack allegation.**
- **Barron Trump memecoin (late 2024/early 2025)** — Polymarket *administratively overrode* a UMA resolution. Precedent that the decentralized backstop is not absolute.
- **TikTok ban before May (Jan 2025)** — ~$120M, contested NO; no confirmed attack but heavy community dispute.
- **Sports + weather** — UMA reports <1.5% dispute rate, >98% undisputed.

**Kalshi internal resolution (CFTC-regulated)**. Rulebook 5.10/5.11 amended August 22, 2025: a Member may request review within 15 minutes of an erroneous trade with a $10,000 review fee (refunded only on "Platform malfunction"). Risk here is **regulatory**, not oracle-manipulation: CFTC dismissed its appeal May 5, 2025; state injunctions in Nevada (reversed), New Jersey (won), Maryland (lost), Massachusetts (Jan 2026 injunction on sports), Arizona (stayed). Model Kalshi risk as `jurisdiction × contract-category`, not with the same OO-based scorer.

**Hyperliquid HIP-4 status (April 18, 2026)**. HIP-4 proposed Sep 16, 2025 (co-authored by John Wang / Head of Crypto at Kalshi plus Felix, Asula Labs, Framework); Hyperliquid officially announced Feb 2, 2026; **currently testnet only**. Kalshi partnership publicly confirmed March 2026. Mechanics publicly known: fully collateralized USDH binary contracts, no leverage, ~15-minute single-price clearing auction, 500K-1M HYPE builder stake, "authorized oracle" model with optional challenge window, asymmetric oracle settlement (upward fast, downward ~50 min). Mainnet date **speculative** — design for a testnet→mainnet config flip.

**Lessons from DeFi oracle failures for feature engineering**. Every major failure used either a single thin venue as price source (Mango Markets Oct 2022 / $110M; Compound DAI Nov 2020 / $88M; Synthetix sKRW 2019) or a subjective resolution criterion (Zelenskyy suit). Both map directly to features below.

### 2.2 Data pipelines

The Graph hosted service is deprecated — **UMA subgraphs now live on Goldsky**. Primary endpoints:

| Subgraph | Goldsky URL |
|---|---|
| Polygon Managed OOV2 (Polymarket) | `api.goldsky.com/.../polygon-managed-optimistic-oracle-v2/1.0.5/gn` |
| Polygon OOV2 | `.../polygon-optimistic-oracle-v2/1.1.0/gn` |
| Mainnet OOV2 + DVM Voting V2 | `.../mainnet-optimistic-oracle-v2/latest/gn`, `.../mainnet-voting-v2/0.1.1/gn` |

Source of truth: `github.com/UMAprotocol/subgraphs`. Queryable entities include `Request, ProposePrice, DisputePrice, Settle, PriceIdentifier`; on the DVM side `PriceRequest, CommittedVote, RevealedVote, VoterGroup`.

Polymarket APIs: **Gamma** (`gamma-api.polymarket.com`, market metadata, no auth), **CLOB** (`clob.polymarket.com`, orderbook), **Data** (`data-api.polymarket.com`, positions). Python SDKs: `github.com/Polymarket/py-clob-client` and the `agents` repo. On-chain: `UmaCtfAdapter` on Polygon emits `QuestionInitialized/Resolved/Reset/EmergencyResolved`. Free RPC: Alchemy 300M CUs/mo is enough for historical backtesting.

Whale tracking: Arkham free gives labels; Nansen Standard ~$150/mo. Free alternatives: Zerion, Zapper, DeBank, community Dune dashboards (`polymarket-whales`). Subgraph lag is 30-60s on Polygon — for intra-minute trading, fall back to direct `web3.py` event filters on a paid RPC.

### 2.3 Feature set (v1) and DuckDB schema

| Feature | Signal |
|---|---|
| `resolution_type` | UMA_OOV2 vs UMA_MOOV2 vs KALSHI vs HIP4_AUTH vs CHAINLINK — base rate |
| `liveness_seconds`, `bond_usdc` | shorter/smaller = thinner review |
| `subjectivity_score ∈ [0,1]` | LLM-scored ambiguity of resolution text |
| `resolution_source_concreteness` | named authoritative source vs. "consensus of reporting" |
| `hhi_uma_top10` | voter Herfindahl (from voting-v2 subgraph) |
| `market_volume_usd`, `max_single_wallet_position_usd` | attack incentive and whale flag |
| `price_distance_from_extremes` | `min(p, 1-p)` — contested-ness |
| `hours_to_resolution`, `price_moved_pct_24h` | last-minute surprise signal |
| `proposer_whitelisted` | MOOV2 flag |
| `similar_market_dispute_rate` | Bayesian beta-binomial shrinkage vs category base |
| `llm_grok_disagrees_with_market` | AI adjudicator disagreement |

Storage lives in DuckDB alongside K-Fish trades — same file, separate schema:

```sql
CREATE TABLE oracle_events (
  event_id VARCHAR PRIMARY KEY, market_id VARCHAR REFERENCES markets(market_id),
  event_type VARCHAR, tx_hash VARCHAR, block_number BIGINT, ts TIMESTAMP,
  actor VARCHAR, price_proposed DOUBLE, price_final DOUBLE,
  bond_usdc DOUBLE, round_id INTEGER);

CREATE TABLE risk_scores (
  market_id VARCHAR, ts TIMESTAMP, risk_mean DOUBLE,
  risk_lo_95 DOUBLE, risk_hi_95 DOUBLE, model_version VARCHAR,
  feature_vec JSON, PRIMARY KEY (market_id, ts, model_version));
```

### 2.4 The modeling core — Bayesian logistic with category shrinkage

Because the "legitimate dispute" sample is tiny (<50 high-profile cases since 2023), frequentist logistic overfits. Use NumPyro NUTS with strong priors:

```python
def bayes_logit(X, y):
    def model(X, y=None):
        beta = numpyro.sample("beta", dist.Normal(jnp.zeros(X.shape[1]), 1.0))
        alpha = numpyro.sample("alpha", dist.Normal(-3.0, 2.0))  # low base rate
        logits = alpha + X @ beta
        numpyro.sample("obs", dist.Bernoulli(logits=logits), obs=y)
    mcmc = MCMC(NUTS(model), num_warmup=1000, num_samples=2000)
    mcmc.run(jax.random.PRNGKey(0), X=jnp.array(X), y=jnp.array(y))
    return mcmc.get_samples()
```

Per-category dispute rates use a beta-binomial with `Beta(1, 50)` weak prior (~2% base rate):

```python
class CategoryDisputeRate:
    def __init__(self, prior_a=1.0, prior_b=50.0):
        self.a, self.b = prior_a, prior_b
    def update(self, disputed): self.a += disputed; self.b += not disputed
    def posterior(self, cred=0.95):
        lo, hi = beta.ppf([(1-cred)/2, 1-(1-cred)/2], self.a, self.b)
        return self.a/(self.a+self.b), lo, hi
```

**Subjectivity scoring via Claude** — pass the raw resolution text with a rubric returning `{"score": float, "reasoning": str, "named_source": bool}`. Regression test: Zelenskyy suit text ("consensus of credible reporting" + binary judgment call) should score ≥0.75. Run Claude + GPT + Gemini in parallel and use inter-model agreement as a meta-feature.

**Backtest methodology**: for every resolved Polymarket market since Q3 2024, pull the market price at `resolved_at - 24h`, define *surprise* as `|outcome - price_t-24h| > 0.25`, define *dispute* from the subgraph `DisputePrice` event. Out-of-time split: train on 2024, validate on 2025 H1, test on 2025 H2 + 2026 Q1. Report Brier, AUC, ECE with isotonic post-calibration.

### 2.5 Refusal gate — the K-Fish integration

```python
HARD_REFUSE_THRESHOLD = 0.35
POSITION_CAP_PCT = {0.15: 1.00, 0.25: 0.30, 0.35: 0.10}

def gate(signal, risk, capital):
    if risk.mean > HARD_REFUSE_THRESHOLD:
        log_refusal(signal.market_id, risk); return None
    adj = signal.edge * (1 - risk.mean)
    cap = next(v for k,v in sorted(POSITION_CAP_PCT.items()) if risk.mean <= k)
    signal.size = min(signal.size, capital * cap * (adj/signal.edge))
    return signal
```

Thresholds calibrated from backtest: Zelenskyy-class markets at t-48h score 0.45-0.55, Ukraine rare-earth ~0.40, both hard-refuse. Most sports and crypto-threshold markets score <0.05.

Because the loss distribution is fat-tailed (one Zelenskyy wipes out many small wins), use **CVaR-style refusal**: refuse when `P(loss > 50%) > 5%`, not just when expected value is marginal.

### 2.6 Ship as open-source

Package as `polymarket-oracle-risk` on PyPI (MIT). Streamlit Community Cloud hosts a public dashboard free; GitHub Actions runs a nightly backtest that updates a read-only DuckDB snapshot in the repo. This doubles as research credibility for PhD-adjacent reputation.

Streamlit UI: ranked table of active markets by `edge × (1 − risk)` with `ProgressColumn` for risk, scatter of risk vs edge sized by volume, and a drill-down showing the feature vector JSON plus all oracle events for the selected market.

### 2.7 Honest limitations

Dispute sample is <50 high-profile events → wide posteriors, directional not precise. Managed Proposer (Aug 2025) is a regime change — re-weight or segment training data. HIP-4 has zero history beyond testnet — ship an `HIP4Scorer` stub that uses oracle-type-only priors until mainnet history accumulates; flag all HIP-4 scores as "low confidence." UMA top-10 voter concentration numbers are community-alleged, not officially published — re-derive from the voting-v2 subgraph before using as a feature. Goldsky endpoint IDs rotate; pin versions in config with a retry path.

---

## Part 3 — Korean HIP-4 interface: build the Telegram bot first

### 3.1 The April 2026 Korean UX reality

**Hyperliquid native Korean support is already live** — hyperliquid.xyz ships 한국어 in the in-app picker. Translation alone is no longer a wedge. The real gaps are onboarding, the KRW→USDC rail, and the social/group-chat UX that KR crypto traders actually use.

**PvP.trade** (~50K MAU, ~$7.2M lifetime revenue via builder codes) proves the channel: agent-wallet-based Telegram bot with `/long`, `/short`, `/positions`, clan leaderboards, multi-chain deposits. Big in Korea because (a) crypto lives in Telegram/KakaoTalk group chats, (b) 자랑 (brag) culture makes PnL sharing sticky, (c) no VPN ritual vs. the web UI, (d) Solana-direct deposits matter for Upbit→HL arbitrage.

**CEX reality**: Upbit and Bithumb cannot list HIP-4 outcome contracts — VAUPA (in force July 19, 2024) effectively bars futures/perps/event contracts from KRW-market CEXs. Users route KRW → BTC/XRP on Upbit → external wallet → USDC on Arbitrum/Solana → Hyperliquid. **That friction is the single biggest Korean UX gap; whoever shortens it wins.**

**Regulatory flags (not legal advice)**: VAUPA governs VASPs (custody, 80% cold-wallet, KYC). A strictly non-custodial bot that never holds funds is defensible as not-a-VASP, but FSC has claimed extraterritorial reach. Travel Rule threshold is ₩1M (~$680-740); FSC plans H1 2026 expansion to sub-₩1M plus blocks on "high-risk offshore exchanges" — **watch this weekly through Q2 2026**. If you ever add a KRW on/off-ramp, you instantly become a VASP. Budget ₩1-3M for a review by Bae Kim & Lee, Lee & Ko, Kim & Chang, or smaller firms (Decent Law, Yulchon) before any public KR-targeted marketing.

### 3.2 Why Telegram first, not KakaoTalk or Next.js

**Do not try to ship a KakaoTalk-native trading bot as a solo dev.** Kakao i OpenBuilder supports webhook chatbots, but Kakao's ToS explicitly prohibits promoting unregulated financial services, verified business channels require 사업자등록, and unofficial bot frameworks (LOCO/IRIS) risk permanent bans. The sane Kakao play is **OAuth social login** in a web companion page via Privy's custom-OAuth provider (Kakao exposes standard OIDC) — same user expectation, zero custody or ToS risk.

**Telegram bot MVP ships in 3-4 weeks; a Next.js web app takes 10-14 weeks.** Both earn via the same builder-code mechanism. The bot is lower regulatory surface area (niche, opt-in) and compounds via clan virality. PhD constraint makes the choice: **Telegram bot first, Next.js read-only dashboard second (Month 4+), no KakaoTalk bot ever**.

### 3.3 Stack — recommended picks

| Layer | Choice | Why |
|---|---|---|
| Telegram | **aiogram 3.27+** | Async/FSM/router, handles WS price streams + user commands concurrently |
| HL client | **hyperliquid-python-sdk** (official) | Has `approve_builder_fee`, order-with-builder, agent-wallet support |
| Web (Mini App sign flow) | **FastAPI + vanilla + viem CDN** or tiny Vite build | User signs ApproveBuilderFee/ApproveAgent in their own wallet via WalletConnect |
| Runtime | **Fly.io NRT (Tokyo) region** | ~30ms to Seoul, Postgres add-on, ~$5-20/mo |
| DB/state | **Postgres (Fly) + Redis** | FSM sessions, rate-limit, pending-signature nonces |
| Signing | **eth-account 0.13+** | Agent-wallet key generation |
| Eventual web | **Next.js 15 + next-intl 3 + shadcn/ui + Tailwind v4** | App Router; next-intl beats next-i18next for RSC |
| Wallet for web | **Privy** with Kakao OAuth custom provider | Embedded wallets + KR-friendly social login |

### 3.4 Wallet custody model — pick B, never C

**A. User signs externally via Mini App / WalletConnect** (lowest regulatory exposure; you never touch keys — use for deposits, ApproveBuilderFee, withdrawals).
**B. Agent wallet** (medium exposure; user signs one-time `ApproveAgent` authorizing a bot-held key that cannot withdraw; funds stay on user's main HL account; same model as pvp.trade). **Use for the routine trading loop.**
**C. Bot-managed main wallet** (high exposure; custody = VASP territory in Korea). **Never.**

### 3.5 Revenue math

Builder fee max is 10 bps perps / 100 bps spot; conservative pick is **5 bps perps**.

- 100 users × $100K/mo volume × 5 bps = **$5,000/mo**
- 500 × $100K/mo × 5 bps = **$25,000/mo**
- 2000 × $200K/mo × 5 bps = **$200,000/mo**

Pvp.trade's $7.2M lifetime and Phantom's ~$100K/day are upper-bound reference points, not projections. Realistic 6-month target for a Korean-niched bot: 600 users / $8K MRR. Optimistic (HIP-4 mainnet launches in window + one KR KOL clan adopts): 2000 / $30K.

### 3.6 Code sketches

**ApproveBuilderFee in the user's browser** via `@nktkas/hyperliquid`:

```typescript
import { ExchangeClient, HttpTransport } from '@nktkas/hyperliquid';
import { createWalletClient, custom } from 'viem';
import { arbitrum } from 'viem/chains';

const walletClient = createWalletClient({ chain: arbitrum, transport: custom(window.ethereum!) });
const hl = new ExchangeClient({ transport: new HttpTransport(), wallet: walletClient });

await hl.approveBuilderFee({ builder: '0xYourBuilderAddress', maxFeeRate: '0.05%' });
await hl.approveAgent({ agentAddress: agentPub, agentName: 'HypeKR' });
```

**Korean `/롱` command** (aiogram 3):

```python
@router.message(Command(commands=["long", "롱"]))
async def cmd_long(msg: Message):
    _, coin, size, *px = msg.text.split()
    ex = await get_exchange_for_user(msg.from_user.id)  # agent-wallet Exchange
    price = float(px[0]) if px else (await ex.info.all_mids())[coin.upper()] * 1.005
    res = ex.order(name=coin.upper(), is_buy=True, sz=float(size),
                   limit_px=round(price,2),
                   order_type={"limit":{"tif":"Ioc"}}, reduce_only=False,
                   builder={"b":"0xYourBuilderAddress", "f":50})  # 5 bps
    await msg.answer("✅ 롱 체결" if res["status"]=="ok" else f"❌ {res}")
```

**HIP-4 outcome buy**:

```python
@router.message(Command(commands=["yes", "예"]))
async def cmd_yes(msg: Message):
    _, market, usd, maxp = msg.text.split()
    ex = await get_exchange_for_user(msg.from_user.id)
    asset_idx = await resolve_outcome_asset(market)
    sz = float(usd) / float(maxp)
    ex.order(name=market, is_buy=True, sz=sz, limit_px=float(maxp),
             order_type={"limit":{"tif":"Gtc"}}, reduce_only=False,
             builder={"b":"0xYourBuilderAddress","f":50})
    await msg.answer(f"🟢 YES 매수: {market} {sz:.2f} 계약 @ {maxp}")
```

### 3.7 Architecture

```
 ┌──────────── Korean user (Telegram chat) ───────────┐
 │   /롱 BTC 0.1              Mini App WebView        │
 │                            (WalletConnect signing)  │
 └──────┬─────────────────────────────┬────────────────┘
        │                             │
        ▼                             ▼
 ┌──────────────────┐       ┌──────────────────────┐
 │ aiogram 3.27     │       │ FastAPI /sign        │
 │ handlers (Fly    │◀─────▶│ endpoint (same app)  │
 │ NRT region)      │       └──────────────────────┘
 └────────┬─────────┘
          │
          ▼
 ┌──────────────────────────────────────────┐
 │ Redis FSM + Postgres users/trades/clans  │
 │ + APScheduler (settlement pings in KST)  │
 └────────┬─────────────────────────────────┘
          ▼
 ┌─────────────────────────────────────┐
 │ hyperliquid-python-sdk              │
 │  info.user_state / exchange.order   │
 │  builder={"b":addr, "f":50}          │
 └────────┬────────────────────────────┘
          ▼ signed JSON
 ┌──────────────────────────────┐
 │ api.hyperliquid.xyz/exchange │ (HyperCore: perps, spot, HIP-4)
 └──────────────────────────────┘
```

### 3.8 12-week plan

Weeks 1-2 foundations (Bot registered, Fly NRT deploy, testnet client, `/start /help /deposit`). Weeks 3-4 onboarding (per-user agent wallet with KMS encryption, Mini App WalletConnect signing, deposit polling). Weeks 5-6 trading core (`/long /short /close /positions /balance` with KR aliases). Weeks 7-8 HIP-4 testnet (`/yes /no`, market browser, KST settlement reminders via APScheduler). Weeks 9-10 social (group mode, leaderboards, referral codes). Week 11 compliance (KR ToS via Lawform ₩50-150K + attorney review ₩1-3M, PIPA-compliant privacy policy, 19+ age gate, disclaimer "본 서비스는 투자 조언을 제공하지 않으며..."). Week 12 launch (3 trusted KR TG groups → Disquiet, r/KoreanCrypto, DC Inside 코인갤, KakaoTalk OpenChat, X/@ksk5429).

### 3.9 Monitoring, positioning, failure modes

Sentry free tier, PostHog self-hosted on Fly or free cloud tier (1M events/mo), Plausible ($9/mo) or Umami self-hosted for the Mini App site.

**Positioning**: don't clone pvp.trade — add the "한국어 퍼스트 + HIP-4 prediction markets" wedge. Display KRW equivalents (Upbit price reference), push YES/NO UX with KST settlement reminders, build the "Polymarket-of-Korea via Hyperliquid" framing.

**Biggest failure modes**: FSC crackdown on offshore-DEX-facing tools mid-2026 (watch H1 Travel Rule finalization); HIP-4 mainnet slips past 2026 (hedge: bot works on perps/spot regardless); pvp.trade ships KR localization first (hedge: lean into Upbit parity view, KR KOL clan tooling, HIP-4 event-contract UX they won't prioritize).

---

## Part 4 — Machine calibration: the recipe that out-scores humans

### 4.1 The human benchmark and why it's hard to beat

Tetlock's Good Judgment Project established the ceiling: superforecasters hit **Brier 0.20-0.25** on IARPA ACE; on cleaner question sets the elite panel reaches **Brier 0.096** (ForecastBench 2025) to **0.023** (Lu 2025). Their traits map directly to engineering choices: actively open-minded thinking (9-agent red team), dragonfly eye (9 agents × 5 samples = 45 views), reference-class forecasting (DuckDB base-rate lookup), granular forecasts in 1% increments (no 0.05 rounding), frequent small updates (rolling isotonic refit).

Humans dominate machines on elite panels today (0.02 vs 0.13) because they bring domain expertise and meta-rationality about novel regimes. Machines dominate on scale, consistency, and proper-score discipline. **The right K-Fish framing is a machine superforecaster panel trading against human market participants — complementary, not replacement.**

### 4.2 Why machines can out-calibrate in principle

Consistent Bayes without fatigue; thousands of reference classes in parallel; perfect recall of priors; emotion-invariant; explicit uncertainty machinery (Bayesian NNs, ensembles, conformal prediction); continuous recalibration; hard-to-fool proper scoring rules.

**Caveats that constrain the promise**: distribution shift breaks conformal exchangeability; RLHF flattens conditional probabilities (Kadavath 2022; Tian 2023) so Claude-class models are overconfident in token logprobs but better calibrated in *verbalized* probabilities; benchmark leakage is rampant (Paleka 2025: 71% of date-filtered backtests leak); reflexivity means once many LLM forecasters hit Polymarket, $p_\ell$ and $p_m$ de-correlate and the edge shrinks; outcome-based RL without regularization pushes probabilities to 0/1 (Turtel 2025).

### 4.3 The 2024-2026 literature — what's actually established

| Paper | Method | Result |
|---|---|---|
| **Halawi et al. 2024** (arXiv:2402.18563, NeurIPS) | Retrieval-augmented GPT-4 + trimmed-mean ensemble on 914 post-cutoff questions | System Brier **0.179** vs human crowd 0.149; **hybrid LLM+crowd beats either alone** |
| **Schoenegger et al. 2024** (Sci Adv 10) | 12-model median ensemble vs 925 humans, 31 binary Qs | Ensemble Brier **0.20**, statistically indistinguishable from crowd; simple averaging of human+LLM beats either |
| **Tian et al. 2023** (EMNLP, arXiv:2305.14975) | Verbalized confidence + multi-hypothesis | ECE reduced ~50% vs raw logprobs on RLHF models |
| **Turtel et al. 2025** (arXiv:2505.17989, TMLR) | ReMax RL on DeepSeek-R1-Distill-Qwen-14B, 10k Polymarket Qs | Brier **0.190**, ECE **0.062**, beats o1; simulated Polymarket $52 vs $39 |
| **ForecastBench** (Karger 2024/25, arXiv:2409.19839) | 500-Q biweekly rounds | Superforecaster median 0.096; best LLM (o3) 0.135; linear extrapolation → parity ~**Nov 2026** |
| **Alur et al. 2025 AIA** (arXiv:2511.07678) | Agentic search + supervisor + post-hoc Platt/extremization | Statistically indistinguishable from superforecasters |
| **Paleka et al. 2025** (leakage audit) | — | 71% of retrospective backtests leak; 41% directly reveal answer |

Three takeaways: (i) frontier LLMs in 2026 are at crowd level but below the superforecaster panel; (ii) **aggregation is the biggest single lever**; (iii) **hybrid LLM+market > either alone**.

### 4.4 The 2-to-4-week K-Fish recipe

**Minimum stack (everything else is optional)**: `scikit-learn`, `venn-abers`, `crepes` (or `MAPIE`), `properscoring`, `duckdb`, `streamlit`, `anthropic`. Skip `laplace-torch`, `bayesian-torch`, `torchuncertainty`, `forecast-skill` until you add a neural layer.

**Week 1 — baseline**. Run the Brier decomposition SQL from Part 1.3 per category. Compute ECE in Python (15 bins). Render a reliability diagram in Streamlit. Goal: honest starting number for `ECE_baseline` and `Brier_baseline`.

**Week 2 — post-hoc calibration + MC swarm**. Refit isotonic + Venn-Abers per category every 50 trades:

```python
def refit_calibrators(conn, min_trades=50):
    df = conn.execute("""SELECT category, p_raw, outcome FROM trades
                         WHERE outcome IS NOT NULL ORDER BY ts DESC LIMIT 2000""").df()
    cals = {}
    for cat, sub in df.groupby('category'):
        if len(sub) < min_trades: continue
        iso = IsotonicRegression(out_of_bounds='clip', y_min=1e-3, y_max=1-1e-3)
        iso.fit(sub.p_raw.values, sub.outcome.values)
        va = VennAbers(); va.fit(sub.p_raw.values.reshape(-1,1), sub.outcome.values)
        cals[cat] = {'iso': iso, 'va': va, 'n': len(sub)}
    joblib.dump(cals, '/tmp/kfish_cals.pkl')
    return cals
```

9-agent MC wrapper around Claude 4.7 with verbalized-probability prompt, temperature 0.5-0.9 per agent, 5 samples each. Aggregate via **median** (robust; Schoenegger 2024), report `std` as epistemic uncertainty for sizing:

```python
CALIB_PROMPT = """You are a disciplined superforecaster. For the question below:
1. State the reference class and base rate.
2. List 2-3 factors pushing above/below the base rate.
3. Give a final probability as a number in [0.01, 0.99] on the line:
   FINAL_PROBABILITY: 0.xx
Do NOT round to multiples of 0.05. Avoid the 0.5 attractor."""
```

**Week 3 — Mondrian conformal + shrinkage ensemble**. `crepes.WrapClassifier` with `MondrianCategorizer` by market category. Operational rule: **only trade when the conformal set is a singleton at α=0.10**. When set is `{0,1}`, skip.

Per-category shrinkage with empirical-Bayes regularization toward a global α:

$$p_\text{final} = \alpha_c \cdot p_\ell + (1-\alpha_c) \cdot p_m, \quad \alpha_c^{\text{shrunk}} = w_c \alpha_c + (1-w_c)\alpha_\text{global}, \quad w_c = \frac{n_c}{n_c+k}$$

with $k \approx 50$ by CV. Expect $\alpha \in [0.2, 0.5]$ on liquid markets, $[0.5, 0.8]$ on illiquid/novel. If $\alpha \to 1$ on holdout, that's diagnostic of a genuinely inefficient niche — precisely the alpha K-Fish is hunting.

**Week 4 — drift monitoring**. Weekly Streamlit dashboard with reliability diagram, ECE trend, per-category Brier, exchangeability martingale (flag when Ville's inequality crosses $p < 0.01$). Alert thresholds: `ECE_30d > 0.05` → refit; `Brier_7d > Brier_90d × 1.3` → investigate category; conformal empirical-coverage drift > 5pp → refit CP.

### 4.5 Superforecasting principles → mechanisation

| Principle | K-Fish mechanism |
|---|---|
| Triage | Conformal singleton filter at α=0.10 |
| Break into sub-questions | 9 agents with specialized prompts |
| Inside vs outside view | Shrinkage $p_\text{final}=\alpha p_\ell + (1-\alpha)p_m$ |
| Update incrementally | Rolling isotonic refit |
| Dragonfly eye | 9 agents × 5 samples |
| Distinguish levels of doubt | Verbalized probs + Venn-Abers intervals |
| Know your biases | ECE dashboard + acquiescence regularization |
| Balance over/underconfidence | Temperature scaling + log-loss α tuning |
| Look for errors retrospectively | Brier decomposition into reliability / resolution / uncertainty |

**Reference-class forecasting is the single highest-leverage technique.** Halawi's retrieval jump from zero-shot GPT-4 (≈0.25) to 0.179 is mostly the outside view arriving. Implement the DuckDB historical-market base-rate lookup before any fancy method.

**Bias-variance in calibration**: isotonic is low-bias/high-variance (needs 500+); Platt is high-bias/low-variance (safe under scarcity); Beta sits between. Never run isotonic with `n < 150` per category.

**Success criteria (pre-register these)**: Brier ≤ 0.18 on 100+ post-cutoff binary markets, ECE ≤ 0.05, conformal empirical coverage within ±2pp of nominal. That puts K-Fish within a factor of 2 of elite human panels and at parity with published LLM forecasting systems — which, empirically (Turtel 2025), is enough to extract positive EV from inefficient prediction markets.

**When calibration breaks**: regime shifts (COVID, wars, elections) violate exchangeability — use ACI (Gibbs & Candès 2021) or widen CP intervals, maintain a `regime_flag` table in DuckDB. Reflexivity: monitor `p_ℓ - p_m` residuals for autocorrelation; if it disappears, the edge is gone. Prompt-injection attacks via headlines → source diversity + 9-agent outlier filter + cap position size when swarm `std > 0.15`. Data contamination: treat all pre-2026 Brier numbers as optimistic upper bounds.

---

## The 90-day integrated roadmap

**Weeks 1-2 — foundation (shared across all three projects)**
WSL2 Ubuntu 24.04 + uv + Python 3.12 + Docker Desktop setup. Git clone `kfish` repo, DuckDB schema migrated, self-hosted Langfuse running via docker compose. Provision Oracle Cloud A1 (or Hetzner CX32 fallback). Register Telegram bot via @BotFather. Register Polymarket Gamma API access, Goldsky subgraph endpoint, Naver Developer app (**verify News API quota this week**). Choose one: commit to Fly.io Tokyo for the bot or Vercel for any future web surface.

**Weeks 3-4 — scraper + oracle ingest**
Korean news scraper for Naver + Yonhap + Hankyoreh + Blockmedia + Daily NK online; Kiwi tokenization; SimHash + BGE-m3 dedup; Papago + Claude translation router with Langfuse telemetry. Prefect 3 scheduler running every 10 min. UMA subgraph ingest and Polymarket Gamma ingest into DuckDB `oracle_events` and `markets` tables. Brier decomposition SQL runs nightly via dbt. **By end of Week 4: you have logged news + oracle events + K-Fish decisions in one DuckDB file.**

**Weeks 5-6 — calibration v1 + bot onboarding**
Run Brier/ECE baseline on whatever K-Fish trades already exist. Fit global isotonic + Venn-Abers. 9-agent MC wrapper with verbalized-probability prompt. Telegram bot deployed on Fly.io NRT with `/start /help /deposit`, per-user agent wallet with KMS-encrypted keys, Mini App page with ApproveBuilderFee + ApproveAgent flow.

**Weeks 7-8 — bot trading core + oracle analyzer v1**
Bot `/long /short /close /positions` with Korean aliases, testnet first then mainnet with 5bps builder code. Oracle analyzer v1: Bayesian logistic on features, subjectivity scoring via Claude, historical backtest on 2024-2025 data, refusal-gate integration into K-Fish. Streamlit dashboard live.

**Weeks 9-10 — HIP-4 testnet + Mondrian conformal + bot social**
Add `/yes /no /predict` HIP-4 handlers on testnet. Mondrian conformal via crepes per category, shrinkage ensemble with market price tuned per-category with empirical-Bayes shrinkage. Bot group-chat mode with clan leaderboards.

**Weeks 11-12 — calibration drift monitor + compliance + publish**
Streamlit drift dashboard with ECE alerts and exchangeability martingale. Smoke-test K-Fish calibration on 100+ post-Claude-cutoff resolved markets; report Brier/ECE/coverage; iterate if ECE > 0.05. Korean ToS + PIPA privacy policy via Lawform + attorney review (₩1-3M budget). `polymarket-oracle-risk` published to PyPI; Streamlit Community Cloud dashboard live. Bot soft launch to 2-3 trusted KR TG groups.

**Week 13 — public launch**
Bot launch: Disquiet, r/CryptoCurrency/r/Korea, DC Inside 코인갤, KakaoTalk OpenChat, X @ksk5429. Write-up of oracle analyzer methodology posted to the K-Fish blog/GitHub. First calibration-drift report published.

### Budget summary

| Line | Monthly |
|---|---|
| VPS (Oracle A1 free or Hetzner CX32) | $0-10 |
| Fly.io for bot (Tokyo, Postgres, Redis) | $10-20 |
| Claude API (all three projects combined) | $100-250 |
| Papago (bulk Korean translation) | $200-250 (verify NCP tier) |
| RPC + subgraph + Gamma (all free tiers) | $0 |
| Backups (B2 + R2) | $2-3 |
| Sentry + PostHog + Plausible + healthchecks.io | $0-10 |
| Domain + Cloudflare | $2 |
| **Total realistic** | **~$315-545/mo** |

One-time: attorney review ₩1-3M (~$750-2200), Korean ToS template ₩50-150K (~$40-120), Lawform subscription if iterating. At $10-100K bankroll, monthly infra is 0.3-5%.

### Time-commitment honesty

At 15 h/week (typical PhD-adjacent), 13 weeks = 195 hours. That covers: scraper + oracle ingest + DuckDB spine + calibration v1 + Telegram bot MVP + oracle analyzer v1 published. **Not covered** without sacrificing PhD: Next.js companion web app, KakaoTalk bot (don't build anyway), HIP-4 mainnet polish if launch slips to H2 2026. The Next.js dashboard belongs in Month 4-6 after the bot has users and the calibration spine is stable.

### Regulatory watch list (Korea)

Monitor monthly: FSC announcements on Travel Rule sub-₩1M expansion (expected H1 2026 finalization); offshore exchange blocks; any enforcement against non-custodial crypto tools operated from Korea. If FSC moves, pause KR-targeted marketing, keep the bot running (niche, non-custodial), consult the crypto-specialist firm before adjusting. Never add a KRW rail without becoming a VASP first.

### What would make me wrong

(1) Naver News Open API is silently deprecated — fallback RSS covers ~70% of signal, rebuild on that. (2) HIP-4 mainnet slips to 2027 — bot still earns on perps/spot; oracle analyzer still publishes. (3) pvp.trade ships Korean before you do — lean harder into the Upbit-parity view, KST settlement reminders, HIP-4 event UX, and KR KOL clan tooling they won't prioritize. (4) Polymarket/UMA governance-attack rate drops sharply under Managed Proposer — great for the ecosystem, but your analyzer becomes "safety green-light" rather than "refusal gate" and still has value as a scoring input. (5) Market reflexivity kills the LLM edge as more bots arrive — the calibration spine still compounds as a PhD-adjacent research asset, and the Korean bot's builder-fee revenue is independent of whether K-Fish itself is profitable.

The single most important conviction: **build the DuckDB calibration spine in Week 1 and refuse to cut corners on it.** Every other project compounds on top of that file. If you ship nothing else in 90 days but a well-calibrated 9-agent swarm with clean oracle-risk scoring, you've built the base case for both the PhD-adjacent research output and any future commercial surface — including ones (Korean bot, Next.js dashboard, PyPI package) that you haven't thought of yet.