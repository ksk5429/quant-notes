---
title: Collaborate
description: How to review, comment, and contribute to the K-Fish project
---

# Collaborate

This site is public on purpose. K-Fish's calibration work, architecture
decisions, and research questions all benefit from second opinions.
Whether you're a researcher, a developer, a trader, or a
friend who's curious what I've been building — there are concrete ways
you can help.

## Three-lane model

=== "🟢 Fast lane — ask or comment (no signup beyond GitHub)"

    **Best for:** first-time contributors, questions, opinions, "did you
    consider X?", pointers to papers you've read.

    **Where:** [github.com/ksk5429/quant-notes/discussions](https://github.com/ksk5429/quant-notes/discussions)

    **Format:** casual — just write the thought. If it turns into a
    thread, it might get promoted to an issue or incorporated into the
    site.

    !!! tip "Great first topics if you're unsure"
        - Anything on the [Open Questions](reviews/open-questions.md) page
        - "I don't follow this step" — confusion is useful signal
        - "I read paper X and think it contradicts Y" — bring the citation
        - "My intuition on risk here is different" — dissent is welcome

=== "🟡 Mid lane — raise a specific issue"

    **Best for:** "page Z contains factual error W", "this link is broken",
    "please write a review of topic T."

    **Where:** [github.com/ksk5429/quant-notes/issues/new/choose](https://github.com/ksk5429/quant-notes/issues/new/choose)

    Pre-built templates exist for:

    - 📝 Correction / typo / broken link
    - ❓ Review question or new-topic request
    - 🐛 Site bug (rendering, navigation, build)

    Issues get triaged into the site's backlog. Large asks may become
    a PR from me; smaller corrections get a fast merge.

=== "🔴 Deep lane — open a pull request"

    **Best for:** anyone comfortable editing Markdown who wants to
    actually make the change.

    **Easiest path:** click the "Edit this page" ✏️ pencil at the
    top-right of any page on [ksk5429.github.io/quant-notes](https://ksk5429.github.io/quant-notes/).
    GitHub handles the fork-and-PR flow in your browser.

    **For larger edits:**
    ```bash
    git clone https://github.com/ksk5429/quant-notes
    cd quant-notes
    pip install -r requirements.txt
    mkdocs serve      # hot-reload at http://127.0.0.1:8000
    ```

    See [CONTRIBUTING.md](https://github.com/ksk5429/quant-notes/blob/main/CONTRIBUTING.md)
    for the full PR checklist.

## What you can actually help with

Concrete ways that move the project forward:

- **Push back on an ADR.** Each [architecture decision record](architecture/adrs.md)
  has clear reasoning. If you think it's wrong, the comments on the
  open-questions page are where that lives — and a single well-argued
  dissent can flip a design.
- **Reproduce a reported result.** [Brier parity](progress/brier-parity.md)
  claims exact match on three legacy runs. If you can run
  `python scripts/brier_parity.py --all` and see the same numbers,
  that's an independent verification.
- **Nominate a paper.** If you've read something in LLM calibration /
  prediction-market research / Bayesian risk modeling that we should
  engage with, drop it in Discussions.
- **Spot contradictions** between the two canonical blueprints and the
  current build log. They've drifted a little; keeping them honest
  matters.
- **Suggest clearer phrasing.** Half the value of this site is
  expository. If something didn't land for you, it won't land for the
  next reader either.

## What is and isn't in scope

### In scope

- Architecture, methodology, calibration math
- Publicly-available code in [polymarket-oracle-risk](https://github.com/ksk5429/polymarket-oracle-risk)
- Literature pointers, prior-art corrections
- Translations (especially 한국어 versions of key pages)
- Review of ADRs, runbooks, and published reports
- Site UX (navigation, rendering, accessibility)

### Out of scope

- **Access to the private [kfish](https://github.com/ksk5429/kfish) repo.**
  Trading alpha stays private — see [ADR-0001](architecture/adrs.md#adr-0001).
  Everything needed for genuine architecture discussion is public.
- **Trading tips / market calls / paid promotion.** This is not a
  trading community. The bot's live trading is blocked pending legal
  review anyway.
- **Requests for legal advice about running your own bot.** See the
  [legal-brief summary](legal/summary.md) — you need your own Korean
  counsel, not a fork of ours.

## Review-credit

Substantive contributors get credit:

- **One good review** that changes a design → acknowledged in the build
  log entry that captures the change.
- **Multiple reviews / sustained engagement** → listed in the
  [AUTHORS](https://github.com/ksk5429/quant-notes/blob/main/AUTHORS)
  file (created as soon as there's a second name to add).
- **Port / translation work** → author credit on the translated page
  via the page's front-matter `authors:` field.

## A note for "friend-level" collaboration

If I've shared this site with you specifically — thanks for reading. The
most useful thing you can do is **comment aggressively on parts you think
are wrong**. I'd rather have five hard challenges on the calibration
math than twenty polite "this looks nice" comments. Direct > polite.

If you're unsure where to start, just open a Discussion thread with
whatever's on your mind. I'll engage.

## What if I want to chat privately first

Email <ksk54299@gmail.com> or DM [@ksk5429](https://github.com/ksk5429) on
GitHub. Fine to scope-out an idea before making it public.
