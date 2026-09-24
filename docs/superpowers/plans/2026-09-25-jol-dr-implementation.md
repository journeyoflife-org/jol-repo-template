# JOL Disaster Recovery System of Record — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the `jol-dr` repository as the authoritative disaster recovery and business continuity system of record for the Journey of Life platform, closing the availability blind spot in `jol-control`'s audit checklists (ISO A.5.29/A.5.30/A.8.13 and SOC 2 A1.x).

**Architecture:** Private governance repo with ISO 22301-style separation: `policy/` for commitments, `registers/` for machine-readable truth (RTO/RPO, service criticality, backup schedule), `plans/` for strategy, `runbooks/` organised by scenario (total infrastructure loss, host failure, DB corruption, tenant restore, ransomware, DNS), `rehearsals/` for honest evidence (status: NOT PERFORMED), `findings/` for the 8 verified defects, `scripts/` for fail-closed harness marked UNVERIFIED. 79 files in jol-dr + ~15 files across jol-deploy and jol-control, executed in four waves.

**Tech Stack:** Markdown (policies, runbooks), YAML (registers), Bash (fail-closed scripts), GitHub Actions (CI), pre-commit (markdownlint 0.41.0, detect-secrets, shellcheck)

**Design spec:** `docs/superpowers/specs/2026-09-25-jol-dr-design.md`

---

## File Structure

**jol-dr (79 files):**

Root (12): README.md, SECURITY.md, CONTRIBUTING.md, CHANGELOG.md, CODE_OF_CONDUCT.md, LICENSE, Makefile, .editorconfig, .gitignore (harden), .markdownlint.yaml, .pre-commit-config.yaml, .secrets.baseline (generate)

.github (9): CODEOWNERS, PULL_REQUEST_TEMPLATE.md, dependabot.yml, ISSUE_TEMPLATE/dr-gap-or-finding.yml, ISSUE_TEMPLATE/runbook-change-request.yml, workflows/docs-ci.yml, workflows/compliance-check.yml, workflows/dr-verification.yml, workflows/backup-freshness.yml

docs (7): index.md, scope-statement.md, control-mapping-matrix.md, document-control.md, review-schedule.md, roles-and-responsibilities.md, evidence-integrity-standard.md

policy (4): disaster-recovery-policy.md, backup-policy.md, business-continuity-policy.md, backup-encryption-key-custody.md

registers (6): rto-rpo.yml, service-inventory.yml, backup-schedule.yml, recovery-order.yml, data-classification-dr.yml, external-dependencies.yml

plans (4): disaster-recovery-plan.md, business-continuity-plan.md, crisis-communication-plan.md, recovery-prioritisation.md

runbooks (9): README.md, RB-DR-01..08

rehearsals (9): README.md, schedule.yml, pass-criteria.yml, STATUS.md, templates/rehearsal-record.md, templates/tabletop-exercise.md, templates/restore-verification-record.md, records/README.md, records/REHEARSAL-2026-01.md

findings (9): README.md, FINDING-DR-001..008

scripts (7): README.md, lib/common.sh, dr-preflight.sh, verify-backups.sh, restore-test.sh, dr-drill.sh, emit-evidence.sh

templates (3): runbook-template.md, dr-plan-template.md, control-mapping-template.md

**jol-deploy (~8 files):** delete migrated files, replace with pointers

**jol-control (~7 files):** update repos/jol-dr.yml, add DR gates to compliance-gates.yml, add A.5.29/A.5.30/A.8.13 to iso27001-checklist.yml, add A1.1-A1.5 to soc2-checklist.yml, fix CC6.3-02 false evidence

---

## Wave 1: Governance Spine + Registers + Policy + Plans + Mapping + Findings (~35 files)

### Task 1: Root Governance Files

