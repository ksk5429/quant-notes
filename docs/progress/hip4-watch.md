---
title: HIP-4 mainnet watch
description: Monitoring when Hyperliquid's event-contract venue goes live
---

# HIP-4 watch

HIP-4 event contracts are Hyperliquid's answer to Polymarket-style binary
markets on-chain. As of April 2026 they are **testnet-only**; mainnet launch
date is speculative. All hypekr-bot HIP-4 handlers are feature-gated on the
`HL_NETWORK` setting; live orders against HIP-4 fail with a clear error
until the mainnet flag is flipped.

## Current state

| Network | Reachable | HIP-4 symbols | Total symbols |
|---|---|---|---|
| mainnet | ✅ | 0 | 229 |
| testnet | ✅ | 0 | 207 |

Probed 2026-04-19. Neither environment exposes `HIP4_*`-prefixed or
`*-YES` / `*-NO` symbols yet.

!!! note "Why this can't be 'fixed'"
    HIP-4 is a venue-side feature we don't own. The correct mitigation is a
    watchdog that pages us the day mainnet goes live, so we can flip
    `HL_NETWORK=mainnet` and re-test. We maintain that watchdog rather than
    trying to ship pseudo-HIP-4 support.

## Running the watchdog

```bash
python scripts/hip4_watch.py              # polls both networks
python scripts/hip4_watch.py --json       # machine-readable
python scripts/hip4_watch.py --only-mainnet
```

Exit codes:

- `0` — both reachable, no HIP-4 yet
- `1` — connectivity failure (worth investigating)
- `2` — **HIP-4 detected on mainnet**; flip the flag and retest

## Scheduling in production

Add to the A1 box's `nightly.yml` or a separate cron that pings
healthchecks.io:

```yaml
- name: HIP-4 watch
  run: |
    python scripts/hip4_watch.py --json > hip4-status.json
    # exit 2 means HIP-4 live — trip the alert
    if [[ $? == 2 ]]; then
      curl -fsS "$HEALTHCHECKS_HIP4_URL/fail"
    else
      curl -fsS "$HEALTHCHECKS_HIP4_URL"
    fi
```

## What flipping the flag implies

When `HL_NETWORK=mainnet` and the watchdog starts seeing HIP-4 symbols:

1. Re-run `pytest -m hyperliquid` to confirm order-building paths still
   assemble valid payloads.
2. Enable the `/yes` / `/no` / `/predict` handlers on mainnet (they already
   work on testnet).
3. Seed a small fraction of prediction-tile budget per ADR-0008's
   `HIP4Scorer` — flag all HIP-4 scores as low confidence until the
   analyzer accumulates its own dispute history.
4. Update this page.

## Related ADRs

- [ADR-0004 — Per-user agent wallets](../architecture/adrs.md#adr-0004) —
  custody model remains identical on HIP-4.
- [ADR-0008 — Isotonic + Venn-Abers calibration](../architecture/adrs.md#adr-0008)
  — HIP-4 gets its own category in the shrinkage ensemble once there's
  history.
