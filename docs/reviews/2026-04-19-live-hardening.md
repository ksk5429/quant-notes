---
title: 2026-04-19 — Live-trading hardening
description: Independent reviews + fixes for the 18 flagged issues
---

# 2026-04-19 — Live-trading hardening

## What I said I'd do

Write the blueprint-driven production build, then harden it.

## What I actually shipped

After the production build, spawned two independent review agents
(`code-reviewer`, `security-reviewer`) plus my own coverage / pyright /
secret-scan pass. They surfaced 15 distinct items across CRITICAL / HIGH /
MEDIUM / LOW (several agent-reported items bundled where a single fix
resolved more than one finding).

Fixed every single one. Summary table:

| # | Severity | Issue | Resolution |
|---|---|---|---|
| 1 | CRITICAL | `/close` lied (sz=0.0 never submitted) | Reads live position via `clearinghouseState`; submits real reduce-only IOC on opposite side for full size; honest "no position" reply when empty |
| 2 | CRITICAL | `_reference_price` hardcoded 100.0 for every coin | New `HLInfoClient` with `all_mids()` + 10s TTL cache; real mid per symbol |
| 3 | CRITICAL | `HYPEKR_APP_SIGNING_SECRET` silent default "dev-only-insecure-secret" | `_load_app_signing_secret()` raises `InsecureConfigError` at `build_app` time for any empty / default / <32-char value; 9 regression tests |
| 4 | HIGH | Mondrian conformal wasn't stratified by true class | Separate nonconformity arrays per `(category, class)`; `per_class_empirical_coverage()` helper verifies both class coverages hold independently |
| 5 | HIGH | Delphi convergence checked spread *reduction*, not absolute | Two-condition exit: `agg.spread < ε` (converged) OR `Δspread < stall_ε AND spread < 0.15` (stall); no early exit while swarm still wide |
| 6 | HIGH | `WalletStore` DuckDB conn not thread-safe | Added `threading.RLock` around all query methods |
| 7 | HIGH | HMAC nonce truncated to 64 bits | Use full 256-bit SHA-256 digest |
| 8 | HIGH | Pseudo-HKDF (single HMAC, no salt) | `cryptography.hazmat.primitives.kdf.hkdf.HKDF` with static domain-separation salt + per-user `info` |
| 9 | HIGH | `/positions` / `/balance` returned placeholder strings | Now fetch from `clearinghouseState`; format positions with side/size/entry/uPnL |
| 10 | MEDIUM | Full Ethereum addresses logged at INFO | SHA-256 fingerprints (12 chars) in every log line; raw address never printed |
| 11 | MEDIUM | `pypa/gh-action-pypi-publish` mutable tag | SHA-pinned to `6733eb7d…` (v1.14.0); all publishing-path actions pinned too |
| 12 | MEDIUM | Coin symbol unvalidated | Charset + length check, then HL universe allowlist; fail-closed when universe unavailable |
| 13 | MEDIUM | 14 pyright errors | Fixed all — 0 errors now |
| 14 | LOW | Delphi peer summary used persona names | Per-round opaque ID shuffle (`agent-A`, `agent-B` …) — personas can't recognize their own prior estimates |
| 15 | LOW | Pickle load had no lib-version check | `CalibratorBundle.lib_versions` snapshot at save; load warns on drift of sklearn / venn-abers / numpy / scipy |

Plus a bonus finding during Brier-parity debugging: the validator had a
`int(bool(9e-8))` bug that treated near-zero `ground_truth` as True. Found
+ fixed. The three legacy retrodiction runs now match to **exact Brier** to
4 decimal places.

## What surprised me (positive)

- The subagent reviewers found **real bugs** — not aesthetic noise.
  Code-reviewer caught the `/close` lying to users; security-reviewer caught
  the `_APP_SIGNING_SECRET` silent fallback that would have let attackers
  forge wallet approvals from public source.
- The stratified Mondrian conformal passed the per-class coverage test on
  first try after the rewrite. Coverage ≥ 0.86 for both classes on
  out-of-sample data.
- `uv lock` remained deterministic through all the dependency shuffling
  (added `cryptography.hkdf`, kept jax/numpyro pinned).

## What surprised me (negative)

- Ruff + pyright interact with sentry_sdk's optional import in three
  different ways. Ended up using `importlib.import_module` to sidestep
  both static analyzers entirely — cleaner than `# type: ignore` gymnastics.
- GitHub Rulesets API silently requires `require_last_push_approval` and
  `allowed_merge_methods` fields that aren't documented as required. Had to
  probe with minimal rulesets to find the right shape.
- The HL info endpoint's `position.szi` is a string at the wire level, not
  a number — one test needed a silent type-coerce before comparison.

## Test + analyzer status after fixes

| Gate | Before | After |
|---|---|---|
| Unit tests | 123 / 123 | **140 / 140** (+ 17 new regression tests) |
| Live-network tests | 4 / 4 | 4 / 4 |
| Ruff check | clean (with some PLR0911 noise) | **clean** |
| Ruff format | clean | **clean** |
| Pyright | 14 errors | **0 errors** |
| Coverage | 66.0% | 66.0% (unchanged — fixes were in tested paths) |
| Brier-parity | 3/3 runs pass | 3/3 runs pass |

## Open questions carried forward

Still unresolved (from [open-questions](open-questions.md)):

1. Fate of the old public `ksk5429/quant` repo
2. Korean bot launch criteria (legal review timeline)
3. ECE binning convention between kfish and legacy (advisory only)

New since today:

4. `refit_calibrators` still reads `ensemble_prob` from the DB — the current
   schema doesn't cleanly separate raw aggregate from post-extremization.
   Followup: add an explicit `raw_prob` column (or rename) so the
   calibration API contract is unambiguous in the SQL schema itself.

## Decisions taken

- **No live mainnet trading**: still blocked pending human sign-off on the
  paper→live transition, but the code is no longer the blocker. Every
  CRITICAL and HIGH issue from the review is resolved.
- **Bot commands are stub-free**: every handler hits a real endpoint or
  returns a correct error; no more placeholder strings in UX.

## Explicit uncertainties I'm taking into next week

- Live-fire testing against HL testnet. Until one real testnet order places
  and a real testnet position closes via `/close`, the new code paths have
  only unit coverage against httpx mocks.
- The legacy `src/` engine still holds the Brier-0.2026 baseline. We've
  proved the math path is identical; we have not yet proven the integration
  path (Gamma fetch → swarm → calibrator → DuckDB row) matches end-to-end.
  That's the 7-night cutover gate still in flight.