**Files:**
- Create: `/opt/jol/repos/jol-repo-template/.stage-dr/README.md`
- Create: `/opt/jol/repos/jol-repo-template/.stage-dr/SECURITY.md`
- Create: `/opt/jol/repos/jol-repo-template/.stage-dr/CONTRIBUTING.md`
- Create: `/opt/jol/repos/jol-repo-template/.stage-dr/CHANGELOG.md`
- Create: `/opt/jol/repos/jol-repo-template/.stage-dr/CODE_OF_CONDUCT.md`
- Create: `/opt/jol/repos/jol-repo-template/.stage-dr/LICENSE`
- Create: `/opt/jol/repos/jol-repo-template/.stage-dr/Makefile`
- Create: `/opt/jol/repos/jol-repo-template/.stage-dr/.editorconfig`
- Modify: `/opt/jol/repos/jol-repo-template/.stage-dr/.gitignore` (harden with backup-artifact patterns)
- Create: `/opt/jol/repos/jol-repo-template/.stage-dr/.markdownlint.yaml`
- Create: `/opt/jol/repos/jol-repo-template/.stage-dr/.pre-commit-config.yaml`

- [ ] **Step 1: Create staging directory**

```bash
mkdir -p /opt/jol/repos/jol-repo-template/.stage-dr
```

- [ ] **Step 2: Write README.md**

```markdown
# JOL Disaster Recovery System of Record

**Status:** PRIVATE · tier: governance
**Compliance:** SOC 2 Type II / GDPR / ISO 27001:2022

This is the authoritative disaster recovery (DR) and business continuity (BCP) system of record for the Journey of Life platform. It closes the availability blind spot in `jol-control`'s audit checklists (ISO A.5.29/A.5.30/A.8.13 and SOC 2 A1.x).

## Rehearsal Status

| Rehearsal | Status | Last Performed | Next Scheduled | Evidence |
|-----------|--------|----------------|----------------|----------|
| Weekly restore test | NOT PERFORMED | — | TBD | — |
| Monthly DR drill | NOT PERFORMED | — | TBD | — |
| Annual full BCP exercise | NOT PERFORMED | — | TBD | — |

**Note:** No DR rehearsal has been performed as of 2026-09-25. The first scheduled rehearsal is REHEARSAL-2026-01 (status: SCHEDULED).

## Repository Structure

- `policy/` — Tier-1 commitments (DR-POL-001..004)
- `registers/` — machine-readable single source of truth (YAML)
- `plans/` — Tier-2 strategy
- `runbooks/` — Tier-3, scenario-led (operator navigates by event)
- `rehearsals/` — honest evidence (status: NOT PERFORMED)
- `findings/` — 8 verified defects, tracked to closure
- `scripts/` — fail-closed harness (UNVERIFIED)
- `docs/` — governance spine

## Related Repositories

- `jol-control` — governance control plane (Terraform)
- `jol-policies` — ISMS policy suite
- `jol-deploy` — deployment automation (DR content migrated here)
- `jol-infrastructure` — Proxmox infrastructure
- `jol-secrets` — SOPS/age secret management

## Reporting a Security Vulnerability

**Do not open a public issue for security vulnerabilities.** See [`SECURITY.md`](SECURITY.md) for the private disclosure channel.
```

- [ ] **Step 3: Write SECURITY.md**

Copy from `jol-policies/SECURITY.md`, adjust repo name.

- [ ] **Step 4: Write CONTRIBUTING.md**

Copy from `jol-policies/CONTRIBUTING.md`, adjust repo name. Add section on evidence-integrity rules.

- [ ] **Step 5: Write CHANGELOG.md**

```markdown
# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added
- Initial DR system of record (governance spine, registers, policy, plans, findings)
```

- [ ] **Step 6: Write CODE_OF_CONDUCT.md**

Copy from `jol-policies/CODE_OF_CONDUCT.md`.

- [ ] **Step 7: Write LICENSE**

```markdown
Copyright (c) 2026 Journey of Life

This repository is PRIVATE and contains operational security detail.
Unauthorized access, distribution, or use is prohibited.

For licensing inquiries, contact: journey4oflife@gmail.com
```

