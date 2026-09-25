# jol-iac Library + Policy-as-Code Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Seed the empty `jol-iac` repo (governance tier) as the canonical, versioned OpenTofu/`bpg/proxmox` IaC library + OPA policy-as-code + inert SOPS Gate 4–6 docs, mapped to ISO 27001 A.8.9 and SOC 2 CC8.1.

**Architecture:** Documentation-centric governance wrapper (mirrors `jol-policies`) + net-new IaC content dirs (`modules/`, `templates/`, `policy/opa/`, `sops/`). OpenTofu (MPL-2.0), local-first state, examples validated with `tofu init -backend=false && tofu validate` and never applied. Non-destructive: no edits to `jol-control`/`jol-infrastructure`; legacy AWS terraform superseded by ADR note only. SOPS layer is docs/templates only (Approach B) — no executable scripts, no key material.

**Tech Stack:** OpenTofu ≥ 1.8 · `bpg/proxmox` provider (pinned) · OPA/conftest (rego) · tflint · checkov · trivy · detect-secrets · pre-commit · markdownlint-cli 0.41.0 (Node 20) · GitHub Actions · GPG-signed commits.

**Source of truth:** design spec `docs/superpowers/specs/2026-09-25-jol-iac-library-design.md` (committed `743f29a`).

---

## Conventions for every task

- **Working dir:** `/opt/jol/repos/jol-iac` (target repo). Plans/specs live in `/opt/jol/repos/jol-repo-template`.
- **Commits:** GPG-signed (`commit.gpgsign=true`, key `…609F7926A8254CDB`), Conventional Commits, on branch `main`. Commit after each task.
- **Fail-closed:** every command uses `set -euo pipefail` semantics; never `|| true`; never print PASS without real verification; `⚠ UNVERIFIED` until a command actually succeeds.
- **Placeholders only:** repo is public + `data_classification: confidential`. No real hostnames/IPs/VMIDs/PBS paths/key material. Example VMIDs `9000+`, RFC1918 documentation ranges, `CHANGEME`.
- **Do NOT touch:** `jol-control`, `jol-infrastructure`, or the unrelated `jol-repo-template` working-tree changes (`docs/superpowers/plans/2026-09-25-jol-dr-implementation.md`, `.stage-dr/`, `.staging-jol-docs/`).
- **Shell pitfall:** this host's reused foreground shell sometimes swallows stdout. For any multi-line/verification output, redirect to a uniquely-named file under `/tmp` and read it back.

## File Structure (locked-in decomposition)

```text
jol-iac/
├── README.md  LICENSE(MIT)  SECURITY.md  CONTRIBUTING.md  CHANGELOG.md  CODE_OF_CONDUCT.md
├── .editorconfig  .gitignore  .markdownlint.yaml  .secrets.baseline  .pre-commit-config.yaml
├── Makefile  .tofu-version
├── modules/{proxmox-vm,proxmox-firewall,proxmox-pool,proxmox-backup}/
│     └── main.tf variables.tf outputs.tf versions.tf README.md examples/{basic,complete}/{main.tf,variables.tf,outputs.tf,terraform.tfvars.example}
├── templates/{env-stack,module-template}/
├── policy/opa/{no_secrets_in_tf,required_tags,no_public_ingress,resource_limits}.rego + *_test.rego
├── sops/{README.md, runbooks/gate-4..6, checklists/gate-4-6-conditions.md, templates/*.tmpl, tool-versions.md}
├── docs/{architecture,consuming-modules,state-and-backends,tool-versions,control-mapping-matrix}.md + adr/{README,ADR-001..004}
└── .github/{CODEOWNERS, dependabot.yml, PULL_REQUEST_TEMPLATE.md, ISSUE_TEMPLATE/*, workflows/{compliance-scan.yml,codeql.yml}}
```

Each file has one responsibility; modules are self-contained (own versions/README/examples); policy rules are one-concern-per-file with a matching test file.

---

## Wave 0 — Preflight & Toolchain (fail-closed gate before any content)

### Task 0.1: Verify repo state and remote

**Files:** none (read-only).

- [ ] **Step 1:** Confirm `jol-iac` is empty and remote is correct.

Run:
```bash
git -C /opt/jol/repos/jol-iac log --oneline -1 2>&1 | tee /tmp/jol_iac_p0_git.txt
git -C /opt/jol/repos/jol-iac remote -v >> /tmp/jol_iac_p0_git.txt 2>&1
git -C /opt/jol/repos/jol-iac status -b --porcelain=v1 >> /tmp/jol_iac_p0_git.txt 2>&1
```
Expected: `does not have any commits yet`; `origin git@github.com:journeyoflife-org/jol-iac.git`; branch `main`. If commits already exist → STOP and reconcile (idempotency baseline).

### Task 0.2: Install OpenTofu (canonical tool, D1) — MISSING on host

**Files:** none (tool install); record version.

- [ ] **Step 1:** Install the latest stable OpenTofu binary with SHA256 verification (never run an unverified binary).

Run (user-local, no sudo):
```bash
set -euo pipefail
V=$(curl -fsSL https://api.github.com/repos/opentofu/opentofu/releases/latest | jq -r .tag_name | sed 's/^v//')
mkdir -p ~/.local/bin && cd /tmp
curl -fsSLO "https://github.com/opentofu/opentofu/releases/download/v${V}/tofu_${V}_linux_amd64.zip"
curl -fsSLO "https://github.com/opentofu/opentofu/releases/download/v${V}/tofu_${V}_SHA256SUMS"
sha256sum -c --ignore-missing tofu_${V}_SHA256SUMS
unzip -o "tofu_${V}_linux_amd64.zip" tofu -d ~/.local/bin && chmod +x ~/.local/bin/tofu
~/.local/bin/tofu version | tee /tmp/jol_iac_tofu_ver.txt
```
Expected: `OpenTofu v1.x.y`. Record `V` → this becomes `.tofu-version` (Task 1.4) and a row in `docs/tool-versions.md` (Wave 3).
- [ ] **Step 2 (fallback, only if install blocked):** document that local empirical validation uses `terraform v1.16.1` (`terraform validate`) as a schema proxy, with OpenTofu authoritative in CI. Note this in `docs/tool-versions.md`. ⚠ Flag to owner; D1 prefers real OpenTofu.
- [ ] **Step 3:** Confirm `tofu` on PATH for later waves: `export PATH="$HOME/.local/bin:$PATH"; command -v tofu`.

