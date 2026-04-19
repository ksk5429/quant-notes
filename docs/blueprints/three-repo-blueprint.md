# Production-grade GitHub blueprint for a three-repo Python stack

**Recommendation: build a private uv-workspace monorepo `kfish` (containing `kfish-core`, `hypekr-bot`, and a shared `kfish-common` library) plus a standalone public repo `polymarket-oracle-risk` for PyPI.** This hybrid layout is the only structure that reconciles GitHub's all-or-nothing repo visibility with your actual semantic boundary: trading alpha and Telegram-bot user funds stay private, while the reputation-building analyzer ships publicly. The monorepo gives you one `uv.lock`, one CI config, atomic refactors across the calibration/Hyperliquid/Langfuse utilities, and a single mental model for Claude Code. The polyrepo for the analyzer cleanly signals OSS intent and isolates PyPI publishing. Every other arrangement — pure monorepo, three polyrepos, submodules — creates more friction than it solves for a solo dev. The rest of this guide turns that recommendation into copy-pasteable configuration for the weekend.

**A quick ground rule:** every third-party GitHub Action in the YAML below is written with a floating major tag for readability. Before you ship, pin each one by commit SHA (Renovate will keep them current). The 2024 `tj-actions/changed-files` incident and the November 2025 Shai-Hulud self-hosted-runner worm made SHA pinning non-optional for anything that touches money.

---

## 1. Structure: why the hybrid beats monorepo and polyrepo

### Why not a pure monorepo

GitHub repos are public **or** private — there is no per-directory visibility. Putting `polymarket-oracle-risk` inside `kfish` would either leak trading alpha or prevent PyPI/OSS credibility. Any "extract it later under pressure" plan ends with alpha in git history. Assume private stays private forever.

### Why not pure polyrepo

Three repos × three CI configs × three dependabot configs × three pre-commit configs is real weekly toll. You'd juggle a fourth "shared lib" repo (`kfish-common`) distributed via `git+ssh://` or Cloudsmith, live with version drift between consumers, and lose atomic refactor. **GitHub Packages is not a supported Python registry** — it lists npm, Maven, NuGet, RubyGems, Docker/Container only. Any blog recommending it for pip is wrong.

### The hybrid at a glance

```
kfish/                                       # PRIVATE monorepo (uv workspace)
├── packages/
│   └── kfish-common/        # DuckDB helpers, Langfuse wrappers, Claude+OpenAI clients
├── apps/
│   ├── kfish-core/          # 9-agent LLM swarm, calibration spine, Polymarket→HIP-4
│   └── hypekr-bot/          # aiogram 3 bot, per-user wallets, builder-fee logic
├── infra/                   # docker-compose (Langfuse), systemd units
└── pyproject.toml           # workspace root

polymarket-oracle-risk/      # PUBLIC polyrepo (MIT, PyPI)
└── standalone package, standalone CI, standalone docs
```

The public repo consumes **nothing** from the private monorepo. If both need the same Polymarket CLOB client, duplicate the small bit (<500 LOC) or publish a separate public `polymarket-clob-py` as its own PyPI package that both consume. Don't reach for submodules.

### When you'd regret each choice

| Choice | Failure mode |
|---|---|
| **Hybrid (recommended)** | Small drift between public and private Polymarket clients. Mitigation: a `make sync-clients` diff-check target, or eventual promotion to its own public package. |
| Pure monorepo | Forced extraction of the analyzer under time pressure, with git-history secret audit. |
| Pure polyrepo | Weekends lost to four Dependabot PRs and lockstep bumps of `kfish-common` across N consumers. |
| Submodules | Every PR requires "did you bump the submodule?" commentary; Claude Code gets confused by detached HEADs. |

### Workspace root `pyproject.toml`

```toml
[project]
name = "kfish"
version = "0.0.0"
requires-python = ">=3.12,<3.13"

[tool.uv.workspace]
members = ["packages/*", "apps/*"]

[tool.uv]
package = false              # virtual root

[dependency-groups]           # PEP 735
dev = ["ruff>=0.8", "mypy>=1.13", "pytest>=8.3", "pytest-asyncio>=0.24",
       "pytest-cov>=6.0", "pre-commit>=4.0", "ipython>=8.30"]

[tool.ruff]
line-length = 100
target-version = "py312"
```

Each member depends on siblings via `[tool.uv.sources] kfish-common = { workspace = true }`. `uv sync --all-packages` installs everything editable into one `.venv/`; `uv run --package kfish-core ...` executes inside a specific member; `uv lock` produces a single cross-platform lockfile with hashes.

### Selective CI via `dorny/paths-filter`

The canonical 2025 pattern: one `changes` job fans out to per-package jobs, with a final `ci-pass` meta-job that branch protection requires. That meta-job sidesteps GitHub's ambiguity around skipped required checks.

```yaml
jobs:
  changes:
    runs-on: ubuntu-latest
    outputs:
      core:   ${{ steps.f.outputs.core }}
      hypekr: ${{ steps.f.outputs.hypekr }}
    steps:
      - uses: actions/checkout@v4
      - uses: dorny/paths-filter@v3
        id: f
        with:
          filters: |
            common:  ['packages/kfish-common/**', 'uv.lock']
            core:    ['apps/kfish-core/**', 'packages/kfish-common/**', 'uv.lock']
            hypekr:  ['apps/hypekr-bot/**', 'packages/kfish-common/**', 'uv.lock']

  ci-pass:
    needs: [test-common, test-core, test-hypekr]
    if: always()
    runs-on: ubuntu-latest
    steps:
      - run: |
          [[ "${{ contains(needs.*.result, 'failure') || contains(needs.*.result, 'cancelled') }}" == "false" ]]
```

### `gh repo create` commands

```bash
# Private monorepo
gh repo create ksk5429/kfish \
  --private --description "Private LLM swarm + Hyperliquid Telegram bot (uv workspace)" \
  --gitignore Python --disable-wiki --source=. --remote=origin --push

gh repo edit ksk5429/kfish \
  --enable-issues --enable-discussions=false --enable-projects=false --enable-wiki=false \
  --enable-squash-merge --enable-merge-commit=false --enable-rebase-merge=false \
  --enable-auto-merge --delete-branch-on-merge --allow-update-branch \
  --add-topic python,uv,monorepo,llm,trading,polymarket,hyperliquid

# Public PyPI repo
gh repo create ksk5429/polymarket-oracle-risk \
  --public --description "Risk analyzer for Polymarket oracle resolutions (UMA OO). PyPI package." \
  --homepage "https://pypi.org/project/polymarket-oracle-risk/" \
  --gitignore Python --license MIT --source=. --remote=origin --push

gh repo edit ksk5429/polymarket-oracle-risk \
  --enable-issues --enable-discussions \
  --enable-squash-merge --enable-merge-commit=false --enable-rebase-merge=true \
  --enable-auto-merge --delete-branch-on-merge --allow-update-branch \
  --add-topic polymarket,prediction-markets,uma,oracle,risk,python,pypi

# Secret scanning + push protection (public is free; private needs GHAS)
gh api -X PATCH repos/ksk5429/polymarket-oracle-risk \
  -F security_and_analysis[secret_scanning][status]=enabled \
  -F security_and_analysis[secret_scanning_push_protection][status]=enabled
gh api -X PUT repos/ksk5429/polymarket-oracle-risk/vulnerability-alerts
gh api -X PUT repos/ksk5429/polymarket-oracle-risk/automated-security-fixes
```