- [ ] **Step 8: Write Makefile**

Copy from `jol-policies/Makefile`, add `dr-verify` target:

```makefile
dr-verify: ## Run DR harness verification (shellcheck + fail-closed contract test)
    shellcheck scripts/*.sh scripts/lib/*.sh
    bash -n scripts/*.sh scripts/lib/*.sh
    @echo "Checking for silent failures (|| true)..."
    @grep -n '|| true' scripts/*.sh scripts/lib/*.sh && { echo "FAIL: found || true"; exit 1; } || echo "OK: no silent failures"
```

- [ ] **Step 9: Write .editorconfig**

Copy from `jol-policies/.editorconfig`.

- [ ] **Step 10: Harden .gitignore**

Append to existing `.gitignore`:

```gitignore
# Backup artifacts — never commit
*.bundle
*.sql.gz
*.tar.gz
*.zst
*.enc
recovery-artifacts/
restore-test-scratch/
```

- [ ] **Step 11: Write .markdownlint.yaml**

Copy from `jol-policies/.markdownlint.yaml` (MD060 disabled, MD040 enforced).

- [ ] **Step 12: Write .pre-commit-config.yaml**

Copy from `jol-policies/.pre-commit-config.yaml` (markdownlint 0.41.0 pinned, detect-secrets, forbid-tabs).

- [ ] **Step 13: Verify files created**

```bash
ls -la /opt/jol/repos/jol-repo-template/.stage-dr/
```

Expected: 12 files listed.

---

### Task 2: .github/ Files

**Files:**
- Create: `/opt/jol/repos/jol-repo-template/.stage-dr/.github/CODEOWNERS`
- Create: `/opt/jol/repos/jol-repo-template/.stage-dr/.github/PULL_REQUEST_TEMPLATE.md`
- Create: `/opt/jol/repos/jol-repo-template/.stage-dr/.github/dependabot.yml`
- Create: `/opt/jol/repos/jol-repo-template/.stage-dr/.github/ISSUE_TEMPLATE/dr-gap-or-finding.yml`
- Create: `/opt/jol/repos/jol-repo-template/.stage-dr/.github/ISSUE_TEMPLATE/runbook-change-request.yml`
- Create: `/opt/jol/repos/jol-repo-template/.stage-dr/.github/workflows/docs-ci.yml`
- Create: `/opt/jol/repos/jol-repo-template/.stage-dr/.github/workflows/compliance-check.yml`

- [ ] **Step 1: Create .github/ directory structure**

```bash
mkdir -p /opt/jol/repos/jol-repo-template/.stage-dr/.github/{ISSUE_TEMPLATE,workflows}
```

- [ ] **Step 2: Write CODEOWNERS**

```text
# CODEOWNERS — JOL DR (disaster recovery system of record)
#
# Each line is a file pattern followed by one or more owners; the last match wins.
# NOTE: only teams verified to exist in `journeyoflife-org` are referenced here.
# (The org's teams are: backend, data, devops, frontend, security.)

# Default ownership — the security team governs the DR system of record.
*                                   @journeyoflife-org/security

# Policy documents and the authoritative mappings require security review.
policy/                             @journeyoflife-org/security
registers/                          @journeyoflife-org/security
docs/control-mapping-matrix.md      @journeyoflife-org/security
docs/document-control.md            @journeyoflife-org/security
docs/scope-statement.md             @journeyoflife-org/security

# Security- and governance-sensitive paths.
SECURITY.md                         @journeyoflife-org/security
CODE_OF_CONDUCT.md                  @journeyoflife-org/security
.github/CODEOWNERS                  @journeyoflife-org/security
.github/workflows/                  @journeyoflife-org/security

# CI / dependency automation.
.github/dependabot.yml              @journeyoflife-org/security
```

- [ ] **Step 3: Write PULL_REQUEST_TEMPLATE.md**

