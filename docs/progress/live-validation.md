---
title: Live-API validation
description: Polymarket Gamma + UMA Goldsky smoke results
---

# Live-API validation

Confirms that the client code isn't just tested against `httpx.MockTransport`
— it actually reaches real upstreams.

## Polymarket Gamma (public, no auth)

Live snapshot taken 2026-04-19:

| market_id | volume USD | price YES | question (truncated) |
|---|---|---|---|
| 540816…  | 1,559,024 | 0.535 | Russia-Ukraine Ceasefire before GTA VI? |
| 540819…  | 11,067,916 | 0.485 | Will Jesus Christ return before GTA VI? |
| 540820…  | 592,771 | 0.515 | Trump out as President before GTA VI? |
| 540817…  | 698,439 | 0.610 | New Rihanna Album before GTA VI? |
| 540818…  | 726,913 | 0.545 | New Playboi Carti Album before GTA VI? |

Every field parses cleanly; volume + price roundtrip through our Pydantic
models; the `GammaClient` retry survives 500s (verified in unit test).

## UMA OOv2 (public Goldsky)

10 most recent polygon OOv2 requests in the last 14 days, bonds in USDC:

```text
YES_OR_NO_QUERY-17765327   state=Requested   bond=$  500.00
YES_OR_NO_QUERY-17764205   state=Requested   bond=$  250.00
YES_OR_NO_QUERY-17764205   state=Requested   bond=$  250.00
YES_OR_NO_QUERY-17764205   state=Requested   bond=$  250.00
YES_OR_NO_QUERY-17764205   state=Requested   bond=$  250.00
```

## UMA Managed OOv2 (Polymarket's post-UMIP-189 oracle)

3 most recent managed oracle requests (proposer whitelisted):

```text
YES_OR_NO_QUERY-17765665   state=Requested   bond=$  500.00
YES_OR_NO_QUERY-17765664   state=Requested   bond=$  500.00
YES_OR_NO_QUERY-17765664   state=Requested   bond=$  500.00
```

$500 floor matches UMIP-189's `store.computeFinalFee` for standard Polymarket
proposals.

## Gotcha: Goldsky URL rotation

The blueprint warned "Goldsky endpoint IDs rotate" — and they did. Today's
project id is `project_clus2fndawbcc01w31192938i` (new), not the blueprint's
`clcxirpj1c5vx2aton4ou0iin`. Both `config.py` and `.env.example` updated;
the [UMA subgraphs README](https://github.com/UMAprotocol/subgraphs) is the
canonical source for the current URL.

## Gotcha: schema changed too

The blueprint's query referenced an entity called `Request` with a flat
`time` field. The live subgraph calls it `OptimisticPriceRequest` and
tracks per-stage timestamps (`requestTimestamp`, `proposalTimestamp`,
`disputeTimestamp`, `settlementTimestamp`). The client now uses those and
flattens them into our `OracleEvent` shape one-per-stage.

## Reproducing

```bash
# Polymarket — no credentials needed
pytest -m network apps/kfish-core/tests/test_live_apis.py::test_polymarket_gamma_live_returns_active_markets

# UMA OOv2 — no credentials needed
pytest -m network apps/kfish-core/tests/test_live_apis.py::test_uma_goldsky_oov2_live_returns_recent_requests

# Managed OOv2
pytest -m network apps/kfish-core/tests/test_live_apis.py::test_uma_goldsky_moov2_live_returns_polymarket_requests
```

All 4 tests green as of 2026-04-19.
