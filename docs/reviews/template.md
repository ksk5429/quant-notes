---
title: Self-review template
description: Structure for end-of-week reviews and collaborator walkthroughs
---

# Self-review template

Copy this template into a new page under `docs/reviews/YYYY-MM-DD-topic.md`
each week, or when a discrete milestone lands. Keep the tone
brutally-honest-with-self; polish lives in the build log.

```markdown
# YYYY-MM-DD — <topic>

## What I said I'd do

- [ ] item A
- [ ] item B

## What I actually shipped

- commit / PR links
- metrics if applicable (Brier, coverage, test count)

## What I didn't ship and why

- item C — blocked on X
- item D — descoped because Y

## What surprised me (positive)

- …

## What surprised me (negative)

- …

## Open questions for next week

- …

## Decisions taken

- If any architectural or risk-profile decision was made, link to the ADR
  (add a new one if needed — `docs/decisions/NNNN-*.md` in the private repo).

## Explicit uncertainties I'm taking into next week

- …
```

## Why the template matters

Without a structured review, a solo-dev project fails the "What went wrong?
What worked well?" reflection in CLAUDE.md. The template is the bare minimum
— it forces honest separation of _intent_, _execution_, _surprises_, and
_decisions_, so future-you (or a collaborator reading this site) can
reconstruct the why.