### Branch protection for a solo dev (the trick)

You cannot approve your own PRs. The workaround is to require **status checks + linear history + signed commits**, not reviews. Use modern Rulesets:

```bash
cat > ruleset.json <<'EOF'
{
  "name": "protect-main",
  "target": "branch",
  "enforcement": "active",
  "conditions": { "ref_name": { "include": ["refs/heads/main"], "exclude": [] } },
  "rules": [
    { "type": "deletion" },
    { "type": "non_fast_forward" },
    { "type": "required_linear_history" },
    { "type": "required_signatures" },
    { "type": "pull_request",
      "parameters": {
        "required_approving_review_count": 0,
        "dismiss_stale_reviews_on_push": true,
        "require_code_owner_review": false,
        "required_review_thread_resolution": true } },
    { "type": "required_status_checks",
      "parameters": {
        "strict_required_status_checks_policy": true,
        "required_status_checks": [ { "context": "ci-pass" } ] } }
  ],
  "bypass_actors": []
}
EOF
gh api -X POST repos/ksk5429/kfish/rulesets --input ruleset.json
gh api -X POST repos/ksk5429/polymarket-oracle-risk/rulesets --input ruleset.json
```

`pull_request` with zero required approvals still forces the PR workflow (no direct pushes to `main`); `required_signatures` enforces SSH/GPG on every commit; `ci-pass` is the meta-check from above. Leave `bypass_actors: []` so you can't accidentally sidestep your own rules. Trunk-based development with short-lived `feat/`, `fix/`, `chore/`, `docs/` branches is the right workflow — Gitflow is ceremony you don't need.

---

## 2. Commit signing, pre-commit, and commits

### Pick SSH signing

GitHub supports GPG, SSH (since Aug 2022), and S/MIME. Sigstore `gitsign` works via `gpg.format x509` but GitHub still doesn't show it as Verified because Sigstore's CA root isn't in GitHub's trust store. For a solo dev on Windows 11 + WSL2, **use SSH signing with a dedicated Ed25519 key stored in 1Password**: zero pinentry/gpg-agent fragility, one key for auth and signing (registered twice on GitHub — the signing slot is separate), native Verified badge, cross-machine sync.

```bash
ssh-keygen -t ed25519 -C "ksk5429@2026-signing" -f ~/.ssh/id_ed25519_signing -N ""
git config --global gpg.format ssh
git config --global user.signingkey ~/.ssh/id_ed25519_signing.pub
git config --global commit.gpgsign true
git config --global tag.gpgsign true
mkdir -p ~/.config/git
echo "ksk5429@users.noreply.github.com $(cat ~/.ssh/id_ed25519_signing.pub)" \
  > ~/.config/git/allowed_signers
git config --global gpg.ssh.allowedSignersFile ~/.config/git/allowed_signers
# Upload ~/.ssh/id_ed25519_signing.pub to GitHub → SSH and GPG keys → type=Signing key
```

For a 1Password-agent bridge into WSL2 (keeps the private key off disk), use `npiperelay.exe` + `socat` to forward the Windows named-pipe agent; then set `git config --global gpg.ssh.program "/mnt/c/.../op-ssh-sign.exe"`.

Use `gitsign` in CI only — workload-identity OIDC from Actions makes it painless for bot-originated commits (release-please PRs) and provides Rekor-backed provenance without the local verification gap.

### Complete `.pre-commit-config.yaml`

```yaml
default_language_version: { python: python3.12 }
default_install_hook_types: [pre-commit, commit-msg, pre-push]
fail_fast: false
ci:
  autofix_commit_msg: "style: pre-commit.ci auto fixes"
  autoupdate_schedule: monthly
  skip: [pytest-local, pyright-local]

repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v5.0.0
    hooks:
      - { id: trailing-whitespace, args: [--markdown-linebreak-ext=md] }
      - { id: end-of-file-fixer }
      - { id: check-added-large-files, args: [--maxkb=5120],
          exclude: '\.(duckdb|parquet|arrow|feather|joblib|pkl)$' }
      - { id: check-merge-conflict }
      - { id: check-yaml, args: [--allow-multiple-documents] }
      - { id: check-toml }
      - { id: check-json }
      - { id: detect-private-key }
      - { id: mixed-line-ending, args: [--fix=lf] }
      - { id: check-case-conflict }
      - { id: check-ast }

  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.8.6
    hooks:
      - { id: ruff-check, args: [--fix, --exit-non-zero-on-fix],
          types_or: [python, pyi, jupyter] }
      - { id: ruff-format, types_or: [python, pyi, jupyter] }

  # Use local pyright so it sees your uv-managed venv (faster than mypy in 2026;
  # keep mypy only if you rely on Django/SQLAlchemy plugins).
  - repo: local
    hooks:
      - id: pyright-local
        name: pyright (local uv env)
        entry: uv run pyright
        language: system
        types: [python]
        pass_filenames: false
        require_serial: true
        stages: [pre-commit]

  - repo: https://github.com/PyCQA/bandit
    rev: 1.8.0
    hooks:
      - { id: bandit, args: [-c, pyproject.toml, -q],
          additional_dependencies: ["bandit[toml]"], exclude: ^tests/ }

  # gitleaks = broad/fast pattern scan, detect-secrets = baseline/allowlist for
  # legacy code. Use both; run TruffleHog weekly in CI for verified-live leaks.
  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.5.0
    hooks:
      - { id: detect-secrets, args: [--baseline, .secrets.baseline],
          exclude: ^(tests/fixtures/|\.secrets\.baseline$|uv\.lock$) }

  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.21.2
    hooks: [ { id: gitleaks } ]

  - repo: https://github.com/abravalheri/validate-pyproject
    rev: v0.23
    hooks:
      - { id: validate-pyproject,
          additional_dependencies: ["validate-pyproject-schema-store[all]"] }

  - repo: https://github.com/rhysd/actionlint
    rev: v1.7.7
    hooks: [ { id: actionlint } ]

  - repo: https://github.com/commitizen-tools/commitizen
    rev: v4.1.0
    hooks:
      - { id: commitizen, stages: [commit-msg] }
      - { id: commitizen-branch, stages: [pre-push] }

  - repo: local
    hooks:
      - id: pytest-local
        name: pytest (pre-push, fast suite)
        entry: uv run pytest -x -q --no-header -m "not slow"
        language: system
        pass_filenames: false
        stages: [pre-push]
        types: [python]
```