```markdown
## Description

<!-- Describe the changes in this PR -->

## Checklist

- [ ] Does this PR change a recovery step? If yes, have you updated the corresponding runbook?
- [ ] Does this PR change RTO/RPO targets? If yes, have you updated `registers/rto-rpo.yml`?
- [ ] Does this PR add a new policy or runbook? If yes, does it have a Document Control block?
- [ ] Does this PR change a control mapping? If yes, have you updated `docs/control-mapping-matrix.md`?
- [ ] Have you run `make lint` and `make check` locally?
- [ ] Have you verified that no script uses `|| true` (silent failure)?

## Related Issues

<!-- Link any related issues -->
```

- [ ] **Step 4: Write dependabot.yml**

Copy from `jol-policies/.github/dependabot.yml`.

- [ ] **Step 5: Write ISSUE_TEMPLATE/dr-gap-or-finding.yml**

```yaml
name: DR Gap or Finding
description: Report a gap in DR coverage or a finding from verification
title: "[FINDING]: "
labels: ["finding", "triage"]
body:
  - type: markdown
    attributes:
      value: |
        Use this template to report a gap in DR coverage or a finding from verification.
  - type: input
    id: finding-id
    attributes:
      label: Finding ID
      description: Will be assigned as FINDING-DR-NNN
    validations:
      required: false
  - type: dropdown
    id: severity
    attributes:
      label: Severity
      options:
        - Critical
        - High
        - Medium
        - Low
    validations:
      required: true
  - type: textarea
    id: description
    attributes:
      label: Description
      description: Describe the gap or finding
    validations:
      required: true
  - type: textarea
    id: evidence
    attributes:
      label: Evidence
      description: Provide evidence (logs, screenshots, etc.)
    validations:
      required: false
  - type: textarea
    id: closure-criteria
    attributes:
      label: Closure Criteria
      description: What must be true for this finding to be closed?
    validations:
      required: true
```

- [ ] **Step 6: Write ISSUE_TEMPLATE/runbook-change-request.yml**

Similar to `jol-policies/.github/ISSUE_TEMPLATE/policy-change-request.yml`, adjusted for runbooks.

- [ ] **Step 7: Write workflows/docs-ci.yml**

Copy from `jol-policies/.github/workflows/docs-ci.yml`.

- [ ] **Step 8: Write workflows/compliance-check.yml**

Adapt from `jol-policies/.github/workflows/compliance-check.yml`:
- Check every policy has Document Control block
- Check every control in matrix maps to at least one document
- Check no document claims "effective" without evidence artefact
- Check no fabricated evidence statuses

---

### Task 3: docs/ Governance Spine

**Files:**
- Create: `/opt/jol/repos/jol-repo-template/.stage-dr/docs/index.md`
- Create: `/opt/jol/repos/jol-repo-template/.stage-dr/docs/scope-statement.md`
- Create: `/opt/jol/repos/jol-repo-template/.stage-dr/docs/control-mapping-matrix.md`
- Create: `/opt/jol/repos/jol-repo-template/.stage-dr/docs/document-control.md`
- Create: `/opt/jol/repos/jol-repo-template/.stage-dr/docs/review-schedule.md`
- Create: `/opt/jol/repos/jol-repo-template/.stage-dr/docs/roles-and-responsibilities.md`
- Create: `/opt/jol/repos/jol-repo-template/.stage-dr/docs/evidence-integrity-standard.md`

- [ ] **Step 1: Create docs/ directory**

```bash
mkdir -p /opt/jol/repos/jol-repo-template/.stage-dr/docs
```

- [ ] **Step 2: Write docs/index.md**

Reading order and incident-time shortcut table.

- [ ] **Step 3: Write docs/scope-statement.md**

DR/BCM scope, boundaries, explicit exclusions.

- [ ] **Step 4: Write docs/control-mapping-matrix.md**

Full control mapping per Section 2.1 of design spec.

- [ ] **Step 5: Write docs/document-control.md**

