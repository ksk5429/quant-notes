---
title: Open questions
description: Currently-ambiguous design choices to discuss
---

# Open questions

Currently-ambiguous design choices. PRs and issue comments welcome.

## 1. Should `ksk5429/quant` (the old public repo) be archived?

The old public repo at <https://github.com/ksk5429/quant> was the scratch
space before we split into `kfish` (private) + `polymarket-oracle-risk`
(public) + `quant-notes` (this site). Three options:

=== "Archive"

    - Mark `ksk5429/quant` read-only; add a deprecation notice pointing to
      the three new repos. Keep history for academic citation.
    - **Pros**: stops accidental alpha leaks through old commits.
    - **Cons**: breaks any existing inbound links.

=== "Repurpose"

    - Keep as-is; let it be the landing-page / README-only repo that points
      at the three new ones.
    - **Pros**: clean "overview" top of funnel.
    - **Cons**: risk of drift; two places to keep current.

=== "Delete"

    - Force-delete. Blueprint-literal action.
    - **Pros**: no alpha-leak surface at all.
    - **Cons**: destroys history that's already public; rude to inbound
      links.

**Current lean**: archive with a deprecation notice.

## 2. Korean bot — launch criteria

The blueprint targets a Q3 2026 soft launch into 2-3 trusted KR Telegram
groups. Pre-launch gates that are currently only partly-ticked:

- [ ] Attorney review of Korean ToS + PIPA privacy policy (budget ₩1-3M)
- [ ] Verified agent-wallet key rotation drill (target: < 15 min)
- [ ] Fly.io NRT p95 `/healthz` < 300 ms under 10 rps load
- [ ] First real builder-fee payout verified against testnet

Honest: without legal review, launching even "KR-niched" is inadvisable.
Paper trading stays default until those gates are green.

## 3. ECE calibration convention — ours vs `netcal`'s

The Brier-parity validator shows ECE diverges 0.01-0.03 from legacy because
legacy uses `netcal`'s bin assignment and ours uses `np.digitize` with open
right boundaries. Two options:

- keep `netcal` as a dev-dep and delegate to it in `compute_ece` — tightens
  parity to near-zero but adds an optional heavy dep.
- document the binning convention, gate on Brier only (current stance).

**Current lean**: document, gate on Brier. ECE is derived, not primary.

## 4. Sharing strategy with collaborators

This site is public. Three modes of collaboration that stay safe:

1. **Discuss architecture** — ADRs, calibration math, oracle-risk
   methodology. All of `polymarket-oracle-risk` is MIT.
2. **Discuss performance** — Brier decomposition, skill score, reliability
   diagrams. All on aggregated metrics; no individual-market alpha.
3. **Discuss roadmap** — the 90-day plan, HIP-4 readiness.

What **never** goes on this site:

- individual market analyses where we hold or will hold a position
- per-persona prompt versions still in active testing
- Fernet master keys, bot tokens, API secrets (obviously)
- whale-address watchlists (could leak via aggregation)

## 5. What to track on this site vs `research-notes`?

Clean line:

- `quant-notes` (this site) — anything about the K-Fish engine, calibration,
  oracle risk, the Telegram bot, trading methodology.
- `research-notes` — offshore-wind PhD work, papers, thesis chapters.

Cross-links only when a technique genuinely transfers (rare).

## How to add a question

Edit this file directly via the "Edit this page" pencil, or open an issue on
[`quant-notes`](https://github.com/ksk5429/quant-notes/issues).