Bootstrap: `uv tool run detect-secrets scan > .secrets.baseline && pre-commit install --install-hooks && pre-commit install --hook-type commit-msg --hook-type pre-push && pre-commit run --all-files`. Run `pre-commit autoupdate` monthly; never let it run in CI without a human seeing the diff — a silent mypy/ruff major bump can rewrite your whole codebase.

### Conventional commits and versioning

**Use commitizen** (Python-native, gives `cz commit` interactive prompts + `cz bump`) over commitlint (Node). Enforce the PR-title format at CI with `amannn/action-semantic-pull-request@v5` — essential if you squash-merge, because squash inherits the PR title.

**Use release-please** for all three repos. Its release-PR model fits solo dev: commits accumulate on `main`, release-please maintains a running "Release PR" with changelog + version bump, and you decide *when* to cut a release by merging it. Set `bump-minor-pre-major: true` so `feat:` bumps 0.2.7 → 0.3.0 during pre-production. Use `major_version_zero = true` in commitizen to keep oracle-analyzer on 0.x until you bless the public API as 1.0.

**Version source of truth:** use release-please to write `pyproject.toml` directly for the public analyzer. Use `hatch-vcs` with `local_scheme = "no-local-version"` for `kfish-core` so every commit gets a unique `0.3.1.devN+g<sha>` version — trivial model-artifact versioning in DuckDB. Never run both on the same repo.

### PyPI publishing via OIDC Trusted Publishing

No long-lived tokens. One-time on pypi.org → *Publishing → Add a pending publisher*: owner=`ksk5429`, repo=`polymarket-oracle-risk`, workflow=`release.yml`, environment=`pypi`. Create a matching GitHub Environment named `pypi`. Then `pypa/gh-action-pypi-publish@release/v1` with `permissions: id-token: write` mints a short-lived token per run and auto-generates + uploads Sigstore PEP 740 attestations. Missing either `id-token: write` or `environment:` produces cryptic 403s — the most common mistake.

---

## 3. CI/CD — complete, copy-pasteable workflows

All workflows below are production-ready. The meta-pattern: default `permissions: read-all`, elevate per job; add `concurrency` to cancel superseded PR runs but never release runs; timeouts on every job; `persist-credentials: false` on every `actions/checkout` (Zizmor hardening).

### `ci.yml` — lint, type, test, security, SBOM

```yaml
name: CI
on:
  pull_request: { branches: [main] }
  push:         { branches: [main] }
  workflow_dispatch:

concurrency:
  group: ci-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}

permissions: { contents: read }
env:
  PYTHON_VERSION: "3.12"
  UV_VERSION: "0.5.x"
  UV_CACHE_DIR: /tmp/.uv-cache
  FORCE_COLOR: "1"

jobs:
  lint:
    runs-on: ubuntu-latest
    timeout-minutes: 5
    steps:
      - uses: actions/checkout@v4
        with: { persist-credentials: false, fetch-depth: 1 }
      - uses: astral-sh/setup-uv@v6
        with:
          version: ${{ env.UV_VERSION }}
          enable-cache: true
          cache-dependency-glob: |
            uv.lock
            pyproject.toml
      - run: uv sync --frozen --only-group dev
      - run: uv run ruff check .
      - run: uv run ruff format --check .

  typecheck:
    needs: lint
    runs-on: ubuntu-latest
    timeout-minutes: 8
    steps:
      - uses: actions/checkout@v4
        with: { persist-credentials: false }
      - uses: astral-sh/setup-uv@v6
        with: { version: "0.5.x", enable-cache: true }
      - run: uv sync --frozen
      - uses: actions/cache@v4
        with:
          path: .mypy_cache
          key: mypy-${{ runner.os }}-${{ env.PYTHON_VERSION }}-${{ hashFiles('uv.lock') }}
          restore-keys: mypy-${{ runner.os }}-${{ env.PYTHON_VERSION }}-
      - run: uv run mypy src tests

  test:
    needs: lint
    runs-on: ubuntu-latest
    timeout-minutes: 15
    strategy:
      fail-fast: false
      matrix: { python: ["3.12"] }
    steps:
      - uses: actions/checkout@v4
        with: { persist-credentials: false }
      - uses: astral-sh/setup-uv@v6
        with: { version: "0.5.x", python-version: ${{ matrix.python }}, enable-cache: true }
      - uses: actions/cache@v4
        with:
          path: ~/.duckdb/extensions
          key: duckdb-${{ runner.os }}-${{ hashFiles('uv.lock') }}
      - run: uv sync --frozen
      - run: |
          uv run pytest -n auto --dist=loadgroup \
            --cov=src --cov-report=xml --cov-report=term-missing \
            --junitxml=junit.xml -o junit_family=legacy
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: coverage-${{ matrix.python }}
          path: |
            coverage.xml
            junit.xml
          retention-days: 7

  security:
    needs: lint
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4
        with: { persist-credentials: false }
      - uses: astral-sh/setup-uv@v6
        with: { version: "0.5.x", enable-cache: true }
      - run: uv sync --frozen --only-group dev
      - run: uv run bandit -r src -ll -ii -f sarif -o bandit.sarif || true
      - uses: github/codeql-action/upload-sarif@v3
        if: always()
        with: { sarif_file: bandit.sarif, category: bandit }
      - run: uv run pip-audit --strict --disable-pip --progress-spinner=off
      - name: Safety scan
        env: { SAFETY_API_KEY: ${{ secrets.SAFETY_API_KEY }} }
        run: uv run safety scan --output json --save-as json safety.json || true
      - uses: actions/upload-artifact@v4
        if: always()
        with: { name: security-reports, path: "bandit.sarif\nsafety.json", retention-days: 14 }

  sbom:
    needs: lint
    runs-on: ubuntu-latest
    timeout-minutes: 5
    steps:
      - uses: actions/checkout@v4
        with: { persist-credentials: false }
      - uses: anchore/sbom-action@v0
        with:
          path: .
          format: spdx-json
          output-file: sbom.spdx.json
          artifact-name: sbom-${{ github.sha }}.spdx.json
          upload-artifact: true

  ci-pass:
    needs: [lint, typecheck, test, security, sbom]
    if: always()
    runs-on: ubuntu-latest
    steps:
      - run: |
          if [[ "${{ contains(needs.*.result, 'failure') }}" == "true" \
             || "${{ contains(needs.*.result, 'cancelled') }}" == "true" ]]; then
            echo "One or more jobs failed." >&2; exit 1
          fi
```

### `release.yml` — release-please + OIDC PyPI + SLSA provenance