Master register of all Doc IDs.

- [ ] **Step 6: Write docs/review-schedule.md**

Review cadence (annual for policies, quarterly for registers).

- [ ] **Step 7: Write docs/roles-and-responsibilities.md**

SOLO-OPERATOR model — no fictional "DR team".

- [ ] **Step 8: Write docs/evidence-integrity-standard.md**

Full evidence-integrity standard per Section 2.2 of design spec.

---

### Task 4: policy/ Documents

**Files:**
- Create: `/opt/jol/repos/jol-repo-template/.stage-dr/policy/disaster-recovery-policy.md`
- Create: `/opt/jol/repos/jol-repo-template/.stage-dr/policy/backup-policy.md`
- Create: `/opt/jol/repos/jol-repo-template/.stage-dr/policy/business-continuity-policy.md`
- Create: `/opt/jol/repos/jol-repo-template/.stage-dr/policy/backup-encryption-key-custody.md`

- [ ] **Step 1: Create policy/ directory**

```bash
mkdir -p /opt/jol/repos/jol-repo-template/.stage-dr/policy
```

- [ ] **Step 2: Write disaster-recovery-policy.md (DR-POL-001)**

Document Control block + policy content. Maps to ISO A.5.29, A.5.30, A.8.13, A.8.14; SOC2 A1.1-A1.5; GDPR Art. 5(1)(f), Art. 32.

- [ ] **Step 3: Write backup-policy.md (DR-POL-002)**

Migrate from `jol-deploy/backup/backup-policy.md`, add Document Control block, add DB PITR overlay (RPO ≤1h). Supersedes jol-deploy copy.

- [ ] **Step 4: Write business-continuity-policy.md (DR-POL-003)**

Document Control block + BCP policy content.

- [ ] **Step 5: Write backup-encryption-key-custody.md (DR-POL-004)**

SOPS/age + Shamir M=2/N=3 aligned with `jol-secrets`.

---

### Task 5: registers/ (YAML)

**Files:**
- Create: `/opt/jol/repos/jol-repo-template/.stage-dr/registers/rto-rpo.yml`
- Create: `/opt/jol/repos/jol-repo-template/.stage-dr/registers/service-inventory.yml`
- Create: `/opt/jol/repos/jol-repo-template/.stage-dr/registers/backup-schedule.yml`
- Create: `/opt/jol/repos/jol-repo-template/.stage-dr/registers/recovery-order.yml`
- Create: `/opt/jol/repos/jol-repo-template/.stage-dr/registers/data-classification-dr.yml`
- Create: `/opt/jol/repos/jol-repo-template/.stage-dr/registers/external-dependencies.yml`

- [ ] **Step 1: Create registers/ directory**

```bash
mkdir -p /opt/jol/repos/jol-repo-template/.stage-dr/registers
```

- [ ] **Step 2: Write registers/rto-rpo.yml**

Migrate from `jol-deploy/backup/rto-rpo.md`, convert to YAML, add DB PITR overlay (RPO ≤1h).

- [ ] **Step 3: Write registers/service-inventory.yml**

Criticality tiers including Article 9 / donation data.

- [ ] **Step 4: Write registers/backup-schedule.yml**

What/when/retention/target/encryption/verified-by.

- [ ] **Step 5: Write registers/recovery-order.yml**

Explicit dependency-ordered restore sequence.

- [ ] **Step 6: Write registers/data-classification-dr.yml**

Handling of personal data inside backups.

- [ ] **Step 7: Write registers/external-dependencies.yml**

GitHub, DNS registrar, PBS, Let's Encrypt.

---

### Task 6: plans/

**Files:**
- Create: `/opt/jol/repos/jol-repo-template/.stage-dr/plans/disaster-recovery-plan.md`
- Create: `/opt/jol/repos/jol-repo-template/.stage-dr/plans/business-continuity-plan.md`
- Create: `/opt/jol/repos/jol-repo-template/.stage-dr/plans/crisis-communication-plan.md`
- Create: `/opt/jol/repos/jol-repo-template/.stage-dr/plans/recovery-prioritisation.md`