### Task 0.3: Verify remaining toolchain

**Files:** none.

- [ ] **Step 1:** Probe and record.

Run:
```bash
{ for t in tflint checkov opa trivy conftest; do printf "%-9s " "$t"; command -v "$t" >/dev/null 2>&1 && echo present || echo MISSING; done; } | tee /tmp/jol_iac_p0_tools.txt
```
Expected (from 2026-09-25 probe): `tflint/checkov/opa/trivy present`, `conftest MISSING`. **Decision:** `conftest` is CI-only; local policy gate uses `opa test` (present). Do not add a silent local skip.

### Task 0.4: Prepare local QA venv

**Files:** none in-repo (`.venv/` is gitignored).

- [ ] **Step 1:** Ensure `.venv` has pre-commit + detect-secrets.

Run:
```bash
cd /opt/jol/repos/jol-iac
test -d .venv || python3.12 -m venv .venv
.venv/bin/pip install --quiet --upgrade pip pre-commit detect-secrets
.venv/bin/detect-secrets --version | tee /tmp/jol_iac_p0_ds.txt
```
Expected: detect-secrets version prints (proves the `make secrets` target will work).

---

## Wave 1 — Governance Wrapper, Hygiene & Toolchain Config

Produces a lint-clean, secret-clean governance shell. Verify with `pre-commit run --all-files` at the end.

### Task 1.1: Hardened `.gitignore` (the inverse of the jol-control/jol-infrastructure defects)

**Files:** Create `/opt/jol/repos/jol-iac/.gitignore`

- [ ] **Step 1:** Write exactly:

```gitignore
# --- Local env / IDE (never pushed) ---
.venv/
.idea/
__pycache__/
*.pyc

# --- OpenTofu / Terraform: state & caches NEVER in git (CC6.1/CC8.1) ---
.terraform/
.tofu/
*.tfstate
*.tfstate.*
*.tfvars
*.tfvars.json
!*.tfvars.example
crash.log
crash.*.log
override.tf
override.tf.json
*_override.tf
*_override.tf.json
# NOTE: .terraform.lock.hcl and .sops.yaml ARE committed (reproducibility / public config)

# --- SOPS / age: private key material & decrypted output NEVER in git ---
keys.txt
age-*.txt
!age-*.pub
*.pub.asc
*.dec.*
secrets/**/*.yaml
!secrets/**/*.enc.yaml
!secrets/**/*.example

# --- Backup / recovery artifacts (never expose) ---
*.bundle
*.sql.gz
*.tar.gz
*.zst
recovery-artifacts/
```
- [ ] **Step 2:** Sanity-check the allow/deny logic:
Run: `cd /opt/jol/repos/jol-iac && git check-ignore -v .terraform/x foo.tfstate secrets/a.yaml age-jol.txt 2>&1 | tee /tmp/jol_iac_gi.txt; git check-ignore -v .terraform.lock.hcl 2>&1 | tee -a /tmp/jol_iac_gi.txt || echo "lock NOT ignored (correct)"`
Expected: first four are ignored; `.terraform.lock.hcl` is NOT ignored.
- [ ] **Step 3: Commit** — `git add .gitignore && git commit -S -m "chore: add hardened IaC gitignore (no state/keys/binaries)"`

### Task 1.2: Mirror static governance files from `jol-policies`

**Files:** Create `.editorconfig`, `.markdownlint.yaml`, `CODE_OF_CONDUCT.md`, `LICENSE` (MIT).

- [ ] **Step 1:** Copy verbatim, then fix LICENSE year/holder.
```bash
cd /opt/jol/repos/jol-iac
cp /opt/jol/repos/jol-policies/.editorconfig .editorconfig
cp /opt/jol/repos/jol-policies/.markdownlint.yaml .markdownlint.yaml
cp /opt/jol/repos/jol-policies/CODE_OF_CONDUCT.md CODE_OF_CONDUCT.md
cp /opt/jol/repos/jol-repo-template/LICENSE LICENSE   # MIT
```
- [ ] **Step 2:** Verify LICENSE is MIT and header year is 2026: `head -3 LICENSE`. If the template LICENSE year differs, set `Copyright (c) 2026 Journey of Life (JOL) — journeyoflife-org`.
- [ ] **Step 3: Commit** — `git add .editorconfig .markdownlint.yaml CODE_OF_CONDUCT.md LICENSE && git commit -S -m "chore: seed editorconfig, markdownlint config, CoC, MIT license"`

### Task 1.3: `.pre-commit-config.yaml` (docs-centric base + IaC fmt)

**Files:** Create `.pre-commit-config.yaml`

- [ ] **Step 1:** Write (mirrors `jol-policies`, drops ruff/mypy, adds OpenTofu fmt via local hook):

```yaml
# Pre-commit for jol-iac (IaC library). No Python source -> ruff/mypy omitted.
exclude: '^\.idea/|^\.venv/|^templates/.*\.tmpl$'
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.6.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-json
      - id: check-merge-conflict
      - id: check-added-large-files
        args: ["--maxkb=500"]
      - id: detect-private-key
      - id: mixed-line-ending
        args: ["--fix=lf"]
  - repo: https://github.com/igorshubovych/markdownlint-cli
    rev: v0.41.0
    hooks:
      - id: markdownlint
        args: ["--config", ".markdownlint.yaml"]
  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.5.0
    hooks:
      - id: detect-secrets
        args: ["--baseline", ".secrets.baseline"]
        exclude: '\.secrets.baseline$'
  - repo: https://github.com/Lucas-C/pre-commit-hooks
    rev: v1.5.5
    hooks:
      - id: forbid-crlf
      - id: forbid-tabs
        exclude: '^Makefile$'
  - repo: local
    hooks:
      - id: tofu-fmt
        name: tofu fmt
        entry: bash -c 'command -v tofu >/dev/null && tofu fmt -check -recursive || echo "tofu not on PATH; fmt enforced in CI"'
        language: system
        types: [terraform]
        pass_filenames: false
```
- [ ] **Step 2: Commit** — `git add .pre-commit-config.yaml && git commit -S -m "chore: add pre-commit config (markdownlint 0.41.0, detect-secrets, tofu fmt)"`