```yaml
name: Release
on:
  push: { branches: [main] }

concurrency:
  group: release-${{ github.ref }}
  cancel-in-progress: false

permissions: { contents: read }

jobs:
  release-please:
    runs-on: ubuntu-latest
    timeout-minutes: 5
    permissions: { contents: write, pull-requests: write, issues: write }
    outputs:
      release_created: ${{ steps.rp.outputs.release_created }}
      tag_name:        ${{ steps.rp.outputs.tag_name }}
    steps:
      - uses: googleapis/release-please-action@v4
        id: rp
        with:
          config-file: release-please-config.json
          manifest-file: .release-please-manifest.json
          token: ${{ secrets.GITHUB_TOKEN }}

  build:
    needs: release-please
    if: needs.release-please.outputs.release_created == 'true'
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4
        with:
          persist-credentials: false
          ref: ${{ needs.release-please.outputs.tag_name }}
          fetch-depth: 0
      - uses: astral-sh/setup-uv@v6
        with: { version: "0.5.x", enable-cache: true, python-version: "3.12" }
      - run: uv build --sdist --wheel --out-dir dist
      - run: uv run --with twine twine check --strict dist/*
      - uses: actions/upload-artifact@v4
        with:
          name: python-package-distributions
          path: dist/
          retention-days: 7
          if-no-files-found: error

  attest:
    needs: [release-please, build]
    runs-on: ubuntu-latest
    timeout-minutes: 5
    permissions: { id-token: write, contents: read, attestations: write }
    steps:
      - uses: actions/download-artifact@v4
        with: { name: python-package-distributions, path: dist }
      - uses: actions/attest-build-provenance@v4
        with: { subject-path: "dist/*" }

  publish-pypi:
    needs: [release-please, build, attest]
    runs-on: ubuntu-latest
    timeout-minutes: 10
    environment:
      name: pypi
      url: https://pypi.org/p/${{ github.event.repository.name }}
    permissions: { id-token: write, contents: read }
    steps:
      - uses: actions/download-artifact@v4
        with: { name: python-package-distributions, path: dist }
      - uses: pypa/gh-action-pypi-publish@release/v1
        with: { attestations: true, print-hash: true, verbose: true }

  finalize-release:
    needs: [release-please, publish-pypi]
    runs-on: ubuntu-latest
    timeout-minutes: 5
    permissions: { contents: write }
    steps:
      - uses: actions/checkout@v4
        with:
          persist-credentials: false
          ref: ${{ needs.release-please.outputs.tag_name }}
      - uses: actions/download-artifact@v4
        with: { name: python-package-distributions, path: dist }
      - uses: anchore/sbom-action@v0
        with:
          path: .
          format: spdx-json
          output-file: sbom.spdx.json
          upload-release-assets: true
      - env: { GH_TOKEN: ${{ secrets.GITHUB_TOKEN }} }
        run: gh release upload "${{ needs.release-please.outputs.tag_name }}" dist/* sbom.spdx.json --clobber
```

### `security.yml` — CodeQL, dependency review, gitleaks, Scorecard

```yaml
name: Security
on:
  push: { branches: [main] }
  pull_request: { branches: [main] }
  schedule: [ { cron: "30 1 * * 6" } ]
  workflow_dispatch:

concurrency:
  group: security-${{ github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}

permissions: read-all

jobs:
  codeql:
    runs-on: ubuntu-latest
    timeout-minutes: 20
    permissions: { security-events: write, actions: read, contents: read }
    strategy: { fail-fast: false, matrix: { language: [python] } }
    steps:
      - uses: actions/checkout@v4
        with: { persist-credentials: false }
      - uses: github/codeql-action/init@v3
        with:
          languages: ${{ matrix.language }}
          queries: +security-extended,security-and-quality
      - uses: github/codeql-action/analyze@v3
        with: { category: "/language:${{ matrix.language }}" }

  dep-review:
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    timeout-minutes: 5
    permissions: { contents: read, pull-requests: write }
    steps:
      - uses: actions/checkout@v4
        with: { persist-credentials: false }
      - uses: actions/dependency-review-action@v4
        with:
          fail-on-severity: high
          comment-summary-in-pr: always
          deny-licenses: AGPL-3.0, GPL-3.0

  gitleaks:
    runs-on: ubuntu-latest
    timeout-minutes: 5
    permissions: { contents: read, pull-requests: write }
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0, persist-credentials: false }
      - uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          GITLEAKS_LICENSE: ${{ secrets.GITLEAKS_LICENSE }}
          GITLEAKS_ENABLE_SUMMARY: "true"
          GITLEAKS_ENABLE_UPLOAD_ARTIFACT: "true"

  scorecard:
    if: github.event_name != 'pull_request'
    runs-on: ubuntu-latest
    timeout-minutes: 10
    permissions:
      security-events: write
      id-token: write
      contents: read
      actions: read
    steps:
      - uses: actions/checkout@v4
        with: { persist-credentials: false }
      - uses: ossf/scorecard-action@v2.4.3
        with: { results_file: results.sarif, results_format: sarif, publish_results: true }
      - uses: actions/upload-artifact@v4
        with: { name: scorecard-sarif, path: results.sarif, retention-days: 5 }
      - uses: github/codeql-action/upload-sarif@v3
        with: { sarif_file: results.sarif, category: ossf-scorecard }
```

### `dependabot.yml` + auto-merge

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "pip"
    directory: "/"
    schedule: { interval: weekly, day: monday, time: "06:00", timezone: "Asia/Seoul" }
    open-pull-requests-limit: 10
    labels: [dependencies, python]
    commit-message: { prefix: "build(deps)", prefix-development: "build(deps-dev)", include: "scope" }
    cooldown: { default-days: 3, semver-major-days: 7 }
    groups:
      python-minor-patch:
        applies-to: version-updates
        update-types: [minor, patch]
      python-security:
        applies-to: security-updates
        patterns: ["*"]
    ignore:
      - dependency-name: "duckdb"
        update-types: ["version-update:semver-major"]

  - package-ecosystem: "github-actions"
    directory: "/"
    schedule: { interval: weekly, day: monday, time: "06:00", timezone: "Asia/Seoul" }
    open-pull-requests-limit: 5
    labels: [dependencies, ci]
    commit-message: { prefix: "ci(deps)", include: "scope" }
    groups:
      actions-minor-patch: { update-types: [minor, patch] }

  - package-ecosystem: "docker"
    directory: "/"
    schedule: { interval: weekly }
    labels: [dependencies, docker]
    commit-message: { prefix: "build(docker)", include: "scope" }
    groups:
      docker-minor-patch: { update-types: [minor, patch] }
```

```yaml
# .github/workflows/dependabot-auto-merge.yml
name: Dependabot auto-merge
on:
  pull_request:
    types: [opened, synchronize, reopened, ready_for_review]

concurrency:
  group: dependabot-automerge-${{ github.event.pull_request.number }}
  cancel-in-progress: true

permissions: { contents: write, pull-requests: write }

jobs:
  automerge:
    if: github.actor == 'dependabot[bot]'
    runs-on: ubuntu-latest
    timeout-minutes: 5
    steps:
      - id: meta
        uses: dependabot/fetch-metadata@v2
        with: { github-token: "${{ secrets.GITHUB_TOKEN }}" }
      - if: |
          steps.meta.outputs.update-type == 'version-update:semver-patch' ||
          (steps.meta.outputs.update-type == 'version-update:semver-minor' &&
           (steps.meta.outputs.package-ecosystem == 'github_actions' ||
            steps.meta.outputs.package-ecosystem == 'docker' ||
            steps.meta.outputs.package-ecosystem == 'pip'))
        env: { GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}, PR_URL: ${{ github.event.pull_request.html_url }} }
        run: |
          gh pr review "$PR_URL" --approve --body "Auto-approving ${{ steps.meta.outputs.update-type }} bump"
          gh pr merge "$PR_URL" --auto --squash --delete-branch