- [ ] **Step 1: Create plans/ directory**

```bash
mkdir -p /opt/jol/repos/jol-repo-template/.stage-dr/plans
```

- [ ] **Step 2: Write disaster-recovery-plan.md**

Migrate from `jol-deploy/docs/disaster-recovery/dr-plan.md`, expand, fix 4h-RTO-vs-6h-phase overrun, remove "DR team" reference.

- [ ] **Step 3: Write business-continuity-plan.md**

BCP strategy document.

- [ ] **Step 4: Write crisis-communication-plan.md**

Communication strategy during DR events.

- [ ] **Step 5: Write recovery-prioritisation.md**

Tiering rationale, links to registers.

---

### Task 7: findings/

**Files:**
- Create: `/opt/jol/repos/jol-repo-template/.stage-dr/findings/README.md`
- Create: `/opt/jol/repos/jol-repo-template/.stage-dr/findings/FINDING-DR-001.md` through `FINDING-DR-008.md`

- [ ] **Step 1: Create findings/ directory**

```bash
mkdir -p /opt/jol/repos/jol-repo-template/.stage-dr/findings
```

- [ ] **Step 2: Write findings/README.md**

Severity + closure rules.

- [ ] **Step 3: Write FINDING-DR-001.md**

DR drill script non-functional stub.

- [ ] **Step 4: Write FINDING-DR-002.md**

Restore test false pass.

- [ ] **Step 5: Write FINDING-DR-003.md**

No availability controls in audit checklists.

- [ ] **Step 6: Write FINDING-DR-004.md**

Backup schedule not evidenced.

- [ ] **Step 7: Write FINDING-DR-005.md**

Prod hosts backup disabled.

- [ ] **Step 8: Write FINDING-DR-006.md**

TFstate false evidence CC6.3-02.

- [ ] **Step 9: Write FINDING-DR-007.md**

DR plan RTO overrun and team reference.

- [ ] **Step 10: Write FINDING-DR-008.md**

World-readable bundle and unverifiable encrypted dir.

---

### Task 8: Wave 1 QA Gate and Commit

- [ ] **Step 1: Copy staging to jol-dr**

```bash
cp -a /opt/jol/repos/jol-repo-template/.stage-dr/. /opt/jol/repos/jol-dr/
```

- [ ] **Step 2: Generate .secrets.baseline**

```bash
cd /opt/jol/repos/jol-dr
detect-secrets scan > .secrets.baseline
```

- [ ] **Step 3: Run markdownlint**

```bash
npx markdownlint-cli@0.41.0 "**/*.md" --config .markdownlint.yaml
```

Expected: PASS (fix any errors).

- [ ] **Step 4: Run pre-commit**

```bash
pre-commit run --files $(find . -type f -not -path './.git/*' -not -path './.venv/*' -not -path './.idea/*')
```

Expected: PASS.

- [ ] **Step 5: Verify Document Control blocks**

```bash
for f in policy/*.md; do grep -q '^## Document Control' "$f" || echo "MISSING: $f"; done
```

Expected: no output (all present).

- [ ] **Step 6: Verify control-mapping coverage**

```bash
for id in DR-POL-001 DR-POL-002 DR-POL-003 DR-POL-004; do grep -q "$id" docs/control-mapping-matrix.md || echo "UNMAPPED: $id"; done
```

Expected: no output (all mapped).

- [ ] **Step 7: Commit Wave 1**

```bash
cd /opt/jol/repos/jol-dr
git add .
git commit -S -m "docs: seed DR system of record — governance spine, registers, policy, plans, findings"
```

---

## Wave 2: Runbooks + Rehearsals (~18 files)

### Task 9: runbooks/