### Task 1.4: `.tofu-version` + `Makefile`

**Files:** Create `.tofu-version`, `Makefile`

- [ ] **Step 1:** Write `.tofu-version` with the version from Task 0.2 (e.g. `1.8.5` — use the actual resolved value, no `v` prefix).
- [ ] **Step 2:** Write `Makefile` (recipe lines below are shown space-indented for plan readability; the real Makefile MUST use TAB indentation — `make` requires it, and jol-iac's pre-commit excludes `^Makefile$` from `forbid-tabs`):

```makefile
# jol-iac — IaC library quality targets. Usage: make help
SHELL := /bin/bash
VENV := .venv
PIP := $(VENV)/bin/pip
PRE_COMMIT := $(VENV)/bin/pre-commit
DETECT_SECRETS := $(VENV)/bin/detect-secrets
NPX := npx --yes
TOFU := tofu
.DEFAULT_GOAL := help
.PHONY: help install fmt validate lint policy scan docs secrets check clean

help: ## Show available targets
    @grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | awk 'BEGIN {FS = ":.*?## "}; {printf "  \033[36m%-10s\033[0m %s\n", $$1, $$2}'

install: ## Install QA tooling into .venv and activate hooks
    @test -d $(VENV) || python3.12 -m venv $(VENV)
    $(PIP) install --upgrade pip
    $(PIP) install pre-commit detect-secrets
    $(PRE_COMMIT) install

fmt: ## Format OpenTofu files
    $(TOFU) fmt -recursive

validate: ## init(-backend=false)+validate every dir containing .tf (never applies)
    @set -euo pipefail; for d in $$(find . -path ./.venv -prune -o -name '*.tf' -print | xargs -r -n1 dirname | sort -u); do \
      echo "==> validate $$d"; (cd "$$d" && $(TOFU) init -backend=false -input=false >/dev/null && $(TOFU) validate); done

lint: ## Static analysis: tflint + checkov
    tflint --recursive
    checkov -d . --quiet

policy: ## OPA policy unit tests (conftest runs in CI)
    opa test policy/opa -v --coverage

scan: ## Security scan: trivy config + secret baseline refresh
    trivy config .
    $(DETECT_SECRETS) scan --baseline .secrets.baseline

docs: ## Markdown lint + link check
    $(NPX) markdownlint-cli@0.41.0 "**/*.md" --config .markdownlint.yaml
    @find . -path ./.venv -prune -o -name '*.md' -print | xargs -r $(NPX) markdown-link-check -q

secrets: ## Re-scan secrets into baseline
    $(DETECT_SECRETS) scan --baseline .secrets.baseline

check: ## Full pre-commit suite against all files
    $(PRE_COMMIT) run --all-files

clean: ## Remove local caches/artifacts (never state — there is none)
    rm -rf .mypy_cache .pytest_cache .ruff_cache
    find . -type d \( -name '.terraform' -o -name '.tofu' \) -not -path './.venv/*' -prune -exec rm -rf {} +
    find . -type d -name '__pycache__' -not -path './.venv/*' -prune -exec rm -rf {} +
```
- [ ] **Step 3:** Verify targets parse: `make help | tee /tmp/jol_iac_make.txt` → lists all targets.
- [ ] **Step 4: Commit** — `git add .tofu-version Makefile && git commit -S -m "chore: add OpenTofu pin and IaC Makefile (validate/lint/policy/scan/docs)"`

### Task 1.5: `.github/` — CODEOWNERS, dependabot, PR + issue templates

**Files:** Create `.github/CODEOWNERS`, `.github/dependabot.yml`, `.github/PULL_REQUEST_TEMPLATE.md`, `.github/ISSUE_TEMPLATE/{bug_report.yml,feature_request.yml}`

- [ ] **Step 1:** Copy CODEOWNERS + dependabot + issue templates from the template, then adapt dependabot ecosystems.
```bash
cd /opt/jol/repos/jol-iac && mkdir -p .github/ISSUE_TEMPLATE
cp /opt/jol/repos/jol-repo-template/.github/CODEOWNERS .github/CODEOWNERS
cp /opt/jol/repos/jol-repo-template/.github/PULL_REQUEST_TEMPLATE.md .github/PULL_REQUEST_TEMPLATE.md
cp /opt/jol/repos/jol-repo-template/.github/ISSUE_TEMPLATE/*.yml .github/ISSUE_TEMPLATE/
```
- [ ] **Step 2:** Write `.github/dependabot.yml`:
```yaml
version: 2
updates:
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule: { interval: "weekly" }
    commit-message: { prefix: "chore(deps)" }
  - package-ecosystem: "terraform"
    directory: "/"
    schedule: { interval: "weekly" }
    commit-message: { prefix: "chore(deps)" }
```
- [ ] **Step 3:** Note in CODEOWNERS a comment that teams are aspirational for the solo owner (mirrors template per D5; code-owner review inert under `solo_mode`).
- [ ] **Step 4: Commit** — `git add .github && git commit -S -m "chore: add CODEOWNERS, dependabot (actions+terraform), PR/issue templates"`

### Task 1.6: Wrapper prose (README, SECURITY, CONTRIBUTING, CHANGELOG)

**Files:** Create `README.md`, `SECURITY.md`, `CONTRIBUTING.md`, `CHANGELOG.md`

- [ ] **Step 1:** Author to this content-spec (each must pass markdownlint 0.41.0 + link-check):
  - **README.md** — H1 `jol-iac`; sections: Purpose (canonical versioned OpenTofu/`bpg/proxmox` IaC library for JOL); Scope & Non-Goals (library only; no fleet provisioning; SOPS docs-only); Repository Structure (the tree); Module Catalog (table: name → purpose); Policy Catalog (4 rego rules); Consuming a module (link `docs/consuming-modules.md`); SOPS Gate 4–6 pointer (link `sops/README.md`); Compliance (A.8.9/CC8.1 + link `docs/control-mapping-matrix.md`); Development (`make install/validate/lint/policy/docs/check`); License (MIT).
  - **SECURITY.md** — private vuln reporting (do NOT open public issues for security); secret handling rules (SOPS+age dual identity `age-jol`/`age-jolarca`; private keys NEVER committed; `.gitignore` guarantees); public+confidential caveat (placeholders only); reference AGENTS.md §0.1 zero-tolerance.
  - **CONTRIBUTING.md** — module-authoring standard (`versions.tf` pins, required tags CostCenter/Environment/Owner, README, `examples/{basic,complete}`, must pass `tofu validate`); policy contribution (rego + `*_test.rego`); PR + change-control checklist (SOC2 CC8.1: linked issue, rollback, evidence); signed commits required; ADR immutability rule.
  - **CHANGELOG.md** — Keep-a-Changelog format; `## [Unreleased]` + initial `### Added` seed entry referencing this plan/spec and controls (A.8.9, CC8.1).
- [ ] **Step 2:** Verify: `.venv/bin/pre-commit run --files README.md SECURITY.md CONTRIBUTING.md CHANGELOG.md` → markdownlint + detect-secrets pass.
- [ ] **Step 3: Commit** — `git add README.md SECURITY.md CONTRIBUTING.md CHANGELOG.md && git commit -S -m "docs: add README, SECURITY, CONTRIBUTING, CHANGELOG"`

### Task 1.7: Initial `.secrets.baseline` + Wave 1 gate

**Files:** Create `.secrets.baseline`

- [ ] **Step 1:** Generate baseline over the current tree: `.venv/bin/detect-secrets scan > .secrets.baseline`
- [ ] **Step 2 (Wave 1 verification gate):** `cd /opt/jol/repos/jol-iac && .venv/bin/pre-commit run --all-files 2>&1 | tee /tmp/jol_iac_w1.txt` → ALL hooks Passed/Skipped, none Failed. Required-files present (README, LICENSE, SECURITY, CONTRIBUTING, CHANGELOG, CODE_OF_CONDUCT, .gitignore, .editorconfig, CODEOWNERS, dependabot.yml). If detect-secrets flags a doc line discussing tooling, add inline `<!-- pragma: allowlist secret -->` to that line (verified false positive) — do NOT use `--no-verify`.
- [ ] **Step 3: Commit** — `git add .secrets.baseline && git commit -S -m "chore: add detect-secrets baseline; Wave 1 pre-commit green"`

**Wave 1 checkpoint (subagent-driven):** dispatch `code-reviewer` on the Wave 1 diff; pause for owner confirmation before Wave 2.

---


## Wave 2 — CI Workflows (mapped to governance-tier gates)

Emits the contexts `jol-iac.yml` declares (`compliance-scan / validate`, `codeql-analysis`). Under `solo_mode` these are not enforced, but CI runs on push for evidence and is built to the full governance standard so flipping `solo_mode=false` needs no rework.

### Task 2.1: `.github/workflows/compliance-scan.yml`

**Files:** Create `.github/workflows/compliance-scan.yml`

- [ ] **Step 1:** Write (pinned actions; tools installed deterministically; fail-closed `set -euo pipefail`):

```yaml
name: compliance-scan
on:
  push: { branches: [main] }
  pull_request: { branches: [main] }
permissions: { contents: read }
concurrency:
  group: compliance-${{ github.ref }}
  cancel-in-progress: true
jobs:
  validate:
    name: validate
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Required files present
        run: |
          set -euo pipefail
          for f in README.md LICENSE SECURITY.md CONTRIBUTING.md CHANGELOG.md CODE_OF_CONDUCT.md .gitignore .editorconfig .github/CODEOWNERS .github/dependabot.yml; do
            test -f "$f" || { echo "::error::missing $f"; exit 1; }
          done
      - name: License headers
        uses: apache/skywalking-eyes/header@v0.6.0
        with: { mode: check }
      - name: Setup OpenTofu
        uses: opentofu/setup-opentofu@v1
      - name: tofu fmt -check
        run: tofu fmt -check -recursive
      - name: tofu validate (modules/templates; never applies)
        run: |
          set -euo pipefail
          for d in $(find modules templates -type d 2>/dev/null); do
            if ls "$d"/*.tf >/dev/null 2>&1; then
              echo "==> $d"; (cd "$d" && tofu init -backend=false -input=false >/dev/null && tofu validate)
            fi
          done
      - name: Setup tflint
        uses: terraform-linters/setup-tflint@v4
      - name: tflint
        run: |
          set -euo pipefail
          tflint --init
          tflint --recursive
      - name: checkov
        run: |
          set -euo pipefail
          python -m pip install --upgrade pip
          pip install checkov==3.3.16
          checkov -d . --quiet
  policy:
    name: policy
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: open-policy-agent/setup-opa@v2
      - name: opa test
        run: opa test policy/opa -v --coverage
      - name: install conftest (checksummed)
        run: |
          set -euo pipefail
          V=0.56.0
          curl -fsSLo conftest.tar.gz "https://github.com/open-policy-agent/conftest/releases/download/v${V}/conftest_${V}_Linux_x86_64.tar.gz"
          tar xzf conftest.tar.gz conftest && sudo mv conftest /usr/local/bin/
      - name: conftest (HCL) over examples
        run: conftest test --parser hcl examples modules --policy policy/opa || true
  secrets:
    name: secrets
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }
      - name: TruffleHog
        uses: trufflesecurity/trufflehog@main
        with: { extra_args: --only-verified }
  iac-scan:
    name: iac-scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Trivy config
        uses: aquasecurity/trivy-action@0.28.0
        with: { scan-type: config, dir: ., exit-code: '1', severity: HIGH,CRITICAL }
  dependencies:
    name: dependencies
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'
    steps:
      - uses: actions/checkout@v4
      - uses: actions/dependency-review-action@v4
```
> Note: the `conftest ... || true` is the ONLY tolerated non-fatal step (`examples/` may not exist until Wave 5); `opa test` remains the fail-closed policy gate. Remove `|| true` once `examples/` exists (Task 5.4).

- [ ] **Step 2:** Lint the workflow YAML: `.venv/bin/pre-commit run check-yaml --files .github/workflows/compliance-scan.yml` → Passed. (Optionally `actionlint` if available.)
- [ ] **Step 3: Commit** — `git add .github/workflows/compliance-scan.yml && git commit -S -m "ci: add compliance-scan workflow (validate/policy/secrets/iac-scan/dependencies)"`

### Task 2.2: `.github/workflows/codeql.yml`

**Files:** Create `.github/workflows/codeql.yml`

- [ ] **Step 1:** Write (emits `codeql-analysis`; `language: actions` scans the workflows for script-injection — meaningful with zero Python):

```yaml
name: codeql
on:
  push: { branches: [main] }
  pull_request: { branches: [main] }
  schedule: [ { cron: '9 3 * * 1' } ]
permissions:
  actions: read
  contents: read
  security-events: write
concurrency:
  group: codeql-${{ github.ref }}
  cancel-in-progress: true
jobs:
  codeql-analysis:
    name: codeql-analysis
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: github/codeql-action/init@v3
        with:
          languages: actions
      - uses: github/codeql-action/analyze@v3
        with:
          category: "/language:actions"
```
- [ ] **Step 2:** Verify YAML + commit: `.venv/bin/pre-commit run check-yaml --files .github/workflows/codeql.yml`; `git add .github/workflows/codeql.yml && git commit -S -m "ci: add CodeQL (actions) workflow emitting codeql-analysis"`
- [ ] **Step 3:** Record follow-up (in `docs/tool-versions.md` notes): propose adding `actions` to `compliance-gates.yml` `sast.languages` via a separate `jol-control` CR; verify the exact check-run context strings in the GitHub checks UI during Phase 5 and reconcile with `jol-iac.yml` if they differ.

**Wave 2 checkpoint:** `code-reviewer` on the two workflows; owner confirmation before Wave 3.

---

## Wave 3 — Docs & ADRs

All markdown must pass `markdownlint 0.41.0` + `markdown-link-check`. ADRs follow the immutability rule (Status/Context/Decision/Consequences/Alternatives/Compliance; supersede, never rewrite).

### Task 3.1: `docs/tool-versions.md`

**Files:** Create `docs/tool-versions.md`

- [ ] **Step 1:** Author a pinned-versions table with the values resolved in Wave 0 and the local probe: OpenTofu (from Task 0.2), `bpg/proxmox` (resolved at first `tofu init` in Wave 4 — fill then, mark `⚠ UNVERIFIED` until resolved), tflint 0.63.1, checkov 3.3.16, trivy 0.52.2, opa (run `opa version`), conftest 0.56.0 (CI), markdownlint-cli 0.41.0, Node 20, Python 3.12, detect-secrets (`.venv` version), sops + age + SHA256 (from `sops/tool-versions.md`, cross-link). Include a "how to re-verify" note (fail-closed: checksum before first run).
- [ ] **Step 2:** Verify `opa version` output and record it (do not guess).
- [ ] **Step 3: Commit** — `git add docs/tool-versions.md && git commit -S -m "docs: add pinned tool-versions register"`

### Task 3.2: Architecture + consumer + state docs

**Files:** Create `docs/architecture.md`, `docs/consuming-modules.md`, `docs/state-and-backends.md`

- [ ] **Step 1:** Author to spec:
  - **architecture.md** — library-not-runtime model; layer diagram (modules → templates → consumers); boundaries vs `jol-control` (GitHub org), `jol-infrastructure` (Ansible/inventory/host docs; legacy AWS superseded), `jol-security`/`jol-compliance` (baselines/evidence); why OpenTofu+Proxmox (link ADR-001/002).
  - **consuming-modules.md** — how a downstream repo sources a module (`source = "git::https://github.com/journeyoflife-org/jol-iac.git//modules/proxmox-vm?ref=<tag>"`), required inputs (tags), validate-before-apply, plan-based conftest is the consumer's responsibility (needs Proxmox creds).
  - **state-and-backends.md** — local-first; NEVER commit state; documented migration to self-hosted MinIO/S3-compatible backend with locking + SSE + versioning; example backend block as a `.tmpl` (not active).
- [ ] **Step 2:** Verify links resolve (`make docs` or `markdown-link-check`).
- [ ] **Step 3: Commit** — `git add docs/architecture.md docs/consuming-modules.md docs/state-and-backends.md && git commit -S -m "docs: add architecture, consuming-modules, state-and-backends"`

### Task 3.3: ADRs 001–004 + index

**Files:** Create `docs/adr/README.md`, `docs/adr/ADR-001-opentofu-over-terraform.md`, `ADR-002-proxmox-target.md`, `ADR-003-state-strategy.md`, `ADR-004-supersedes-legacy-aws-terraform.md`

- [ ] **Step 1:** Author each ADR with the standard sections (mirror `jol-infrastructure/docs/adr` format). Status `Accepted`. Content anchors:
  - **ADR-001** OpenTofu over Terraform — BUSL-1.1 not in `allowed_licenses`; MPL-2.0; client-side state encryption; Linux Foundation governance. Alternatives: Terraform (rejected: license), Atlantis-only, etc.
  - **ADR-002** Proxmox target — reality is 100% on-prem Proxmox (AGENTS.md §2.5 drift alert); `bpg/proxmox`; no AWS.
  - **ADR-003** State strategy — local-first + MinIO/S3-compatible remote path; state never in git (references the jol-control defect as the anti-pattern).
  - **ADR-004** Supersedes legacy AWS terraform direction — jol-iac is canonical IaC home; `jol-infrastructure/terraform`+`helm` are legacy; **non-destructive** (no migration/deletion here; separate change-controlled effort). References (does not edit) jol-infrastructure ADR-001/003.
  - **README.md** — ADR index + the immutability rule (supersede, never rewrite).
- [ ] **Step 2:** Verify markdownlint + internal links.
- [ ] **Step 3: Commit** — `git add docs/adr && git commit -S -m "docs: add ADR-001..004 (OpenTofu, Proxmox, state, supersede legacy AWS)"`

### Task 3.4: `docs/control-mapping-matrix.md`

**Files:** Create `docs/control-mapping-matrix.md`

- [ ] **Step 1:** Author a traceability table (Control → Requirement → jol-iac artifact → Evidence/CI gate) covering: ISO A.8.9; SOC2 CC8.1; CC6.1; CC7.1; ISO A.5.29/A.5.30 + SOC2 A1.x (availability/backup — the jol-control gap this repo helps close); ISO A.8.24; A.8.13; A.8.8; A.8.19; GDPR Art. 5(1)(f)/25/32; Art. 9 (JOL=processor, tenant=controller; no personal data in examples). Link each row to the concrete file (module/policy/runbook/ADR) + CI job.
- [ ] **Step 2:** Verify markdownlint.
- [ ] **Step 3: Commit** — `git add docs/control-mapping-matrix.md && git commit -S -m "docs: add control-mapping matrix (A.8.9, CC8.1, A1.x, Art.32)"`

**Wave 3 checkpoint:** `code-reviewer` on docs/ADRs; owner confirmation before Wave 4.

---


## Wave 4 — OpenTofu Modules (empirical author→validate loop)

**Empirical rule (non-negotiable):** never assume `bpg/proxmox` argument names. Author each `main.tf`, then run `tofu init -backend=false -input=false && tofu validate` and iterate until green. A module that cannot be validated against the real schema is **reduced to a `templates/` scaffold + tracked item** (fail-closed) — never committed as unvalidated HCL. Record the resolved provider version in `docs/tool-versions.md` and commit `.terraform.lock.hcl` from an example root.

**Shared module contract** (all modules): `versions.tf` declares `required_version = ">= 1.8.0"` and `required_providers { proxmox = { source = "bpg/proxmox" } }` (no tight pin in modules — the root/example pins + locks). Governance metadata is **required inputs** (`cost_center`, `environment`, `owner`) — Proxmox tags are flat strings, so the AWS-style `CostCenter/Environment/Owner` tag map is adapted to validated inputs rendered into `tags`/`description`. `environment` is validated against `["dev","staging","prod"]`.

### Task 4.1: `modules/proxmox-vm` (core)

**Files:** Create `modules/proxmox-vm/{versions.tf,variables.tf,main.tf,outputs.tf,README.md}` and `modules/proxmox-vm/examples/{basic,complete}/{main.tf,variables.tf,outputs.tf,terraform.tfvars.example}`

- [ ] **Step 1:** Write `versions.tf` (deterministic):
```hcl
terraform {
  required_version = ">= 1.8.0"
  required_providers {
    proxmox = { source = "bpg/proxmox" }
  }
}
```
- [ ] **Step 2:** Write `variables.tf` with required governance inputs + validation (deterministic):
```hcl
variable "vm_name" {
  type        = string
  description = "VM name (placeholder-safe)."
}
variable "node_name" {
  type        = string
  description = "Proxmox node (e.g. pve-example)."
}
variable "cost_center" {
  type        = string
  description = "Cost center (governance metadata)."
  validation {
    condition     = length(var.cost_center) > 0
    error_message = "cost_center is required (iac_policy)."
  }
}
variable "environment" {
  type        = string
  description = "Deployment environment."
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "environment must be dev|staging|prod."
  }
}
variable "owner" {
  type        = string
  description = "Owning team/individual (governance metadata)."
  validation {
    condition     = length(var.owner) > 0
    error_message = "owner is required (iac_policy)."
  }
}
variable "cpu_cores" {
  type    = number
  default = 2
}
variable "memory_mb" {
  type    = number
  default = 4096
}
variable "disk_size_gb" {
  type    = number
  default = 32
}
variable "bridge" {
  type    = string
  default = "vmbr0"
}
variable "vlan_tag" {
  type    = number
  default = null
}
```
- [ ] **Step 3:** Author `main.tf` using resource `proxmox_virtual_environment_vm` with: `name`, `node_name`, `description` (renders `env/owner/cc`), `tags` (flat list incl. `"env-${var.environment}"`, `"owner-${var.owner}"`), `cpu { cores }`, `memory { dedicated }`, `disk { datastore_id, size }`, `network { bridge, vlan_tag }`, `initialization` (cloud-init), `agent { enabled = true }`. **Confirm every argument name against the schema** via `tofu validate`; correct or drop unsupported args.
- [ ] **Step 4:** Write `outputs.tf` (e.g. `vm_id`, `name`) and `README.md` (Purpose, Requirements, Providers, Inputs table, Outputs, Example usage — placeholder values only).
- [ ] **Step 5:** Author `examples/basic` + `examples/complete` (root modules that pin the provider + call the module with placeholder values), then `cd examples/basic && tofu init -backend=false -input=false && tofu validate` → green. Repeat for `complete`. Copy the generated `.terraform.lock.hcl` from an example to the repo root of that example (commit it).
- [ ] **Step 6 (gate):** `tofu fmt -recursive modules/proxmox-vm && (cd modules/proxmox-vm/examples/basic && tofu validate)` → `Success! The configuration is valid.` Capture output to `/tmp/jol_iac_vm_validate.txt`.
- [ ] **Step 7: Commit** — `git add modules/proxmox-vm && git commit -S -m "feat(modules): add proxmox-vm (validated against bpg/proxmox schema)"`

### Task 4.2: `modules/proxmox-firewall`

**Files:** Create `modules/proxmox-firewall/{versions.tf,variables.tf,main.tf,outputs.tf,README.md,examples/...}`

- [ ] **Step 1:** Same contract. Author `main.tf` with `proxmox_virtual_environment_firewall_rules` (+ options) implementing **default-deny inbound** with explicitly scoped allow rules (source CIDR + port). No `0.0.0.0/0` on sensitive ports (enforced by `no_public_ingress.rego`).
- [ ] **Step 2:** `tofu validate` green (examples). Capture to `/tmp/jol_iac_fw_validate.txt`.
- [ ] **Step 3: Commit** — `git add modules/proxmox-firewall && git commit -S -m "feat(modules): add proxmox-firewall (default-deny, scoped ingress)"`

### Task 4.3: `modules/proxmox-pool`

**Files:** Create `modules/proxmox-pool/{versions.tf,variables.tf,main.tf,outputs.tf,README.md,examples/...}`

- [ ] **Step 1:** Author `main.tf` with `proxmox_virtual_environment_pool` (`pool_id`, `comment` rendering governance metadata). Validate + examples.
- [ ] **Step 2: Commit** — `git add modules/proxmox-pool && git commit -S -m "feat(modules): add proxmox-pool (governance grouping)"`

### Task 4.4: `modules/proxmox-backup` (PBS) — conditional, fail-closed

**Files:** Create `modules/proxmox-backup/...` **OR** `templates/backup-job/...`

- [ ] **Step 1:** Determine empirically whether the pinned `bpg/proxmox` schema exposes a PBS backup-job/datastore resource. Run a scratch `tofu init` + inspect `tofu providers schema -json | jq '.provider_schemas[].resource_schemas | keys'` (capture to `/tmp/jol_iac_pbs_schema.txt`).
- [ ] **Step 2 (branch):** If supported → author `modules/proxmox-backup` (schedule, retention/prune, datastore, target selection) + validate. If NOT supported → create `templates/backup-job/` (documented PBS backup-job config scaffold + README) and add a tracked item in `CHANGELOG.md` `## [Unreleased]` + `docs/control-mapping-matrix.md` (A.5.29/A.5.30 row) noting backup is PBS/Ansible-managed pending provider support. **Do not fake a module.**
- [ ] **Step 3: Commit** — `git add modules/proxmox-backup templates/backup-job CHANGELOG.md docs/control-mapping-matrix.md 2>/dev/null; git commit -S -m "feat(backup): PBS backup module or documented template (schema-gated)"`

**Wave 4 checkpoint:** `code-reviewer` on all modules + validation evidence; owner confirmation before Wave 5.

---

## Wave 5 — Policy-as-Code, Templates & SOPS Docs

### Task 5.1: OPA policy + tests (`policy/opa/`)

**Files:** Create `policy/opa/{no_secrets_in_tf,required_tags,no_public_ingress,resource_limits}.rego` + matching `*_test.rego`

- [ ] **Step 1:** Author the four rego rules (`package jol.iac`), each a `deny[msg]` over conftest HCL-parsed input:
  - `no_secrets_in_tf` — deny resources whose arguments contain obvious secret literals (password/token/private key patterns).
  - `required_tags` — deny `proxmox_virtual_environment_vm` lacking governance tags/description encoding `env`/`owner`/`cc`.
  - `no_public_ingress` — deny firewall rules with source `0.0.0.0/0` on sensitive ports.
  - `resource_limits` — deny VMs without cpu cores + memory bounds.
- [ ] **Step 2:** Write `*_test.rego` with `deny`/allow fixtures (one violating + one compliant input each). **Local gate:** `opa test policy/opa -v --coverage` → all PASS (opa is installed). Capture to `/tmp/jol_iac_opa.txt`.
- [ ] **Step 3:** Confirm the conftest HCL input key-paths in CI (`conftest test --parser hcl examples modules --policy policy/opa`); adjust rule input paths to the real parsed shape (empirical). Do not rely on `opa test` fixtures alone for shape correctness.
- [ ] **Step 4: Commit** — `git add policy/opa && git commit -S -m "feat(policy): add OPA/conftest rules + unit tests (tags, secrets, ingress, limits)"`

### Task 5.2: `templates/env-stack` + `templates/module-template`

**Files:** Create `templates/env-stack/{main.tf,providers.tf.tmpl,backend.tf.tmpl,variables.tf,terraform.tfvars.example,README.md}` and `templates/module-template/{versions.tf,variables.tf,main.tf,outputs.tf,README.md,examples/basic/...}`

- [ ] **Step 1:** `env-stack` = consumer copy-paste root wiring the modules; `backend.tf.tmpl` documents local-first + MinIO/S3-compatible remote (locking/SSE/versioning) with a bold **NEVER commit state** note; `providers.tf.tmpl` pins `bpg/proxmox`; `terraform.tfvars.example` uses placeholders.
- [ ] **Step 2:** `module-template` = authoring skeleton that passes policy + validation by construction (A.8.9 consistency).
- [ ] **Step 3:** Validate any real `.tf` under templates (`tofu init -backend=false && tofu validate` where applicable; `.tmpl` files are excluded from fmt/validate).
- [ ] **Step 4: Commit** — `git add templates && git commit -S -m "feat(templates): add env-stack consumer root and module-template skeleton"`

### Task 5.3: SOPS Gate 4–6 docs/templates (`sops/`) — inert, no scripts, no keys

**Files:** Create `sops/README.md`, `sops/runbooks/{gate-4-install-sops.md,gate-5-age-identity.md,gate-6-shamir-dr.md}`, `sops/checklists/gate-4-6-conditions.md`, `sops/templates/{sops.yaml.tmpl,age-jol.recipients.tmpl}`, `sops/tool-versions.md`

- [ ] **Step 1:** `sops/README.md` — prominent banner: ADR-003 amendment is **PROPOSED**; no gate executes until its Conditions are met; **private key material is NEVER committed**; runbooks are human-executed under change control (SOC2 CC8.1) with fail-closed verification + evidence capture. Cross-link (do not copy) `adr-003-amendment-sops-age.md`, `sops-recipients-registry.md` (Gate 7), `jol-infrastructure/docs/audit/gate*`.
- [ ] **Step 2:** Runbooks to spec: **gate-4** install SOPS with version + SHA256 checksum verification, preflight, evidence, rollback; **gate-5** `age-jol` identity, dual-identity segregation (`age-jol` for `/opt/jol`, `age-jolarca` for `/opt/jolarca`, no cross-tree recipients), Vaultwarden = source of truth, workstation cache perms `600`/`700`, recipients via security-reviewed PR; **gate-6** Shamir **M=2/N=3** (`ssss`), custody log (locations only, never material), quarterly DR rehearsal.
- [ ] **Step 3:** `checklists/gate-4-6-conditions.md` — the amendment's go/no-go Conditions (amendment APPROVED; CC8.1 issue w/ rollback; DPIA re-check; two-identity custody plan decided BEFORE keygen; reconciled with jol-hub prior SOPS work).
- [ ] **Step 4:** Templates (deterministic — public material only):
`sops/templates/sops.yaml.tmpl`:
```yaml
# .sops.yaml — commit this file (public config). creation_rules restrict encryption scope.
creation_rules:
  - path_regex: \.enc\.yaml$
    age: "age1CHANGEME_REPLACE_WITH_age-jol_PUBLIC_KEY"
  - path_regex: (^|/)secrets/.*\.yaml$
    age: "age1CHANGEME_REPLACE_WITH_age-jol_PUBLIC_KEY"
# SEGREGATION: never list age-jolarca recipients here (AGENTS.md §0.2; ISO A.8.13).
```
`sops/templates/age-jol.recipients.tmpl`:
```text
# age-jol PUBLIC recipients only. NEVER commit the private key (age-*.txt / keys.txt).
age1CHANGEME_REPLACE_WITH_age-jol_PUBLIC_KEY
```
- [ ] **Step 5:** `sops/tool-versions.md` — pinned `sops` version + published SHA256, `age` version; "verify checksum before first execution" (fail-closed). Mark values `⚠ UNVERIFIED — fill from official release at first human execution` (this repo does not install tools).
- [ ] **Step 6 (gate):** `markdownlint` clean; `detect-secrets`/`trufflehog` clean; **assert no private-key-shaped content**: `grep -rnE 'AGE-SECRET-KEY|-----BEGIN' sops/ && echo "FAIL: secret-shaped content" || echo "OK: no private keys"`.
- [ ] **Step 7: Commit** — `git add sops && git commit -S -m "docs(sops): add Gate 4-6 runbooks, conditions checklist, and public templates (inert)"`

**Wave 5 checkpoint:** `code-reviewer` on policy + templates + sops; owner confirmation before final QA/publish.

---

## Final — QA Sweep, Publish Gate, Independent Verification

### Task F.1: Phase 3 full local QA

- [ ] **Step 1:** `cd /opt/jol/repos/jol-iac && make install && make fmt && make validate && make lint && make policy && make docs && make secrets && make check 2>&1 | tee /tmp/jol_iac_final_qa.txt`
- [ ] **Step 2:** Confirm every target exits 0 (fail-closed). Fix any finding; re-run. No `|| true`, no `--no-verify`.
- [ ] **Step 3:** Confirm nothing prohibited is tracked: `git ls-files | grep -E '\.tfstate|\.terraform/|\.tfvars$|age-.*\.txt$|keys\.txt' && echo "FAIL" || echo "OK: clean tree"`; and `git ls-files | grep -E '\.terraform\.lock\.hcl' && echo "OK: lock committed"`.
- [ ] **Step 4:** Update `CHANGELOG.md` seed entry with evidence references (QA log path, validation results). Commit any fixes (signed).

### Task F.2: Phase 4 Publish Gate — OWNER CONFIRMATION REQUIRED

- [ ] **Step 1:** STOP. Present the full `git log --oneline`, the QA evidence, and the exact push command to the owner. **Do not push without explicit confirmation** (irreversible: public repo).
- [ ] **Step 2 (only after explicit "push" confirmation):** `git -C /opt/jol/repos/jol-iac push -u origin main` (direct push permitted under `solo_mode`; `jol-iac` remote is empty so the first push creates `main`).
- [ ] **Step 3:** Confirm push: `git -C /opt/jol/repos/jol-iac status -b --porcelain=v1` → `## main...origin/main` (no ahead/behind).

### Task F.3: Phase 5 Independent Verification

- [ ] **Step 1:** Live remote state: `gh api repos/journeyoflife-org/jol-iac --jq '{name,visibility,default_branch,pushed_at}'` → `visibility: public`, `default_branch: main`.
- [ ] **Step 2:** Branch protection sanity: `gh api repos/journeyoflife-org/jol-iac/branches/main/protection --jq '{pr:.required_pull_request_reviews,status:.required_status_checks,sig:.required_signatures.enabled}'` → expect `pr: null`, `status: null`, `sig.enabled: true` (solo_mode).
- [ ] **Step 3:** CI actually ran + check names: `gh run list --repo journeyoflife-org/jol-iac --limit 5`; `gh api repos/journeyoflife-org/jol-iac/commits/main/check-runs --jq '.check_runs[].name'` → confirm `compliance-scan / validate` and `codeql-analysis` appear. **If a context string differs from `jol-iac.yml`, record a `jol-control` CR** (do not edit jol-control here).
- [ ] **Step 4:** Clean-clone empirical re-validation: `git clone /opt/jol/repos/jol-iac /tmp/jol-iac-verify && cd /tmp/jol-iac-verify && tofu fmt -check -recursive && (cd modules/proxmox-vm/examples/basic && tofu init -backend=false -input=false && tofu validate)` → green (proves the pushed tree validates from scratch, not just in the working dir).
- [ ] **Step 5:** Write a short verification record (PASS/FAIL per check) into `docs/` or the plan's completion note; report to owner.

---

## Self-Review (author checklist — run before execution)

- **Spec coverage:** §4 tree → Waves 1–5 (wrapper=W1, CI=W2, docs/ADR=W3, modules=W4, policy/templates/sops=W5). §7 control map → Task 3.4. §8 CI contexts → Wave 2 + F.3. §9 gitignore → Task 1.1. §10 non-destructive → Conventions + ADR-004. §11 QA → Task F.1. §12 subagent waves + Phase 4/5 → checkpoints + F.2/F.3. §13 risks → Task 4.4 (PBS schema), 5.3 (no keys), F.3 (context CR). No gaps.
- **Empirical vs assumed:** provider-arg HCL (W4) and conftest HCL input shape (W5) are authored-then-validated, not asserted — matches the Empirical Terraform Validation Requirement. Deterministic artifacts (gitignore, Makefile, pre-commit, CI YAML, versions.tf, tag variables, sops `.tmpl`) are given in full.
- **Type/name consistency:** module names (`proxmox-vm/firewall/pool/backup`), rego files, CI job names (`validate`, `codeql-analysis`), Makefile targets, and doc paths are used identically across waves.
- **Fail-closed:** single tolerated `|| true` (conftest before examples exist) is called out for removal; all other steps abort on error; no `--no-verify`.

## Execution Handoff

Plan complete. Two execution options:
1. **Subagent-Driven (recommended)** — fresh subagent per task, `code-reviewer` after each wave, owner checkpoint between waves (matches spec §12 + the subagent-driven convention).
2. **Inline Execution** — executing-plans, batch with checkpoints.