```

### `nightly.yml` — DuckDB → partitioned Parquet → B2/R2

```yaml
name: Nightly pipeline + backup
on:
  schedule: [ { cron: "17 18 * * *" } ]   # 03:17 JST
  workflow_dispatch:
    inputs:
      full_refresh: { type: boolean, default: false }

concurrency: { group: nightly-${{ github.ref }}, cancel-in-progress: false }
permissions: { contents: read }

env:
  UV_VERSION: "0.5.x"
  BACKUP_BUCKET: "s3://kfish-backups"
  AWS_ENDPOINT_URL_S3: "https://s3.us-west-004.backblazeb2.com"
  AWS_DEFAULT_REGION: "us-west-004"

jobs:
  pipeline:
    runs-on: [self-hosted, Linux, ARM64, duckdb]
    timeout-minutes: 90
    steps:
      - uses: actions/checkout@v4
        with: { persist-credentials: false }
      - uses: astral-sh/setup-uv@v6
        with:
          version: ${{ env.UV_VERSION }}
          enable-cache: true
          cache-local-path: ${{ github.workspace }}/.uv-cache
      - run: uv sync --frozen --no-dev
      - env:
          LANGFUSE_HOST: "http://langfuse.lan:3000"
          FULL_REFRESH: ${{ inputs.full_refresh || 'false' }}
        run: uv run python -m kfish_core.pipeline.nightly
      - run: |
          uv run python -m kfish_core.pipeline.export_parquet \
            --src data/warehouse.duckdb --dst out/parquet
      - name: Install aws-cli
        run: |
          if ! command -v aws >/dev/null; then
            curl "https://awscli.amazonaws.com/awscli-exe-linux-aarch64.zip" -o awscli.zip
            unzip -q awscli.zip && sudo ./aws/install --update
          fi
      - env:
          AWS_ACCESS_KEY_ID:     ${{ secrets.B2_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.B2_APPLICATION_KEY }}
        run: |
          TODAY=$(date -u +%Y-%m-%d)
          aws s3 sync out/parquet "${{ env.BACKUP_BUCKET }}/warehouse/run=${TODAY}/" \
            --endpoint-url "${{ env.AWS_ENDPOINT_URL_S3 }}" --only-show-errors --size-only
          aws s3 sync out/parquet "${{ env.BACKUP_BUCKET }}/warehouse/latest/" \
            --endpoint-url "${{ env.AWS_ENDPOINT_URL_S3 }}" --delete --only-show-errors
      - if: always()
        run: rm -rf out/ .uv-cache/
```

### `deploy-bot.yml` — Fly.io NRT on tag push

```yaml
name: Deploy bot → Fly.io (NRT)
on:
  push: { tags: ["v*.*.*"] }
  workflow_dispatch: { inputs: { image_tag: { required: false } } }

concurrency: { group: deploy-bot-fly, cancel-in-progress: false }
permissions: { contents: read, packages: write, id-token: write, attestations: write }

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    timeout-minutes: 20
    environment: { name: production, url: https://hypekr-bot.fly.dev }
    env: { IMAGE: ghcr.io/${{ github.repository }}/bot }
    steps:
      - uses: actions/checkout@v4
        with: { persist-credentials: false }
      - uses: docker/setup-qemu-action@v3
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - id: tag
        run: |
          if [[ "${{ github.event_name }}" == "push" ]]; then
            echo "tag=${GITHUB_REF_NAME}" >> "$GITHUB_OUTPUT"
          else
            echo "tag=${{ inputs.image_tag || github.sha }}" >> "$GITHUB_OUTPUT"
          fi
      - id: build
        uses: docker/build-push-action@v6
        with:
          context: .
          file: Dockerfile
          platforms: linux/amd64
          push: true
          tags: |
            ${{ env.IMAGE }}:${{ steps.tag.outputs.tag }}
            ${{ env.IMAGE }}:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max
          provenance: true
          sbom: true
          build-args: PY_VERSION=3.12
      - uses: actions/attest-build-provenance@v4
        with:
          subject-name: ${{ env.IMAGE }}
          subject-digest: ${{ steps.build.outputs.digest }}
          push-to-registry: true
      - uses: superfly/flyctl-actions/setup-flyctl@master
        with: { version: "0.4.x" }
      - env: { FLY_API_TOKEN: ${{ secrets.FLY_API_TOKEN }} }
        run: |
          flyctl deploy --app hypekr-bot \
            --image "${{ env.IMAGE }}@${{ steps.build.outputs.digest }}" \
            --strategy rolling --wait-timeout 300
      - run: |
          for i in 1 2 3 4 5; do
            curl -fsS https://hypekr-bot.fly.dev/healthz && break
            sleep 10
          done
```

### `deploy-analyzer.yml` — GHCR + Streamlit Cloud

```yaml
name: Deploy analyzer → GHCR (+ Streamlit Cloud)
on:
  push:
    branches: [main]
    paths: ["apps/analyzer/**", "Dockerfile.analyzer", ".github/workflows/deploy-analyzer.yml"]
  workflow_dispatch:

concurrency: { group: deploy-analyzer, cancel-in-progress: true }
permissions: { contents: read, packages: write, id-token: write, attestations: write }

jobs:
  build-push:
    runs-on: ubuntu-latest
    timeout-minutes: 20
    env: { IMAGE: ghcr.io/${{ github.repository }}/analyzer }
    outputs: { digest: ${{ steps.build.outputs.digest }}, tag: ${{ steps.meta.outputs.version }} }
    steps:
      - uses: actions/checkout@v4
        with: { persist-credentials: false }
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with: { registry: ghcr.io, username: ${{ github.actor }}, password: ${{ secrets.GITHUB_TOKEN }} }
      - id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.IMAGE }}
          tags: |
            type=raw,value=latest,enable={{is_default_branch}}
            type=sha,format=short
            type=ref,event=branch
      - id: build
        uses: docker/build-push-action@v6
        with:
          context: .
          file: Dockerfile.analyzer
          platforms: linux/amd64,linux/arm64
          push: true
          tags:   ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: |
            type=gha
            type=registry,ref=${{ env.IMAGE }}:buildcache
          cache-to: |
            type=gha,mode=max
            type=registry,ref=${{ env.IMAGE }}:buildcache,mode=max
          provenance: true
          sbom: true
      - uses: actions/attest-build-provenance@v4
        with:
          subject-name: ${{ env.IMAGE }}
          subject-digest: ${{ steps.build.outputs.digest }}
          push-to-registry: true

  streamlit-cloud-redeploy:
    needs: build-push
    if: vars.STREAMLIT_CLOUD_ENABLED == 'true'
    runs-on: ubuntu-latest
    timeout-minutes: 3
    steps:
      - env: { HOOK: ${{ secrets.STREAMLIT_CLOUD_HOOK }} }
        run: curl -fsS -X POST "$HOOK"
