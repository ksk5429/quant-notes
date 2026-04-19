# Contributing to K-Fish quant-notes

This site exists so collaborators can read, comment, review, and suggest
improvements to the K-Fish research and development work without needing
access to the private trading-code repo.

**You don't need to be a developer to contribute.** Reading carefully
and asking "I don't follow step 3" is genuinely helpful — the site is
an artifact of thinking, and thinking benefits from pushback.

## Three ways to contribute, from lowest to highest effort

### 1. Post a Discussion — the default

For: questions, reviews, opinions, "did you consider X?", "this section
confused me", general conversation.

👉 <https://github.com/ksk5429/quant-notes/discussions>

Categories you'll see:

- **📣 Announcements** — read-only, posts from maintainer
- **💡 Ideas** — "what if you tried…"
- **🙋 Q&A** — ask a question, get an answer (replies can be marked
  "answer" so the thread is self-contained)
- **🗣 General** — everything else
- **📖 Show & tell** — related work, papers, tools

**Good first discussion topics**, if you're not sure what to say:

- "Walk me through how the [9-persona swarm](https://ksk5429.github.io/quant-notes/architecture/swarm-calibration/)
  is different from just asking Claude 9 times."
- "Why [DuckDB instead of Postgres](https://ksk5429.github.io/quant-notes/architecture/adrs/#adr-0002)?"
- "Help me understand the [Brier-parity evidence](https://ksk5429.github.io/quant-notes/progress/brier-parity/)
  — what would falsify it?"
- Any item on the [Open Questions](https://ksk5429.github.io/quant-notes/reviews/open-questions/)
  page — they exist specifically so others can weigh in.

### 2. Open an Issue — for a specific, actionable problem

For: bugs in the site itself (broken link, typo, wrong date), or a
concrete request for a new topic / page / correction.

Templates:

- **📝 Correction / typo** — "the ADR says X but the code says Y"
- **❓ Review-question** — "I'd like a write-up on X"
- **🐛 Site bug** — rendering, navigation, broken cross-link

### 3. Open a Pull Request — for edits you can make yourself

For: anyone who's comfortable editing Markdown.

**Easiest path:** every page on the site has an **"Edit this page"**
pencil icon at the top-right. Clicking it opens the raw `.md` in GitHub's
web editor; GitHub forks the repo for you, lets you edit, and opens a PR
with one click. No local setup.

**Local setup** (only needed for larger edits):

```bash
git clone https://github.com/ksk5429/quant-notes
cd quant-notes
pip install -r requirements.txt
mkdocs serve
# → http://127.0.0.1:8000 — hot-reload
```

PR checklist:

- [ ] Conventional-commit title (`docs: …`, `fix: …`, `feat: …`)
- [ ] If you're quoting a statute / paper / article, include a
      footnote-style citation with URL
- [ ] Preview rendering locally or in GitHub's raw view before submitting

## What we happily accept

- Corrections (factual, typographic, broken links)
- Constructive criticism on architecture decisions (ADRs)
- Pointers to relevant literature we've missed
- Suggestions for Open Questions
- Clarifying questions that expose unclear writing
- Translations (we're open to adding 한국어 versions of key pages)

## What isn't in scope here

- **Requests to see the private `kfish` repo.** Trading alpha stays
  private — [ADR-0001](https://ksk5429.github.io/quant-notes/architecture/adrs/#adr-0001).
  Discuss architecture and methodology here freely; proprietary logic
  stays private.
- **Trading tips, live market calls, user recruitment, or paid
  promotions.** This is a research-notes site, not a trading group.
- **Unrelated cryptocurrency marketing / token shilling.** Will be
  removed and the poster blocked.
- **Requests for legal advice about running your own bot.** This site
  is not legal advice; see the [legal-brief summary](https://ksk5429.github.io/quant-notes/legal/summary/).

## Code of Conduct

We use the [Contributor Covenant 2.1](https://www.contributor-covenant.org/version/2/1/code_of_conduct/).
In short: be direct, be polite, attack the idea not the person, stay on
topic. Disagree strongly in comments; be respectful to the person.

Report unacceptable behaviour via a GitHub private message to
[@ksk5429](https://github.com/ksk5429) or via the repo's
[Security tab](https://github.com/ksk5429/quant-notes/security/advisories/new)
for anything sensitive.

## Licensing

Contributions to this site are licensed under the same terms as the
repo's existing content — see [LICENSE](LICENSE). By opening a PR you
agree that your contribution is your own and can be published under
those terms.
