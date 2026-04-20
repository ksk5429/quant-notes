---
title: Future HITL Tooling
description: Shippable human-in-the-loop tools and integration paths for the K-Fish forecasting stack
---

# Future HITL tooling

K-Fish is compounding infrastructure, and compounding stops the moment a silent
miscalibration, a hallucinated news item, or a half-approved trade slips past
the developer. Claude Code is fast but blind to three things: what a chart
actually looks like after a layout change, what an expert forecaster thinks of
a persona's reasoning trace, and whether a live trade is *actually* ready.
This page surveys the real, shippable HITL tools that would close those gaps,
with concrete integration paths into the existing workspace
(`apps/kfish-core/`, `packages/kfish-common/`, `hypekr-bot/`, `dbt_kfish/`).

!!! note "Scope"
    Every tool below is either open-source or has a usable free tier. No
    enterprise-only platforms, no vaporware. Where a tool has been sunset
    (Humanloop) it is called out. All URLs were verified in April 2026.

## 1. Visual UI annotation for vibe-coded surfaces

### Vibe Annotations

- **URL:** [vibe-annotations.com](https://www.vibe-annotations.com/) -
  [GitHub: RaphaelRegnier/vibe-annotations](https://github.com/RaphaelRegnier/vibe-annotations) -
  [Chrome Web Store listing](https://chromewebstore.google.com/detail/vibe-annotations-visual-f/gkofobaeeepjopdpahbicefmljcmpeof)
- **One-liner:** Browser extension + local MCP server that lets you click any
  DOM element on a running app, type a note, and have Claude Code read and act
  on the annotation as structured context.
- **Fit:** The `hypekr-bot/` Telegram Mini App and any Streamlit/Gradio
  calibration dashboard developed around `apps/kfish-core/calibration/` both
  run on localhost during development. Annotating "this Brier-by-bucket chart
  is flipped on the y-axis" or "the confidence-band legend overlaps the 80%
  bin" and letting Claude Code pick up the DOM selectors is faster than
  screenshot-and-describe. Batches up to 200 annotations across routes in one
  run.
- **Integration complexity:** Low. Install extension, run `npx vibe-annotations-mcp`,
  add to `.mcp.json`, done. Works on localhost and `file://` pages with no
  account or tracking.
- **License:** Free, open-source (AGPL-like per repo). No cloud dependency.

### Agentation

- **URL:** [agentation.com/mcp](https://www.agentation.com/mcp) -
  [Product Hunt launch (March 27, 2026, #1 day)](https://www.producthunt.com/products/agentation)
- **One-liner:** Desktop overlay tool that captures CSS selectors, React
  component names, computed styles, and a human note, and exposes them through
  an MCP server so coding agents can `read`, `acknowledge`, `reply`, `resolve`,
  and `watch` annotations.
- **Fit:** Agentation's two-way protocol is the stronger bet for the Telegram
  Mini App flow, because the agent can mark annotations `resolved` when a fix
  lands - creating an auditable trail that maps to the commit. Useful during
  the HIP-4 handler build-out when the bot UI is iterated tightly against
  design review.
- **Integration complexity:** Low-medium. The MCP server (`agentation-mcp`)
  ships as an npm package; free for personal use.
- **License:** Proprietary desktop app with a free tier; MCP bridge open per
  the product page.

!!! tip "Pick one, not both"
    Vibe Annotations is simpler and truly OSS. Agentation has the nicer
    two-way agent lifecycle. For a solo INTJ workflow, Vibe Annotations wins
    on minimalism; Agentation wins if you ever add a second reviewer.

## 2. Scientific figure review (the calibration-plot problem)

### FigureFirst

- **URL:** [figurefirst.readthedocs.io](https://figurefirst.readthedocs.io/en/latest/) -
  [GitHub: FlyRanch/figurefirst](https://github.com/FlyRanch/figurefirst) -
  [SciPy 2017 paper](https://proceedings.scipy.org/articles/shinma-7f4c6e7-009)
- **One-liner:** Python library that lets you author SVG layouts in Inkscape
  and populate named `<rect>` placeholders with matplotlib axes, so figure
  layout and data generation stay synchronized across iterations.
- **Fit:** The Brier reliability diagram, ECE-by-category plot, and PIT
  histogram that gate every cutover (`apps/kfish-core/calibration/plots.py`)
  are the highest-stakes artifacts in the repo - they are the reason a model
  ships or doesn't. FigureFirst lets you fix the layout once, then re-run
  `uv run kfish-retrodict && uv run python -m kfish_core.calibration.figures`
  and have every panel re-populate in place. The human reviewer marks up the
  Inkscape SVG (adds callouts, repositions panels) without ever having to
  touch matplotlib rcParams. The data layer regenerates; the annotation layer
  persists.
- **Integration complexity:** Medium. Requires restructuring `plots.py` to
  load a `layout.svg`, call `figurefirst.FigureLayout(...)`, and route each
  `fig.axes_by_name(...)` to an existing matplotlib call.
- **License:** BSD-3. Free.

### Scientific Inkscape

- **URL:** [GitHub: burghoff/Scientific-Inkscape](https://github.com/burghoff/Scientific-Inkscape)
- **One-liner:** Inkscape extension suite whose Scaler preserves text and tick
  positioning while resizing only the plot area - critical for assembling
  multi-panel figures from matplotlib SVGs without typography drift.
- **Fit:** Paired with FigureFirst. When Claude generates a new reliability
  diagram that needs to go into a 3-panel figure next to the PIT histogram and
  the ECE bar chart, the Scaler extension resizes each panel without breaking
  the axis labels. Use before every ADR referencing a calibration metric.
- **Integration complexity:** Low. Drop extensions into the Inkscape user
  extensions folder, restart.
- **License:** Apache 2.0. Free.

### Claude vision as fast-path reviewer

- **URL:** [Vision docs](https://docs.claude.com/en/docs/build-with-claude/vision) -
  [Pricing](https://platform.claude.com/docs/en/about-claude/pricing)
- **One-liner:** Every current Claude model (Haiku 4.5, Sonnet 4.6, Opus 4.7)
  accepts image input via the Messages API using `base64`, `url`, or Files API
  `file_id` sources, with `image/jpeg`, `image/png`, `image/gif`, and
  `image/webp` supported up to 8000x8000 px.
- **Fit:** Before a human sits down to annotate a reliability diagram, run
  Haiku 4.5 over the PNG with a structured prompt: "List every axis
  mislabeling, every bin with coverage inversion, every legend-data overlap."
  This catches ~80% of mechanical defects at ~$1/MTok input. Use Opus 4.7 for
  substantive critiques of the shape (overconfidence at tails, step function
  in isotonic fit). **Caveat:** Claude is explicitly documented as *not*
  reliable for extracting precise numeric values from image-only charts;
  always pair image review with the underlying Parquet/CSV.
- **Integration complexity:** Low. Already available in `kfish-common/llm`.
- **License:** Commercial. Image tokenization is billed against input tokens;
  Opus 4.7 can consume ~3x more tokens per high-resolution image than older
  models, so downsample to 1024px before sending unless fidelity matters.

## 3. Human annotation queues for LLM traces

### Langfuse Annotation Queues

- **URL:** [langfuse.com/docs/evaluation/evaluation-methods/annotation-queues](https://langfuse.com/docs/evaluation/evaluation-methods/annotation-queues) -
  [Self-hosted pricing](https://langfuse.com/pricing-self-host) -
  [Docker Compose deployment](https://langfuse.com/self-hosting/deployment/docker-compose)
- **One-liner:** Self-hostable LLM observability platform with first-class
  annotation queues - create a queue, push traces/observations/sessions into
  it, define a score config, and review with the built-in UI or a
  [public API](https://langfuse.com/changelog/2025-03-13-public-api-annotation-queues).
- **Fit:** This is already the chosen spine per
  [ADR-0007](https://github.com/ksk5429/kfish/blob/main/docs/decisions/0007-self-hosted-langfuse-oracle-a1.md).
  The 9-Fish swarm emits one trace per persona per market; pipe the
  high-disagreement traces (swarm variance > 0.15) into a "persona-review"
  queue with a score config of `{reasoning_quality: 1-5, citation_valid: bool,
  hallucination: bool}`. Review 10-20 traces per week during quiet periods
  and use the resulting labels to fine-tune persona prompts. All core features
  free when self-hosted on Oracle A1 - no feature gating.
- **Integration complexity:** Low (already in ADR path). Add a nightly Python
  job that calls `langfuse.create_annotation_queue_item(...)` against
  `apps/kfish-core/pipeline/nightly.py` output.
- **License:** MIT. Unlimited when self-hosted.

### LangSmith Annotation Queues

- **URL:** [docs.smith.langchain.com/evaluation/how_to_guides/human_feedback/annotation_queues](https://docs.smith.langchain.com/evaluation/how_to_guides/human_feedback/annotation_queues)
- **One-liner:** LangChain's proprietary observability platform with
  single-run and pairwise annotation queue modes for human review and
  evaluator calibration.
- **Fit:** The pairwise mode is the interesting one for K-Fish - run
  contrarian-Fish and inside-view-Fish on the same market and have the human
  choose which reasoning chain was closer to the post-hoc resolution truth.
  Feeds the persona-weight tuning loop.
- **Integration complexity:** Medium. Requires LangSmith SDK
  instrumentation, which overlaps with Langfuse; choose one.
- **License:** Proprietary, free tier up to 5k traces/month.

### Argilla

- **URL:** [argilla.io](https://argilla.io/) - [GitHub: argilla-io/argilla](https://github.com/argilla-io/argilla)
- **One-liner:** Open-source data curation platform purpose-built for LLM
  feedback collection - filters, AI-suggested labels, semantic search over
  traces.
- **Fit:** Overkill for a solo operator today, but the right landing spot if a
  second reviewer ever joins. The "Feedback" workspace is designed for
  multi-aspect scoring (one trace, five dimensions), which maps cleanly to
  the K-Fish persona reasoning rubric.
- **Integration complexity:** Medium. Self-hosted via Docker; Hugging Face
  Spaces deployment available.
- **License:** Apache 2.0.

### Label Studio

- **URL:** [labelstud.io](https://labelstud.io/) -
  [HumanSignal pricing](https://humansignal.com/pricing/)
- **One-liner:** General-purpose open-source data labeling tool with
  configurable LLM evaluation templates.
- **Fit:** Useful specifically for the *news annotation* problem - when the
  news retrieval pipeline surfaces an article, a human quickly labels
  `{relevant: bool, sentiment: -1/0/+1, priced_in: bool}` to build a
  supervised training set for the news-filter model.
- **Integration complexity:** Low-medium. Docker Compose, Apache 2.0
  community edition. Self-hosted is free (~$5-10/mo on Railway).
- **License:** Apache 2.0 community; enterprise paid.

### Sunset note: Humanloop

[Humanloop](https://humanloop.com/) was sunset September 8, 2025 after
Anthropic acquired the team. Any pre-2026 guide recommending it is stale.
The replacement in that niche is Langfuse + Argilla for OSS, LangSmith for
proprietary.

### Commercial data labeling (for completeness, not recommended here)

[Scale AI / Scale Studio](https://scale.com/pricing) (enterprise contracts,
typically $400k+) and [Surge AI](https://surgehq.ai/) (Anthropic's own RLHF
vendor, revenue ~$1.2B in 2024) are the tier-1 managed-annotator platforms.
Both are overkill for a solo quant workflow and neither has a usable free
tier.

## 4. MCP servers that implement review queues

### Human-In-the-Loop MCP Server (GongRzhe)

- **URL:** [GitHub: GongRzhe/Human-In-the-Loop-MCP-Server](https://github.com/GongRzhe/Human-In-the-Loop-MCP-Server)
- **One-liner:** MCP server exposing `get_text_input`, `get_multi_choice`,
  `get_multiline_input`, `confirm`, and `show_info` as tool calls that render
  native GUI dialogs (Tkinter/PyQt) with configurable 5-minute timeouts.
- **Fit:** The blocking dialog before a paper-trade lifts to live-trade. Claude
  drafts a trade plan, calls `confirm("Lift this trade? YES/NO")`, and the
  pipeline blocks until the human clicks. Enforces R2 (paper-trading default)
  at the protocol level rather than the code level.
- **Integration complexity:** Low. `uvx hitl-mcp-server`, add to `.mcp.json`.
- **License:** MIT.

### interactive-mcp (ttommyth)

- **URL:** [GitHub: ttommyth/interactive-mcp](https://github.com/ttommyth/interactive-mcp)
- **One-liner:** Local, cross-platform MCP server explicitly branded as
  "vibe coding should have human in the loop," with notification and input
  prompting.
- **Fit:** Alternative to GongRzhe's if you want a more modern UI. Same role.
- **Integration complexity:** Low.
- **License:** MIT.

### AgentRQ

- **URL:** [agentrq.com](https://agentrq.com) -
  [MCP integration](https://agentrq.com/features/mcp-integration) -
  [Connect Claude Code](https://agentrq.com/docs/connect-claude-code)
- **One-liner:** Hosted dashboard + MCP server that lets Claude Code
  `createTask` for a human, delivers replies via Claude's native notification
  channel in sub-second, and lets the agent resume the same session with full
  context.
- **Fit:** The right tool for asynchronous review - Claude runs the nightly
  retrodiction at 03:00 KST, flags 5 markets with model-vs-crowd divergence
  greater than 20%, and creates AgentRQ tasks. You review them on the phone
  during breakfast. Claude's next session picks up the decisions without a
  re-prompt.
- **Integration complexity:** Low. Public beta, free.
- **License:** Proprietary, free during beta.

### MCP Elicitation (native)

- **URL:** [MCP elicitation support (Claude Code 2.1.76+)](https://claudelab.net/en/articles/claude-code/mcp-elicitation-support) -
  [PulseMCP newsletter on elicitations](https://www.pulsemcp.com/posts/newsletter-claude-integrations-elicitations-mcp-conference)
- **One-liner:** Part of the MCP spec since late 2025 - servers can send
  `elicitation/create` requests with a message and JSON schema, and Claude
  Code renders an interactive form inline, blocking until the user submits.
- **Fit:** This is the protocol-level way to do what GongRzhe's server does
  in userland. If you build a `kfish-preflight` MCP server in-house (see
  synthesis below), use elicitation rather than rolling your own Tkinter.
- **Integration complexity:** Medium (building the server), low (consuming).
- **License:** Protocol-level; spec is open under the MIT-licensed
  [Model Context Protocol](https://github.com/modelcontextprotocol).

### Claude Agent SDK `canUseTool` callback

- **URL:** [Agent SDK permissions](https://docs.claude.com/en/docs/agent-sdk/permissions) -
  [Handle approvals and user input](https://platform.claude.com/docs/en/agent-sdk/user-input)
- **One-liner:** The `canUseTool` callback in the Claude Agent SDK fires
  before any non-auto-approved tool call; returns `{behavior: "allow"}` or
  `{behavior: "deny", message}` synchronously.
- **Fit:** The finest-grained control point. Wrap
  `hypekr_bot.wallet.place_order` as an agent tool, set deny-rule on
  `place_order` in live mode, and route through `canUseTool` that opens a
  console prompt or forwards to AgentRQ. Enforces R2 and R5 without modifying
  the core bot code.
- **Integration complexity:** Low if SDK is already in use.
- **License:** Commercial (Claude API).

## 5. Screenshot + canvas annotation (diagram-heavy review)

### tldraw chat template

- **URL:** [tldraw.dev/docs/ai](https://tldraw.dev/docs/ai) -
  [GitHub: tldraw/chat-template](https://github.com/tldraw/chat-template) -
  [Agent starter kit](https://tldraw.dev/starter-kits/agent)
- **One-liner:** Collaborative infinite canvas with an agent starter kit that
  sends both a PNG screenshot and structured shape data to the LLM, enabling
  iterative "draw on this chart" review.
- **Fit:** When the discussion is "should Mondrian conformal shrink more
  aggressively in the 0.7-0.9 band?", drawing an arrow on the reliability
  diagram is faster than typing. The agent sees the annotation as both an
  image and a shape graph.
- **Integration complexity:** Medium. React component; lift into the docs
  site as a dedicated `/annotate/<run_id>` route.
- **License:** Apache 2.0 (tldraw SDK has a watermark for commercial; OSS
  version is free for this use case).

### Excalidraw MCP server

- **URL:** [GitHub: yctimlin/mcp_excalidraw](https://github.com/yctimlin/mcp_excalidraw)
- **One-liner:** MCP server around Excalidraw with `describe_scene` and
  `get_canvas_screenshot` tools - closed feedback loop where the agent can
  inspect and iterate on a diagram.
- **Fit:** Lightweight alternative for ADR diagrams and architecture review.
  Less relevant to calibration plots.
- **Integration complexity:** Low.
- **License:** MIT.

## Synthesis: three highest-leverage HITL insertion points

Not all human-AI handoffs are equal. Three points in the K-Fish workflow have
disproportionate leverage because an error there is compounding, not linear.

!!! warning "Rank by cost of a silent miss"
    These are ranked by *what a silent miss costs you over 12 months* - not
    by how often the decision is made.

### Leverage point 1 - Cutover gate between legacy engine and new workspace

**Artifact:** `apps/kfish-core/calibration/figures/reliability.svg` and the
Brier/ECE numbers printed by the nightly retrodiction job in
`apps/kfish-core/src/kfish_core/pipeline/nightly.py`.

**Handoff:** Before flipping any reader of `kfish-retrodict` away from the
legacy `src.prediction` shim (the Phase 1 cutover rule: seven consecutive
nightly runs within 0.002 of legacy Brier), a human eyeballs the reliability
diagram to confirm the isotonic step function hasn't collapsed into a flat
line and the Venn-Abers band width is non-pathological.

**Tool stack:** FigureFirst (layout-stable matplotlib) + Scientific Inkscape
(typography-safe resize) + Claude vision (mechanical defect pre-pass with
Haiku 4.5) + AgentRQ (async notification with attached PNG).

**Why this matters:** A silent calibration regression over seven days would
ship bad probabilities to every downstream consumer for weeks before any
trade-level signal caught it. This is the one gate where the human *must*
look at the chart.

### Leverage point 2 - Persona reasoning review for the 9-Fish swarm

**Artifact:** Langfuse traces for each Fish persona invocation, one per
market per night. File: `apps/kfish-core/src/kfish_core/agents/` (the
contrarian, inside-view, premortem, etc. modules).

**Handoff:** A nightly sampling rule pushes the top-5 highest-disagreement
traces (swarm variance > 0.15) into a Langfuse annotation queue. The human
scores each on `{reasoning_quality 1-5, citation_valid bool, hallucination
bool}`. Monthly, those labels drive (a) prompt tuning and (b) persona weight
re-balancing in `apps/kfish-core/src/kfish_core/calibration/refit.py`.

**Tool stack:** Langfuse annotation queues (self-hosted on Oracle A1) +
optional Argilla later if a second reviewer joins.

**Why this matters:** Baseline Brier 0.206 does *not* yet beat the crowd
(0.084). The gap closes not by adding more Fish but by making existing
Fish reason better. Human labels on reasoning quality are the only signal
that tells you *which* Fish is bullshitting.

### Leverage point 3 - Paper-to-live trade preflight

**Artifact:** `hypekr-bot/src/hypekr_bot/wallet/` - specifically the
`place_order` call site and the `paper_trading` flag.

**Handoff:** Per Project Rule R2, `paper_trading: true` is default. The flip
to live must be *manual, per-order* for the first 30 days on mainnet. Claude
drafts the order, formats the trade plan (market, side, size, implied edge
vs. crowd, calibrated probability, Kelly fraction, expected P&L), and calls
a `confirm_live_trade` MCP tool. The human sees the plan on mobile (AgentRQ
notification), clicks approve, and only then does the HIP-4 handler submit.

**Tool stack:** MCP Elicitation (native, clean) or Human-In-the-Loop MCP
Server (userland, GUI dialog) + AgentRQ for mobile reach + Claude Agent SDK
`canUseTool` deny-rule on `place_order` as a belt-and-suspenders backup.

**Why this matters:** Revenue is builder-fee (5 bps) - a single wrong-side
live trade at 1 ETH is worth ~$3000 in adverse fill. A protocol-level
approval gate, reinforced by an SDK callback, is strictly safer than a
runtime config flag anyone can flip.

## What NOT to add

!!! tip "Every HITL addition has a failure mode of its own"
    More review surface means more places to get tired and click through.

- **No enterprise-managed annotation** (Scale, Surge). The cost and latency
  ruin the signal for a solo operator.
- **No CI-gated visual regression** on calibration plots. Matplotlib output
  is pixel-volatile; the review should be semantic (Claude vision +
  human eyeball), not byte-diff.
- **No approval for read-only tools** (market scan, retrodiction). Save
  the `canUseTool` callback for operations that touch money or emit
  public commitments (a forecast publication counts).

## Implementation sequencing

1. **Week 1 (free, zero-install):** Turn on Langfuse annotation queues -
   the self-hosted instance is already in the ADR path. Create the
   `persona-review` queue. Do not try to review anything yet, just push
   traces in.
2. **Week 2 (one afternoon):** Add the Human-In-the-Loop MCP server to the
   `.mcp.json` used for the hypekr-bot dev loop. Wire the `confirm` tool
   into `wallet.place_order`. Even in paper mode, exercise the dialog so
   muscle memory builds.
3. **Week 3 (one day):** Rewrite `apps/kfish-core/calibration/plots.py` on
   top of FigureFirst. Install Scientific Inkscape. Commit the canonical
   `calibration_layout.svg`.
4. **Week 4 (half a day):** Install Vibe Annotations. Use it the next time
   the Telegram Mini App UI needs a visual tweak. If it sticks, leave
   it. If not, uninstall; it was zero sunk cost.
5. **Later:** AgentRQ for mobile async approval when the workflow actually
   demands off-desk review (paper to live, anomalous nightly).

The compounding argument: every tool above costs a day or less and each one
removes a recurring friction or a silent-failure mode. None of them require
rewriting the core. All of them respect R5 (human owns strategy, risk
tolerance, and final trading approval).