```

### `test-matrix.yml` — Python × OS

```yaml
name: Test matrix
on:
  schedule: [ { cron: "0 5 * * 1" } ]
  pull_request:
    paths: ["src/**", "tests/**", "pyproject.toml", "uv.lock"]
  workflow_dispatch:

concurrency: { group: matrix-${{ github.ref }}, cancel-in-progress: true }
permissions: { contents: read }

jobs:
  matrix:
    runs-on: ${{ matrix.os }}
    timeout-minutes: 20
    strategy:
      fail-fast: false
      matrix:
        python: ["3.11", "3.12", "3.13"]
        os: [ubuntu-latest, macos-latest]
        include:
          - python: "3.12"
            os: [self-hosted, Linux, ARM64]
        exclude:
          - python: "3.13"
            os: macos-latest
    steps:
      - uses: actions/checkout@v4
        with: { persist-credentials: false }
      - uses: astral-sh/setup-uv@v6
        with: { version: "0.5.x", python-version: ${{ matrix.python }}, enable-cache: true }
      - run: uv sync --frozen
      - run: uv run pytest -n auto --maxfail=3 -q
      - if: failure()
        uses: actions/upload-artifact@v4
        with: { name: logs-${{ matrix.os }}-${{ matrix.python }}, path: .pytest_cache, retention-days: 3 }
```

### Self-hosted runners: Oracle A1 ARM64 + Hetzner CX32

**Why:** GitHub's free 2,000 min/mo dies on one ML nightly run; Oracle A1 is 4 OCPU/24 GB free; Hetzner CX32 is ~€6.30/mo; both give you local access to Langfuse and DuckDB on a Tailscale LAN. **Never attach a self-hosted runner to a public repo** — fork PRs execute arbitrary code. Reserve self-hosted runners for `kfish` and `hypekr-bot`; use GitHub-hosted for `polymarket-oracle-risk`.

```bash
# On the runner VM (Ubuntu 24.04)
sudo adduser --system --group --home /opt/gha gha
sudo usermod -L gha
sudo ufw default deny incoming && sudo ufw default allow outgoing
sudo ufw allow 22/tcp && sudo ufw enable

RUNNER_VERSION=2.325.0
cd /opt/gha && sudo -u gha mkdir -p actions-runner && cd actions-runner
sudo -u gha curl -fsSL \
  "https://github.com/actions/runner/releases/download/v${RUNNER_VERSION}/actions-runner-linux-arm64-${RUNNER_VERSION}.tar.gz" \
  -o runner.tar.gz
sudo -u gha tar xzf runner.tar.gz && sudo -u gha rm runner.tar.gz
# Get registration token from Repo → Settings → Actions → Runners → New
sudo -u gha ./config.sh \
  --url  https://github.com/ksk5429/kfish \
  --token "$REG_TOKEN" \
  --name oracle-a1-tokyo-01 \
  --labels self-hosted,Linux,ARM64,docker,duckdb,tokyo \
  --ephemeral --disableupdate --unattended --replace
```

Systemd unit `/etc/systemd/system/gha-runner@.service`:

```ini
[Unit]
Description=GitHub Actions runner (%i)
After=network-online.target docker.service
Wants=network-online.target

[Service]
Type=simple
User=gha
Group=gha
WorkingDirectory=/opt/gha/actions-runner
ExecStart=/opt/gha/actions-runner/run.sh --once
Restart=always
RestartSec=15
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/opt/gha /var/lib/gha /tmp
ProtectKernelTunables=true
ProtectKernelModules=true
ProtectControlGroups=true
LockPersonality=true
CapabilityBoundingSet=CAP_CHOWN CAP_DAC_OVERRIDE CAP_FOWNER CAP_SETGID CAP_SETUID
LimitNOFILE=524288
MemoryMax=22G
CPUQuota=380%
Environment=RUNNER_ALLOW_RUNASROOT=0
Environment=ACTIONS_RUNNER_HOOK_JOB_COMPLETED=/opt/gha/hooks/post-job.sh

[Install]
WantedBy=multi-user.target
```

Use `--ephemeral` for CI matrix jobs (clean environment every run) and a persistent runner for `nightly.yml` (DuckDB warm-up is expensive). Post-job hook wipes `_temp/` and stale Docker layers. Don't add the runner user to the `docker` group — use rootless Docker instead.

### Secret management

| Where | For what | Masked? | Reviewer gate? |
|---|---|---|---|
| **OIDC (no secret)** | PyPI, AWS, GCP, Azure, Cloudflare R2 (mid-2025+) | N/A | N/A |
| Environment secrets (`environment: production`) | `FLY_API_TOKEN`, GHCR tokens, sensitive deploy creds | yes | yes — required reviewers / branch filters |
| Repo secrets | last-resort static creds (`SAFETY_API_KEY`, Gitleaks license) | yes | no |
| Repo variables (`vars.X`) | non-sensitive config: region, bucket URI, flags | no | no |

Rotate static tokens every 90 days (B2 keys, Doppler SA). Everything else goes OIDC. For in-repo encrypted secrets (`.env.enc.yaml` for Langfuse local-dev keys), SOPS + age works nicely — the only GitHub secret becomes the age private key.

---

## 4. Reproducible builds and supply chain

### uv as the single source of truth

**Pick uv in 2026.** It subsumes pip-tools, Poetry, pyenv, pipx, and rye (rye merged into uv in 2024). `uv lock` produces a cross-platform lockfile with hashes; `uv sync --frozen` fails on any drift in CI (use `--locked` to fail if the lockfile would change). Pin Python patch-level in `.python-version` (`3.12.8`). Use `dependency-groups` (PEP 735) for dev deps rather than extras; they're portable to any compliant tool. `SOURCE_DATE_EPOCH=$(git log -1 --pretty=%ct)` in wheel-building jobs produces byte-identical artifacts. For private ML repos, `hatch-vcs` with `local_scheme = "no-local-version"` gives every commit a unique `0.3.1.devN` version for model-artifact tracking in DuckDB.

### Reproducible multi-stage Dockerfile

```dockerfile
# syntax=docker/dockerfile:1.10
FROM python:3.12-slim-bookworm@sha256:<pin-weekly> AS base
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1 \
    UV_COMPILE_BYTECODE=1 UV_LINK_MODE=copy \
    UV_PYTHON_DOWNLOADS=never UV_PROJECT_ENVIRONMENT=/app/.venv \
    PATH="/app/.venv/bin:$PATH"

