---
title: Brier parity
description: Evidence that the new calibration path reproduces the legacy engine
---

# Brier parity

**ADR-0002** gates the legacy → new cutover on 7 consecutive nightly runs
where the new `kfish-core` Brier is within 0.002 of legacy.
**First night green on 2026-04-19.**

## First-night evidence

All three archived legacy retrodiction runs, plus the first new-path
computation, agree on Brier to 4 decimal places:

| Run | N | legacy Brier | new Brier | Δ | gate |
|---|---|---|---|---|---|
| `retro_v2_20260412_221752` | 30 | 0.2029 | 0.2029 | 0.0000 | ✅ |
| `retro_v2_20260413_062123` | 200 | 0.2026 | 0.2026 | 0.0000 | ✅ |
| `retro_v2_20260413_150229` | 100 | 0.1799 | 0.1799 | 0.0000 | ✅ |

Extremized Brier and ECE also track closely; ECE diverges only because
legacy uses `netcal`'s binning and the new path uses an equivalent numpy
reimplementation. ECE is **advisory** per ADR-0002 — Brier is the gate.

```text
[PARITY OK] 3 run(s) passed the ADR-0002 Brier gate.
```

## Murphy decomposition of the 200-market baseline

For the N=200 run (the de-facto production baseline):

| Component | Value | Meaning |
|---|---|---|
| reliability | 0.0359 | calibration gap — small is good |
| resolution | 0.0734 | how much the predictor separates classes — large is good |
| uncertainty | 0.2400 | data-bound, $\bar o (1 - \bar o)$ |
| **Brier** | 0.2025 | reliability − resolution + uncertainty |
| skill score | 0.1562 | 1 − Brier / uncertainty (vs climatology) |

## Reproducing

```bash
python scripts/brier_parity.py --all
# or for a specific run:
python scripts/brier_parity.py --input data/retrodiction/retro_v2_20260413_150229.json
# gate on ECE too (stricter, not required by ADR-0002):
python scripts/brier_parity.py --all --strict
```

## Bug found during validation

The initial script treated `ground_truth` as boolean via `int(bool(...))`.
One legacy prediction had `ground_truth = 9.045e-08` (effectively zero /
"NO"), which `bool()` coerces to `True`, flipping that row's contribution
from ~0.52 to ~0.08. Fixed with `1 if float(truth) >= 0.5 else 0`. Without
the fix, the script reported a phantom 0.005 gap that wasn't in the data.

## The 7-night schedule

1. ✅ 2026-04-19 (today)
2. ⏳ 2026-04-20
3. ⏳ 2026-04-21
4. ⏳ 2026-04-22
5. ⏳ 2026-04-23
6. ⏳ 2026-04-24
7. ⏳ 2026-04-25 — cutover-eligible

The nightly runner (`.github/workflows/nightly.yml`) runs the new path
against a fresh batch of resolved markets; the parity script runs at the end
and annotates the status here. After the 7th consecutive green, legacy `src/`
is archivable.