**Files:**
- Create: `/opt/jol/repos/jol-dr/runbooks/README.md`
- Create: `/opt/jol/repos/jol-dr/runbooks/RB-DR-01-total-infrastructure-loss.md` through `RB-DR-08-age-key-loss-recovery.md`

- [ ] **Step 1: Create runbooks/ directory**

```bash
mkdir -p /opt/jol/repos/jol-dr/runbooks
```

- [ ] **Step 2: Write runbooks/README.md**

"What happened?" decision tree.

- [ ] **Step 3: Write RB-DR-01..08**

Scenario-led runbooks with Document Control blocks. Migrate content from jol-deploy where applicable.

---

### Task 10: rehearsals/

**Files:**
- Create: `/opt/jol/repos/jol-dr/rehearsals/README.md`
- Create: `/opt/jol/repos/jol-dr/rehearsals/schedule.yml`
- Create: `/opt/jol/repos/jol-dr/rehearsals/pass-criteria.yml`
- Create: `/opt/jol/repos/jol-dr/rehearsals/STATUS.md`
- Create: `/opt/jol/repos/jol-dr/rehearsals/templates/rehearsal-record.md`
- Create: `/opt/jol/repos/jol-dr/rehearsals/templates/tabletop-exercise.md`
- Create: `/opt/jol/repos/jol-dr/rehearsals/templates/restore-verification.md`
- Create: `/opt/jol/repos/jol-dr/rehearsals/records/README.md`
- Create: `/opt/jol/repos/jol-dr/rehearsals/records/REHEARSAL-2026-01.md`

- [ ] **Step 1: Create rehearsals/ directory structure**

```bash
mkdir -p /opt/jol/repos/jol-dr/rehearsals/{templates,records}
```

- [ ] **Step 2: Write rehearsals/ files**

Honest status board (NOT PERFORMED), schedule, pass criteria, templates.

---

### Task 11: templates/

**Files:**
- Create: `/opt/jol/repos/jol-dr/templates/runbook-template.md`
- Create: `/opt/jol/repos/jol-dr/templates/dr-plan-template.md`
- Create: `/opt/jol/repos/jol-dr/templates/control-mapping-template.md`

- [ ] **Step 1: Create templates/ directory**

```bash
mkdir -p /opt/jol/repos/jol-dr/templates
```

- [ ] **Step 2: Write templates**

Reusable templates for runbooks, DR plans, control mappings.

---

### Task 12: Wave 2 QA Gate and Commit

- [ ] **Step 1: Run markdownlint + pre-commit**

Same as Wave 1.

- [ ] **Step 2: Commit Wave 2**

```bash
cd /opt/jol/repos/jol-dr
git add .
git commit -S -m "docs: add scenario-led runbooks and honest rehearsal programme"
```

---

## Wave 3: Scripts Harness + CI (~10 files)

### Task 13: scripts/lib/common.sh

**Files:**
- Create: `/opt/jol/repos/jol-dr/scripts/README.md`
- Create: `/opt/jol/repos/jol-dr/scripts/lib/common.sh`

- [ ] **Step 1: Create scripts/ directory structure**

```bash
mkdir -p /opt/jol/repos/jol-dr/scripts/lib
```

- [ ] **Step 2: Write scripts/README.md**

UNVERIFIED banner semantics.

- [ ] **Step 3: Write scripts/lib/common.sh**

Fail-closed primitives: die, require_tool, require_file, require_host, assert, log, emit_evidence, no_silent_failures.

---

### Task 14: scripts/ (dr-preflight.sh, verify-backups.sh)

**Files:**
- Create: `/opt/jol/repos/jol-dr/scripts/dr-preflight.sh`
- Create: `/opt/jol/repos/jol-dr/scripts/verify-backups.sh`

- [ ] **Step 1: Write dr-preflight.sh**

Environment detection with UNVERIFIED banner.

- [ ] **Step 2: Write verify-backups.sh**

Read-only backup verification with UNVERIFIED banner.

---

### Task 15: scripts/ (restore-test.sh, dr-drill.sh, emit-evidence.sh)