FROM base AS builder
COPY --from=ghcr.io/astral-sh/uv:0.5.14@sha256:<pin> /uv /uvx /usr/local/bin/
RUN apt-get update && apt-get install -y --no-install-recommends \
      build-essential ca-certificates git && rm -rf /var/lib/apt/lists/*
WORKDIR /app
# Deps layer — cache-mount + bind-mount means no source change invalidates
RUN --mount=type=cache,target=/root/.cache/uv \
    --mount=type=bind,source=uv.lock,target=uv.lock \
    --mount=type=bind,source=pyproject.toml,target=pyproject.toml \
    uv sync --locked --no-install-project --no-dev
# Project layer
COPY . /app
RUN --mount=type=cache,target=/root/.cache/uv uv sync --locked --no-dev

FROM base AS runtime
RUN groupadd --system --gid 1001 app && \
    useradd --system --uid 1001 --gid app --home-dir /app --shell /usr/sbin/nologin app && \
    install -d -o app -g app /app /app/data /app/logs
COPY --from=builder --chown=app:app /app /app
WORKDIR /app
USER app
HEALTHCHECK --interval=30s --timeout=5s --start-period=20s --retries=3 \
    CMD python -c "import sys; sys.exit(0)"
CMD ["python", "-m", "kfish_core"]
```

Pin base images by `@sha256:...` digest; Dependabot Docker updates will propose digest bumps weekly. Distroless tempts on size but breaks debugging for a solo dev — slim is the right default. Verify reproducibility with `diffoci` between two builds on different runners. Use `cache-from: type=gha` + `type=registry` for fast warm builds in CI.

### SBOM, attestations, SLSA

You're already at **SLSA Level 2** with the workflows above: builds happen in GitHub-hosted runners, provenance via `actions/attest-build-provenance@v4`, Sigstore-signed attestations for every PyPI wheel and container image. SLSA 3 requires a hardened, isolated builder service — not realistic for a solo dev without extra infra. That's fine. PEP 740 PyPI attestations auto-generate when `pypa/gh-action-pypi-publish` runs with `id-token: write`. Verify any downloaded artifact with `gh attestation verify ./dist/*.whl --repo ksk5429/polymarket-oracle-risk`. syft generates SBOMs in SPDX (GitHub-preferred) or CycloneDX (broader tooling); default to SPDX.

### Dependency supply chain

Use **Renovate for routine version updates** (one PR per grouped update across all three repos via a shared preset) and **Dependabot for security alerts** (zero-setup GitHub Advisory Database integration). Add a 24-hour cooldown to both — the September 2025 axios incident showed how fast malicious packages propagate through auto-merge. `pip-audit --strict` in CI catches known CVEs. OSSF Scorecard runs weekly; target ≥ 7.5/10 on the public repo (quick wins: pinned action SHAs, signed releases, SECURITY.md, branch protection). `trivy` for container images fails the build on HIGH CVEs.

### Data pipeline reproducibility

For DuckDB + Parquet pipelines: **dbt-duckdb** is the right fit for the calibration SQL layer (reproducible, testable, version-controlled transformations). DVC is worth the complexity only if you're versioning multi-GB datasets; for calibration snapshots, nightly `COPY TO (FORMAT PARQUET, PARTITION_BY)` + B2/R2 sync is simpler and sufficient. lakeFS and Nix are both overkill for solo dev — skip them until a second contributor joins.

### Git history hygiene

Before first push and before any handoff, scan history: `gitleaks detect --log-opts="--all" --redact`. To remove a leaked secret, use **git-filter-repo** (BFG is legacy): `git-filter-repo --replace-text replacements.txt` then force-push the mirror. **History rewrite alone is insufficient** — always rotate the leaked credential immediately, because old clones still contain it.

---

## 5. ADRs and documentation

### MADR 4, lightly tooled

MADR 4.0.0 (Sept 2024) is the current standard. Store ADRs in `docs/decisions/` (not `docs/adr/`), use `adr-tools` CLI (`adr new -s 9 "..."` auto-supersedes ADR-9), skip log4brains — for three repos, plain Markdown rendered by mkdocs-material is enough. Claude Code can scaffold ADRs from a one-line prompt; that subsumes most tooling. ADRs are **immutable** except for `status:` — supersede with new numbers rather than editing.

### The ten ADRs to write in weeks 4–6

| # | Title | Decision in one line |
|---|---|---|
| 0001 | Hybrid repo structure | Private `kfish` monorepo + public `polymarket-oracle-risk` |
| 0002 | DuckDB for calibration | Embedded columnar, single-writer, Parquet interop |
| 0003 | aiogram 3 over python-telegram-bot | Async-native, FSM, matches asyncio swarm |
| 0004 | Per-user agent wallets (HIP-4) | Trust boundary at protocol layer; leaked key = one user |
| 0005 | Claude 4.7 primary + OpenAI fallback | Via litellm; evaluator uses opposite provider for independence |
| 0006 | uv over Poetry/pip-tools | 10-100× faster resolves; PEP 621/735 compliant |
| 0007 | Self-hosted Langfuse on Oracle A1 | Trading data stays in-house; zero marginal cost |
| 0008 | Isotonic + Venn-Abers calibration | Distribution-free, small-sample valid |
| 0009 | Hybrid cloud: Oracle A1 + Fly.io NRT | Always-on services free; bot edge latency in APAC |
| 0010 | Trunk-based with signed commits | Required signatures + Scorecard-friendly |

Each follows the MADR 4 template with Context / Decision Drivers / Considered Options / Decision Outcome / Consequences / Confirmation. Keep them 100-250 words; link them from commits and PR descriptions via `Refs ADR-NNNN`.

### Documentation stack

**mkdocs-material + mkdocstrings + awesome-pages + gen-files** is the 2026 gold standard for Python. For the public analyzer, deploy to GitHub Pages; for private repos, keep `mkdocs serve` local only. **Mermaid in Markdown** is enough for architecture diagrams at your scale — reach for Structurizr DSL only when you have ≥ 8 services needing a true C4 model.

Key files in every repo: `README.md` (badges row: CI, coverage, PyPI version, Python versions, license, Ruff, pre-commit, OSSF Scorecard for public; DOI via Zenodo for citeable release), `CONTRIBUTING.md`, `SECURITY.md` (private vulnerability reporting link), `CODE_OF_CONDUCT.md` (Contributor Covenant 2.1), `CITATION.cff` (CFF 1.2.0 — GitHub auto-renders a "Cite this repository" widget), `LICENSE` (MIT for public, explicit `Copyright (c) 2026 <Name>. All rights reserved. UNLICENSED.` for private).

`CHANGELOG.md` follows Keep a Changelog 1.1.0, auto-generated by release-please. Hand-author release notes only for major versions with breaking changes — drop them in `docs/releases/v2.0.md` with a migration guide table.

---

## 6. Day-to-day operations and handoff

### Workflow rhythm

Branch names: `feat/`, `fix/`, `chore/`, `docs/`, `refactor/`, `test/`, `perf/` (matches Conventional Commits types). Draft PRs first for early CI feedback, then `gh pr merge --auto --squash --delete-branch`. `cz commit` for interactive conventional commits when you want guidance. `CLAUDE.md` at the repo root describes conventions and ADR locations — Claude Code reads it automatically.

`.vscode/settings.json` essentials:

```jsonc
{
  "python.defaultInterpreterPath": ".venv/bin/python",
  "[python]": {
    "editor.defaultFormatter": "charliermarsh.ruff",
    "editor.formatOnSave": true,
    "editor.codeActionsOnSave": {
      "source.fixAll.ruff": "explicit",
      "source.organizeImports.ruff": "explicit"
    }
  },
  "git.enableCommitSigning": true,
  "files.eol": "\n",
  "editor.rulers": [100],
  "terminal.integrated.defaultProfile.windows": "Ubuntu (WSL)"
}
```

### Dependency update cadence

Patch bumps auto-merge after CI green + 24 h cooldown. Minor bumps auto-merge for dev-dependencies only; manual review for runtime. Major bumps always manual — read upstream release notes, grep for affected symbols, write a 1-paragraph PR-body note for anything in `aiogram`, `anthropic`, `openai`, `duckdb`, or the Hyperliquid SDK. Consider an ADR amendment if behaviour changes materially.

### Backups and disaster recovery

Git repos mirror to GitLab via `pixta-dev/repository-mirroring-action@v1` on every push to main; optionally also to self-hosted Forgejo on Oracle A1. DuckDB nightly: `EXPORT DATABASE` → tar → age-encrypt → B2 + R2 (R2 as hot-restore, B2 as cold). Langfuse: weekly logical Postgres dump + Clickhouse parts backup via `clickhouse-backup`. Fly.io volumes: `flyctl volumes snapshots create` nightly + 30-day retention. **Quarterly restore drill** (target RPO 24 h / RTO 4 h): spin fresh Oracle tenancy via Terraform, restore from R2, verify canonical-hash smoke test, tear down. Document wall-clock in `runbooks/dr-log.md`.

### Handoff preparation (even as solo dev)

Even if you never hand off, write as though next quarter's collaborator joins tomorrow. Every repo gets `ONBOARDING.md` (day-1 checklist), `runbooks/deploy.md`, `runbooks/restore.md`, `runbooks/rotate-keys.md`, `runbooks/incident.md`, `docs/architecture.md` (one-page Mermaid C4 context diagram), and `.env.example` with every variable commented. Before any repo ever becomes accessible to anyone else: `gitleaks detect --log-opts="--all" --redact` must return clean; rotate any credential that appears; force-push the sanitized mirror.

---

## 7. The minimum viable advanced setup (≤1 hour)

If you can only do part of this weekend, do this and nothing else — it captures ~80% of the value:

1. **Install toolchain (5 min).** `curl -LsSf https://astral.sh/uv/install.sh | sh`, `sudo apt install gh`, `uv tool install pre-commit commitizen`.
2. **Global git + SSH signing (3 min).** Generate Ed25519, register in GitHub signing-key slot, `git config --global commit.gpgsign true` + `gpg.format ssh`.
3. **Create the three repos (5 min).** The `gh repo create` commands from §1.
4. **Per-repo scaffold (10 min each).** `uv init`, drop in `pyproject.toml`, `.pre-commit-config.yaml`, `.gitignore`, `.editorconfig`, `.python-version`.
5. **One CI workflow (5 min).** A single `ci.yml` running `ruff check`, `ruff format --check`, `mypy`, `pytest --cov`. Skip the matrix and security jobs initially.
6. **Dependabot (3 min).** Minimal `dependabot.yml`: pip + github-actions + docker, weekly, no groups.
7. **First signed commit and push (5 min).** `pre-commit install --install-hooks`, `git commit -S`, watch CI.
8. **Minimal branch protection (3 min).** `gh api` with `required_status_checks`, `enforce_admins=true`, `required_signatures=true`, `allow_force_pushes=false`, `required_approving_review_count=0`.

Stop here. You now have: reproducible env, formatted/linted/typed/secured code path, green CI, signed commits enforced, automated dep updates, a release pipeline primed to run on tag. Everything else — self-hosted runners, SLSA attestations, mkdocs site, CodeQL, Scorecard, matrix tests — is strict polish you can add week-by-week.

## 8. The 13-week rollout

| Week | Goal | Done when |
|---|---|---|
| **1** | Bootstrap | Three green CI badges; signed commits enforced; branch protection live |
| **2** | Self-hosted runner online | Oracle A1 runner shows green; Dependabot has merged its first PR |
| **3** | First release cut | Three `v0.1.0` tags; one TestPyPI artifact downloadable |
| **4** | ADRs + docs live | ≥ 2 ADRs per repo in MADR 4 format; oracle analyzer docs at ksk5429.github.io |
| **5** | Matrix + coverage | py3.11/3.12/3.13 × ubuntu/macOS green; ≥ 70% coverage |
| **6** | Fly.io staging | `/ping` on hypekr-bot staging returns < 1 s; testnet Hyperliquid key loaded |
| **7** | Langfuse + Claude path | First swarm-agent trace tree with ≥ 3 spans in Langfuse UI |
| **8** | First TestPyPI publish + Streamlit Cloud | `pip install -i test.pypi.org …` works; dashboard opens publicly |
| **9** | Security hardening | CodeQL + gitleaks + trufflehog + trivy gates live; cosign image signing |
| **10** | Observability | Langfuse cost dashboard; Prometheus `/metrics`; Fly logs → BetterStack |
| **11** | Performance + load | pytest-benchmark baseline tracked; bot p95 < 300 ms under 10 rps |
| **12** | Production cutover | hypekr-bot on mainnet with one real trade under size cap; 2-minute rollback drill done |
| **13** | PyPI v1.0.0 | `pip install polymarket-oracle-risk==1.0.0` from prod PyPI; retro committed |

---

## Conclusion: the boring stack is the right stack

The architecture above is deliberately unexciting. It picks mature, standards-based tools — PEP 621, PEP 735, MADR 4, CFF 1.2, Conventional Commits, Keep a Changelog, SemVer 2.0, SLSA Level 2, OSSF Scorecard — so that when you pivot the analyzer toward a JOSS submission, invite a collaborator, or spin off the bot, no documentation infrastructure has to change. **The two non-obvious wins** are (a) the hybrid repo structure, which aligns GitHub's visibility primitive with your actual public/private semantic boundary, and (b) the solo-dev branch-protection trick of requiring status checks + signatures + linear history instead of reviews — giving you all the enforcement of a multi-person workflow without the self-approval paradox.

Three things to watch in 2026 and push back against if they drift toward you: self-hosted runners on anything public (Shai-Hulud persistence is real), auto-merged major bumps without a 24 h cooldown (the axios-style supply-chain attack vector), and dual version sources of truth (`release-please` writing `pyproject.toml` **or** `hatch-vcs` reading tags — never both on the same repo). Everything else in this guide is recoverable if you pick the "wrong" option; those three are the ones that bite silently.

Ship the minimum viable setup in §7 this weekend. Layer in self-hosted runners, attestations, and docs over the following 13 weeks using §8 as the Gantt. By week 13, the trading stack is private and signed, the Telegram bot is deployed to NRT with builder-fee logic isolated behind strict mypy, and `polymarket-oracle-risk` v1.0.0 is live on PyPI with a green Sigstore-verified badge and a Zenodo DOI ready for your thesis bibliography.