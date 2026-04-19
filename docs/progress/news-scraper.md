---
title: Korean news scraper
description: Status + evidence that the news ingestion pipeline is operational
---

# Korean news scraper

**Status as of 2026-04-20: operational on the RSS tier.** 6 sources
landing live Korean news into `kfish.articles` with full Kiwi
tokenization and FTS indexing. Naver News search tier pending
credentials.

## End-to-end evidence

One live run against real sources:

```text
$ uv run --package kfish-core kfish-news --sources rss --max-per-source 10
naver: credentials absent; skipping
rss source=yonhap       -> 10 articles
rss source=yonhap_econ  -> 10 articles
rss source=hankyoreh    -> 10 articles
rss source=blockmedia   -> 10 articles
rss source=dailynk      -> 10 articles
rss source=donga        -> 10 articles
built FTS index kfish.articles
ingest_news: fetched=60 dupes=4 inserted=56 tokenized=0 translated=0 duration=13.5s
```

60 articles fetched, 4 SimHash-64 near-duplicates dropped (≈7%), 56
inserted, FTS index auto-built. 13.5 s end-to-end.

### Real headlines that landed

```text
[yonhap] 고가 작품 수차례 거래한 미술상…법원 "사업소득 과세 대상"
[yonhap] [오늘의 국회일정](20일·월)
[yonhap] [2026 와이팜 엑스포] ① 경기·강원·제주
[dailynk] 실력 미달 학생은 탄광행?…기대 이하 성적에 강력 경고
[dailynk] [북한읽기] 아프리카돼지열병 재확산, 어떻게 대응해야 하나
```

### Kiwi tokenization on the same articles

```text
title: 실력 미달 학생은 탄광행?…기대 이하 성적에 강력 경고
tokens: 실력 미달 학생 탄광 행 기대 이하 성적 강력 경고 북한 올해 학기
        고급 중학교 고등학교 선택 과목 전면 도입 가운데 전공 반 시험 대거 낙제 …
```

Particles (은/는/이/가) and endings (했다/한다) are correctly stripped;
nouns + verbs preserved. Exactly what FTS needs.

## What was built

New pipeline module, new CLI, wired into the nightly runner:

| File | Purpose |
|---|---|
| `apps/kfish-core/src/kfish_core/pipeline/ingest_articles.py` | Orchestrator: fetch → dedup → tokenize → upsert → FTS |
| `apps/kfish-core/src/kfish_core/cli/news.py` | Operator CLI (`kfish-news`) |
| `apps/kfish-core/src/kfish_core/pipeline/nightly.py` | News ingestion added to the nightly flow with graceful degradation |
| `runbooks/scraper-ops.md` | Full operator runbook (Naver app registration, scheduling, troubleshooting) in the private kfish repo |

## Design invariants

1. **Every step is independently failure-tolerant.** A dead upstream
   feed, missing credential, or translation-API outage degrades only
   that step. Nightly cron keeps making forward progress.
2. **RSS works without any credentials.** Zero-config path for
   development. Only Naver + Papago + Claude need keys.
3. **Idempotent on re-run.** `ON CONFLICT(url) DO NOTHING` + sliding
   SimHash window means running twice in 20 seconds inserts nothing
   new. Running daily over a month produces clean monotonic growth.
4. **Bit-exact SimHash storage.** DuckDB BIGINT is signed int64;
   SimHash64 is unsigned 0–2⁶⁴−1. A `_to_signed_64` helper reinterprets
   the bit pattern so Hamming distances stay meaningful across
   store/fetch cycles.
5. **FTS index is lazy and once-only.** `ensure_article_fts()` writes a
   `kfish.fts_state` bookkeeping row so repeated runs skip the rebuild.

## Test coverage

- **7 unit tests** in `apps/kfish-core/tests/test_ingest_articles.py`
  covering: happy path, re-run idempotency, SimHash near-duplicate
  dropping, missing-creds graceful skip, per-source failure isolation,
  token-column write, stats summary shape.
- **7 live-network tests** in `apps/kfish-core/tests/test_news_live.py`
  (under `-m network`, skipped by default) that hit all 6 canonical
  RSS feeds plus Naver if creds are set. Green on 2026-04-20.
- **Existing 9 tests** in `apps/kfish-core/tests/test_news.py` for
  SimHash, Kiwi tokenizer, translation router — unchanged.

## Cost profile

Target volume is blueprint §1.8's 1000 articles/day. At that throughput:

| Component | ~₩/day | Notes |
|---|---|---|
| RSS fetches (6 feeds × 24 polls/day) | free | Negligible bandwidth |
| Naver News API (within 25k/day tier) | free | |
| Kiwi tokenization | free | Local |
| Papago bulk (titles only, ~200K chars/day) | ~₩3 000 | Optional |
| Claude high-signal (50 articles/day) | ~₩6 000 | Sonnet + prompt caching |
| **Total** | **~₩9 000 / day** (~$6.50) | under blueprint budget |

## What this unlocks

The scraper operational was the last hard dependency for **M4
(scraper + oracle ingest)** from the 90-day blueprint. With it:

- The DuckDB spine now has a news feed every night on top of markets +
  oracle events — the calibration spine is information-complete per the
  blueprint §1 vision.
- The 9-persona swarm's future `news-aware Fish` can read
  `kfish.articles` via FTS + ASOF-join to markets for reference-class
  lookups.
- The `PROMPT_EXTRACT_SIGNAL_V3` prompt in `agents/prompts.py` now has
  a real corpus to run against; it wasn't fired in production yet, but
  is the next natural extension.

## Related docs

- [Scraper operator runbook](https://github.com/ksk5429/kfish/blob/main/runbooks/scraper-ops.md) — full ops guide in the private repo
- [Architecture: swarm + calibration](../architecture/swarm-calibration.md) — where the news feed plugs into the forecasting loop
- [ADR-0002: DuckDB calibration spine](../architecture/adrs.md#adr-0002) — the warehouse schema that hosts `kfish.articles`
- [Build log](build-log.md) — the commit that shipped this