**Files:**
- Create: `/opt/jol/repos/jol-dr/scripts/restore-test.sh`
- Create: `/opt/jol/repos/jol-dr/scripts/dr-drill.sh`
- Create: `/opt/jol/repos/jol-dr/scripts/emit-evidence.sh`

- [ ] **Step 1: Write restore-test.sh**

Isolated restore verification with --allow-destructive gate.

- [ ] **Step 2: Write dr-drill.sh**

Orchestrator with fail-closed contract.

- [ ] **Step 3: Write emit-evidence.sh**

Evidence formatter with GPG signing.

---

### Task 16: CI Workflows

**Files:**
- Create: `/opt/jol/repos/jol-dr/.github/workflows/dr-verification.yml`
- Create: `/opt/jol/repos/jol-dr/.github/workflows/backup-freshness.yml`

- [ ] **Step 1: Write dr-verification.yml**

shellcheck + bash -n + no_silent_failures + fail-closed contract test.

- [ ] **Step 2: Write backup-freshness.yml**

Nightly cron: runs verify-backups.sh, fails if stale.

---

### Task 17: Wave 3 QA Gate and Commit

- [ ] **Step 1: Run shellcheck + bash -n**

```bash
shellcheck scripts/*.sh scripts/lib/*.sh
bash -n scripts/*.sh scripts/lib/*.sh
```

- [ ] **Step 2: Run no_silent_failures scan**

```bash
grep -n '|| true' scripts/*.sh scripts/lib/*.sh && { echo "FAIL"; exit 1; } || echo "OK"
```

- [ ] **Step 3: Commit Wave 3**

```bash
cd /opt/jol/repos/jol-dr
git add .
git commit -S -m "feat: add fail-closed DR harness (UNVERIFIED) and verification workflows"
```

---

## Wave 4: Cross-Repo Migration + jol-control Updates + Publish (~15 files across 3 repos)

### Task 18: jol-deploy Migration

- [ ] **Step 1: Delete migrated files in jol-deploy**

```bash
cd /opt/jol/repos/jol-deploy
rm backup/backup-policy.md backup/rto-rpo.md backup/disaster-recovery-test.sh backup/restore-test.sh
rm docs/disaster-recovery/dr-plan.md docs/runbooks/backup-restore.md docs/runbooks/emergency-procedures.md
```

- [ ] **Step 2: Replace with pointers**

Write short pointer files to jol-dr.

- [ ] **Step 3: Commit jol-deploy**

```bash
git add .
git commit -S -m "refactor: migrate DR content to jol-dr (authoritative source)"
```

---

### Task 19: jol-control Updates

- [ ] **Step 1: Update repos/jol-dr.yml**

Change `visibility: public` to `visibility: private`.

- [ ] **Step 2: Add DR gates to compliance-gates.yml**

backup_freshness and rehearsal_schedule gates.

- [ ] **Step 3: Add A.5.29/A.5.30/A.8.13 to iso27001-checklist.yml**

- [ ] **Step 4: Add A1.1-A1.5 to soc2-checklist.yml**

- [ ] **Step 5: Fix CC6.3-02 false evidence**

- [ ] **Step 6: Commit jol-control**

```bash
cd /opt/jol/repos/jol-control
git add .
git commit -S -m "feat: add DR gates and availability controls to audit checklists"
```

---

### Task 20: Publish Gate and Push

- [ ] **Step 1: Present QA results**

Show file count, commit summary.

- [ ] **Step 2: Request explicit GO**

Ask user for explicit confirmation before push.

- [ ] **Step 3: Push jol-dr**

```bash
cd /opt/jol/repos/jol-dr
git push -u origin main
git tag -s v1.0.0 -m "DR system of record v1.0.0"
git push origin v1.0.0
```

- [ ] **Step 4: Independent verification**

Verify via GitHub API: visibility private, isEmpty false, file count matches.

---

**End of plan.**
