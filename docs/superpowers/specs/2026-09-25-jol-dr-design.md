# JOL Disaster Recovery System of Record — Design Spec

**Date:** 2026-09-25
**Author:** JOL Platform Architect
**Status:** Approved for implementation
**Compliance:** SOC 2 Type II / GDPR / ISO 27001:2022

---

## Executive Summary

This spec defines the `jol-dr` repository as the **authoritative disaster recovery (DR) and business continuity (BCP) system of record** for the Journey of Life platform. The repo closes the **availability blind spot** in `jol-control`'s audit checklists (ISO A.5.29/A.5.30/A.8.13 and SOC 2 A1.x were entirely absent) and consolidates DR content currently scattered across `jol-deploy` into a single, auditable source of truth.

**Key design principles:**

1. **Private, full-fidelity runbooks** — repo is private to allow operational security detail (hostnames, paths, restore commands)
2. **jol-dr authoritative** — supersedes `jol-deploy` DR content; `jol-deploy` retains pointers
3. **Fail-closed harness** — scripts abort non-zero on missing tooling, never print "PASSED" unless checks actually succeeded, marked UNVERIFIED until first supervised rehearsal
4. **Tiered RTO/RPO** — infrastructure RPO 24h (PBS snapshots), database RPO ≤1h (PostgreSQL PITR)
5. **Honest evidence** — rehearsal status is NOT PERFORMED; no fabricated evidence

**Scope:** ~79 files in `jol-dr` + ~15 files across `jol-deploy` and `jol-control`, executed in four waves.

---

## Preflight Findings

Empirical verification (not from prior reports) revealed eight defects:

1. **SOPS gates are not DR gates.** `adr-003-amendment-sops-age.md` scopes "SOPS gates 4–8 rollout" for secret encryption, not DR. No DR gate exists in any gate tracker.

2. **DR rehearsal is untracked, not "pending".** `jol-control/audit/iso27001-checklist.yml` has no A.5.29, A.5.30, A.8.13. `jol-control/audit/soc2-checklist.yml` has no Availability (A1.x) category at all, though framework is declared "SOC 2 Type II".

3. **DR test script is a non-functional stub.** `jol-deploy/backup/disaster-recovery-test.sh` has 8 steps that are bare `log` calls with no commands. It cannot fail and prints "=== DR Drill complete ===". Under SOC 2 Type II this is materially misleading evidence.

4. **Backup controls not evidenced.** `jol-deploy/backup/backup-policy.md` claims daily DB dumps and monthly DR drills. Actual `/opt/jol/backups/`: newest artifact is `jol-hub-20260911.bundle` (110 MB, mode 0644, 14 days stale). Zero drill logs or rehearsal records exist anywhere in the fleet.

5. **Production backup coverage inconsistent.** `backup_enabled: false` on `her-prod-lt01`, `mcp-prod-lt01`, `group_vars/llm.yml`. The qdrant cron is `when: backup_enabled | default(false)` and swallows failure with `|| true`.

6. **DR content already exists in jol-deploy.** `docs/disaster-recovery/dr-plan.md`, `docs/runbooks/backup-restore.md`, `emergency-procedures.md`, `backup/*`, `database/*`. Without a settled ownership boundary, two competing sources of truth.

7. **jol-dr is PUBLIC.** The recorded risk acceptance R6 explicitly expires at first content push. DR runbooks disclose recovery sequencing, backup paths, RTO windows, and real hostnames (`pve-prod-hv01`, `rag-prod-lt01`, `her-prod-lt01`, `mcp-prod-lt01`).

8. **False-evidence control in SOC 2 checklist.** CC6.3-02 asserts "Terraform state encrypted at rest (S3 SSE)" citing `backend 's3' { encrypt = true }`. No backend block exists in `main.tf`; state is local plaintext `terraform.tfstate` (221 KB).

**Professional opinion:** the gap is not "a rehearsal is pending". It is that availability is an unmanaged framework category — no A1.x criteria, no A.5.29/A.5.30/A.8.13 controls, no DR gate, a stub drill script, and backup assertions with no evidence. `jol-dr` should therefore be built as the system of record that *closes* that category and records the rehearsal as honestly **not yet performed**, never as completed.

---

## Decisions Made

| # | Decision | Outcome |
|---|----------|---------|
| 1 | Visibility | **Private**, full-fidelity (+ update `jol-control/repos/jol-dr.yml` to stop Terraform drift) |
| 2 | Ownership | **`jol-dr` authoritative**; migrate content, repoint `jol-deploy` to stubs |
| 3 | Automation | **Fail-closed harness, marked UNVERIFIED**; no pass signal unless checks ran |
| 4 | RTO/RPO | **Adopt existing + DB overlay**: infra RPO 24h, database RPO ≤1h via WAL/PITR |
| 5 | Structure | **Governance spine + scenario-led runbooks** (ISO 22301-style separation) |

---

## Design Section 1: Repository Tree

```text
jol-dr/                                    PRIVATE · tier: governance
│
├── README.md                              entry point; honest status banner up top
├── SECURITY.md                            private disclosure channel (no public issues)
├── CONTRIBUTING.md                        change process for runbooks + evidence rules
├── CHANGELOG.md                           Keep-a-Changelog; supersedes noted here
├── CODE_OF_CONDUCT.md
├── LICENSE                                internal-use (NOT CC-BY — repo is private + ops-sensitive)
├── Makefile                               install/lint/links/secrets/check + new: dr-verify
├── .editorconfig
├── .gitignore                             HARDENED with backup-artifact patterns
├── .markdownlint.yaml                     MD060 disabled, MD040 enforced
├── .pre-commit-config.yaml                markdownlint 0.41.0 pinned, detect-secrets, forbid-tabs
├── .secrets.baseline                      GENERATED by detect-secrets scan (never hand-written)
│
├── .github/
│   ├── CODEOWNERS                         @journeyoflife-org/security (verified team)
│   ├── PULL_REQUEST_TEMPLATE.md           includes "does this change a recovery step?" checklist
│   ├── dependabot.yml
│   ├── ISSUE_TEMPLATE/
│   │   ├── dr-gap-or-finding.yml          raises a FINDING-DR-nnn
│   │   └── runbook-change-request.yml
│   └── workflows/
│       ├── docs-ci.yml                    markdownlint + link check + doc-control presence
│       ├── compliance-check.yml           every doc mapped; no fabricated evidence statuses
│       ├── dr-verification.yml            shellcheck + fail-closed contract test
│       └── backup-freshness.yml           scheduled: FAILS if backups stale > threshold
│
├── docs/                                  governance spine (mirrors jol-policies)
│   ├── index.md                           reading order; incident-time shortcut table
│   ├── scope-statement.md                 DR/BCM scope, boundaries, explicit exclusions
│   ├── control-mapping-matrix.md          A.5.29/A.5.30/A.8.13 · SOC2 A1.1–A1.3 · GDPR
│   ├── document-control.md                master register (all Doc IDs)
│   ├── review-schedule.md
│   ├── roles-and-responsibilities.md      SOLO-OPERATOR model — no fictional "DR team"
│   └── evidence-integrity-standard.md     what may be claimed as evidence; bans stub-pass
│
├── policy/                                Tier-1 commitments
│   ├── disaster-recovery-policy.md        DR-POL-001
│   ├── backup-policy.md                   DR-POL-002  authoritative; SUPERSEDES jol-deploy's
│   ├── business-continuity-policy.md      DR-POL-003
│   └── backup-encryption-key-custody.md   DR-POL-004  SOPS/age + Shamir M=2/N=3 aligned
│
├── registers/                             machine-readable single source of truth (YAML)
│   ├── rto-rpo.yml                        infra RPO 24h · DB RPO ≤1h (PITR) · RTO fixed
│   ├── service-inventory.yml              criticality tiers incl. Article 9 / donation data
│   ├── backup-schedule.yml                what/when/retention/target/encryption/verified-by
│   ├── recovery-order.yml                 explicit dependency-ordered restore sequence
│   ├── data-classification-dr.yml         handling of personal data inside backups
│   └── external-dependencies.yml          GitHub, DNS registrar, PBS, Let's Encrypt
│
├── plans/                                 Tier-2 strategy
│   ├── disaster-recovery-plan.md          master; FIXES 4h-RTO vs 6h-phase overrun
│   ├── business-continuity-plan.md
│   ├── crisis-communication-plan.md
│   └── recovery-prioritisation.md
│
├── runbooks/                              Tier-3, SCENARIO-LED (operator navigates by event)
│   ├── README.md                          "what happened?" decision tree
│   ├── RB-DR-01-total-infrastructure-loss.md
│   ├── RB-DR-02-proxmox-host-failure.md
│   ├── RB-DR-03-database-corruption-pitr.md
│   ├── RB-DR-04-single-tenant-restore.md
│   ├── RB-DR-05-ransomware-recovery.md
│   ├── RB-DR-06-dns-failure-or-hijack.md
│   ├── RB-DR-07-backup-target-failure.md
│   └── RB-DR-08-age-key-loss-recovery.md
│
├── rehearsals/                            the honesty core
│   ├── README.md                          weekly restore test · monthly drill · annual full
│   ├── schedule.yml
│   ├── pass-criteria.yml                  objective, measurable — no self-declared pass
│   ├── STATUS.md                          board: DR REHEARSAL = NOT PERFORMED
│   ├── templates/rehearsal-record.md
│   ├── templates/tabletop-exercise.md
│   ├── templates/restore-verification.md
│   └── records/REHEARSAL-2026-01.md       status: SCHEDULED — never "completed"
│
├── findings/                              the 8 verified defects, tracked to closure
│   ├── README.md                          severity + closure rules
│   └── FINDING-DR-001 … 008.md            one file per finding from preflight
│
├── scripts/                               fail-closed harness
│   ├── README.md                          UNVERIFIED banner semantics
│   ├── lib/common.sh                      die/require_tool/assert — no `|| true`
│   ├── dr-preflight.sh                    aborts non-zero on missing tooling
│   ├── verify-backups.sh                  read-only integrity + staleness + retention
│   ├── restore-test.sh                    isolated scratch only; --allow-destructive gate
│   ├── dr-drill.sh                        orchestrator; emits evidence JSON
│   └── emit-evidence.sh
│
└── templates/
    ├── runbook-template.md
    ├── dr-plan-template.md
    └── control-mapping-template.md
```

**79 files.** Three deliberate design points:

- **`registers/*.yml` are the single source of truth.** `jol-control` already declares `language: YAML` for this repo. Prose documents *reference* the registers; they never restate RTO/RPO numbers.
- **`findings/` exists as a first-class directory.** The 8 preflight defects get tracked with severity and closure criteria rather than being quietly absorbed.
- **`rehearsals/STATUS.md` will say NOT PERFORMED.** I will not generate a completed rehearsal record, because none has occurred.

---

## Design Section 2: Control Mapping and Evidence-Integrity Model

### 2.1 Control-Mapping Matrix Structure

The matrix closes the **availability blind spot** in `jol-control`'s audit checklists. It must trace every DR/BCP commitment to at least one framework control, and every control must map back to at least one document.

| Doc ID | Document | ISO 27001:2022 | SOC 2 TSC | GDPR |
|--------|----------|----------------|-----------|------|
| DR-POL-001 | Disaster Recovery Policy | A.5.29, A.5.30, A.8.13, A.8.14 | A1.1, A1.2, A1.3, A1.4, A1.5 | Art. 5(1)(f), Art. 32 |
| DR-POL-002 | Backup Policy | A.5.29, A.5.30, A.8.13 | A1.2, A1.3 | Art. 5(1)(f), Art. 17, Art. 32 |
| DR-POL-003 | Business Continuity Policy | A.5.29, A.5.30, A.8.14 | A1.1, A1.4, A1.5 | Art. 5(1)(f), Art. 32 |
| DR-POL-004 | Backup Encryption & Key Custody | A.5.29, A.8.13, A.8.24 | A1.2, CC6.1 | Art. 5(1)(f), Art. 32 |
| RB-DR-01..08 | Runbooks | A.5.29, A.5.30 | A1.2, A1.3 | Art. 32 |
| REH-001..N | Rehearsal records | A.5.29, A.5.30 | A1.3 | — |
| FINDING-DR-001..008 | Findings register | A.5.29, A.5.30, A.8.13 | A1.2, A1.3 | — |

**Reverse index** (control → documents) must show that every control is covered by at least one document. Gaps are tracked in `findings/` with severity and closure date.

**Key mappings:**

- **ISO A.5.29 (Information security during disruption)**: covered by `DR-POL-001`, `DR-POL-003`, `plans/disaster-recovery-plan.md`, `plans/business-continuity-plan.md`, and every runbook.
- **ISO A.5.30 (Technology readiness for business continuity)**: covered by `DR-POL-001`, `DR-POL-002`, `registers/rto-rpo.yml`, `registers/service-inventory.yml`, `registers/backup-schedule.yml`, `scripts/`, and rehearsal records.
- **ISO A.8.13 (Information backup)**: covered by `DR-POL-002`, `DR-POL-004`, `registers/backup-schedule.yml`, `registers/backup-encryption-key-custody.md`, `scripts/verify-backups.sh`, and weekly restore verification records.
- **SOC 2 A1.1 (Capacity planning)**: covered by `DR-POL-003`, `registers/service-inventory.yml`.
- **SOC 2 A1.2 (Backup processes and recovery infrastructure)**: covered by `DR-POL-002`, `DR-POL-004`, `registers/backup-schedule.yml`, `scripts/verify-backups.sh`, `scripts/restore-test.sh`.
- **SOC 2 A1.3 (Recovery plan testing)**: covered by `rehearsals/README.md`, `rehearsals/schedule.yml`, `rehearsals/records/REHEARSAL-2026-01.md`, `scripts/dr-drill.sh`.
- **SOC 2 A1.4 (Effects of system changes during recovery)**: covered by `plans/disaster-recovery-plan.md` Phase 4 (Verification), `runbooks/RB-DR-01..08` verification steps.
- **SOC 2 A1.5 (System inventory for disaster recovery)**: covered by `registers/service-inventory.yml`, `registers/recovery-order.yml`.
- **GDPR Art. 5(1)(f) (Integrity and confidentiality)**: covered by `DR-POL-004`, `registers/data-classification-dr.yml`, `policy/backup-encryption-key-custody.md`.
- **GDPR Art. 17 (Right to erasure)**: covered by `DR-POL-002` (retention), `registers/backup-schedule.yml` (retention periods), `runbooks/RB-DR-04-single-tenant-restore.md` (tenant deletion).
- **GDPR Art. 32 (Security of processing)**: covered by `DR-POL-001..004`, `registers/data-classification-dr.yml`, `scripts/`, rehearsal records.

### 2.2 Evidence-Integrity Standard

This is the anti-false-assurance layer. The standard defines what may be claimed as evidence and what may not.

**Principles:**

1. **Evidence of existence ≠ evidence of restorability.** A backup file that exists on disk is not a verified backup. A backup is only verified if it has been restored to an isolated environment and the restoration has been validated (data integrity, application health, tenant accessibility).

2. **A control may be claimed "implemented" only if evidence of operation exists.** A policy document is not evidence of implementation. A script that prints "PASSED" without performing checks is not evidence of implementation. Evidence must be generated by a verifiable process: timestamped, signed (GPG), traceable to a specific execution.

3. **A control may be claimed "effective" only if evidence of testing/verification exists.** A control is effective only if it has been tested and the test passed. Testing must be independent of the control's operation (e.g., a restore test is independent of the backup process).

4. **Scripts that emit pass signals without performing checks are classified as false-assurance defects.** They must be deleted or replaced with fail-closed implementations. A fail-closed implementation:
   - Aborts non-zero if any required tool is missing (`require_tool proxmox-backup-client`).
   - Refuses to print "PASSED" unless every check actually succeeded.
   - Never uses `|| true` to swallow errors.
   - Only restores into an isolated scratch target by default; anything touching production requires an explicit `--allow-destructive` flag.
   - Is labelled "UNVERIFIED — never executed; requires [specific host/tooling]" until first supervised rehearsal.

5. **"NOT PERFORMED" is a valid and honest status.** Fabricating a pass is a material misrepresentation. A rehearsal record must include: date, scope, operator, pass/fail criteria, actual results, evidence artefacts (logs, screenshots, signed hashes). If the rehearsal did not occur, the record states "NOT PERFORMED" and is scheduled for a future date.

6. **Backup retention must be enforced, not just declared.** `registers/backup-schedule.yml` specifies retention periods; `scripts/verify-backups.sh` checks that retention is actually enforced (no backups older than declared retention without explicit exception).

7. **Backup encryption must be verifiable.** `DR-POL-004` declares the encryption standard (SOPS/age); `scripts/verify-backups.sh` checks that backup files are encrypted (or stored in an encrypted target); `registers/backup-encryption-key-custody.md` documents key custody (Shamir M=2/N=3 per `jol-secrets`).

8. **Findings must be tracked to closure.** `findings/FINDING-DR-001..008.md` each carry: severity (Critical/High/Medium/Low), discovery date, owner, closure criteria, closure date. A finding is closed only when the closure criteria are met and evidence of closure is retained.

**Anti-patterns banned by this standard:**

- A script that prints "PASSED" without performing the check it claims to perform.
- A control claimed "implemented" with evidence that does not exist (e.g., `soc2-checklist.yml` CC6.3-02 asserting "Terraform state encrypted at rest (S3 SSE)" when no backend block exists).
- A rehearsal record marked "completed" when the rehearsal did not occur.
- A backup claimed "verified" when only its existence was checked, not its restorability.
- A runbook that references a team or role that does not exist (e.g., "Activate DR team" in a solo-operator org).
- A plan whose own timeline exceeds its stated RTO (e.g., 4h RTO but phases run to 6h).

**Enforcement:** `docs/evidence-integrity-standard.md` is normative. `workflows/compliance-check.yml` enforces it in CI: every document must have a Document Control block; every control in the matrix must map to at least one document; no document may claim a control is "effective" without an evidence artefact; `scripts/dr-verification.yml` runs shellcheck on all scripts and checks that no script uses `|| true` or prints "PASSED" without a preceding check.

**Honesty gate:** `rehearsals/STATUS.md` is the authoritative board of rehearsal status. It will initially say:

```markdown
# Rehearsal Status

| Rehearsal | Status | Last Performed | Next Scheduled | Evidence |
|-----------|--------|----------------|----------------|----------|
| Weekly restore test | NOT PERFORMED | — | TBD | — |
| Monthly DR drill | NOT PERFORMED | — | TBD | — |
| Annual full BCP exercise | NOT PERFORMED | — | TBD | — |

**Note:** No DR rehearsal has been performed as of 2026-09-25. The first scheduled rehearsal is REHEARSAL-2026-01 (status: SCHEDULED).
```

This is the honest starting position. I will not generate a completed rehearsal record.

---

## Design Section 3: Fail-Closed Harness Contract and Testing Strategy

### 3.1 Fail-Closed Contract

Every script in `scripts/` must satisfy the **fail-closed contract**:

1. **Abort on missing prerequisites.** If a required tool, host, file, or configuration is missing, the script exits non-zero with a clear diagnostic. It never proceeds and never prints a pass signal.

2. **Never print "PASSED" unless every check actually succeeded.** A pass signal is emitted only after every verification step has run and succeeded. If any step is skipped (e.g., due to a missing tool), the script exits non-zero.

3. **No silent failures.** The script never uses `|| true` to swallow errors. Every command that can fail is checked. If a command's failure is acceptable, it is explicitly documented and the script continues with a logged warning, not a silenced error.

4. **Read-only by default.** The script never modifies production state. It only reads backups, verifies integrity, and restores into isolated scratch targets (`/tmp/jol-restore-test/`). Anything that touches production (e.g., restoring over a live database) requires an explicit `--allow-destructive` flag and a confirmation prompt.

5. **UNVERIFIED banner.** Every script carries a banner at the top:
   ```bash
   # =============================================================================
   # WARNING: UNVERIFIED — never executed on production infrastructure
   # =============================================================================
   # This script has been written and syntax-checked, but has NOT been functionally
   # verified on the target infrastructure (Proxmox Backup Server, production hosts).
   #
   # Before first use:
   # 1. Review the script for correctness against your actual infrastructure
   # 2. Test on a non-production environment
   # 3. Document the verification in rehearsals/records/
   # =============================================================================
   ```

6. **Evidence generation.** Every script emits machine-readable evidence (JSON) to stdout or a specified file. The evidence includes: timestamp, operator (from git config or environment), script version, pass/fail status, details of each check performed, and any artefacts generated (logs, checksums).

7. **Fail-closed helpers.** All scripts source `lib/common.sh`, which provides:
   - `die()` — print error to stderr and exit 1
   - `require_tool <tool>` — check if a command exists, die if not
   - `require_file <path>` — check if a file exists, die if not
   - `require_host <host>` — check if a host is resolvable, die if not
   - `assert <condition>` — check a condition, die if false
   - `log <message>` — timestamped log to stderr
   - `emit_evidence <json>` — write evidence JSON to stdout or file
   - `no_silent_failures` — a function that scans the script for `|| true` and aborts if found (used in CI)

### 3.2 Script Inventory

**`lib/common.sh`** — shared fail-closed primitives (see above). No executable logic, only functions.

**`dr-preflight.sh`** — environment detection. Checks:
- Required tools: `proxmox-backup-client`, `pvesh`, `age`, `sops`, `pg_dump`, `psql`, `ansible-playbook`
- Required hosts: `pve-prod-hv01` (DNS resolution)
- Required directories: `/opt/jol/backups`, `/var/log/jol-deploy`
- Required configuration: `SOPS_AGE_KEY_FILE` (if SOPS-encrypted backups)
- Emits preflight report JSON listing present/absent prerequisites

**`verify-backups.sh`** — read-only backup verification. Checks:
- Backup existence (per `registers/backup-schedule.yml`)
- Backup freshness (not older than RPO threshold: 24h for infra, 1h for DB)
- Backup integrity (`gunzip -t` for `.sql.gz`, `tar -tzf` for `.tar.gz`, `proxmox-backup-client verify` for PBS snapshots)
- Backup encryption (if required by `DR-POL-004`)
- Retention compliance (no backups older than declared retention without explicit exception in `registers/backup-schedule.yml`)
- Never restores anything
- Emits verification report JSON with per-backup pass/fail and details

**`restore-test.sh`** — isolated restore verification. Default behaviour:
- Restores to `/tmp/jol-restore-test/<tenant>-<date>/` (isolated scratch)
- Restores database to isolated PostgreSQL instance (if `pg_dump`/`psql` available)
- Restores files to isolated directory
- Verifies data integrity (row counts, checksums)
- Verifies application health (if app can be started in isolation)
- Cleans up isolated environment
- Emits restore verification report JSON
- Never touches production

With `--allow-destructive`:
- Can restore to production targets (requires explicit confirmation)
- Can overwrite live databases
- Requires operator to specify `--target <tenant>` and `--confirm`

**`dr-drill.sh`** — orchestrator. Runs:
1. `dr-preflight.sh` — abort if prerequisites missing
2. `verify-backups.sh` — verify all backups per schedule
3. `restore-test.sh` — restore a sample tenant to isolated scratch
4. Emits drill evidence JSON aggregating all steps
- Fail-closed: if any step fails, drill fails
- Never touches production unless `--allow-destructive`

**`emit-evidence.sh`** — evidence formatter. Takes JSON from other scripts and:
- Adds metadata: timestamp, operator (from `git config user.name`), script version
- GPG-signs the evidence (using the same key as commits: `609F7926A8254CDB`)
- Writes to `rehearsals/records/` with proper naming: `REHEARSAL-YYYY-NN.md` or `VERIFY-YYYY-MM-DD.json`
- Optionally commits to git with signed commit (if `--commit` flag)

### 3.3 Testing Strategy

**Syntax validation (CI):**
- `bash -n` on all `.sh` files
- `shellcheck` on all `.sh` files (severity: style)
- `no_silent_failures` scan: grep for `|| true` and abort if found

**Fail-closed contract test (CI):**
- Verify that `dr-preflight.sh` aborts non-zero when a required tool is missing (mock by removing a tool from PATH)
- Verify that `verify-backups.sh` aborts non-zero when a backup is missing (mock by pointing to an empty directory)
- Verify that `restore-test.sh` aborts non-zero when `--allow-destructive` is not set and a production target is specified
- Verify that no script prints "PASSED" without a preceding check (grep for `PASSED` and verify it is preceded by a check command)

**Functional testing (NOT on this host):**
- Functional testing requires the target infrastructure (Proxmox Backup Server, production hosts)
- This host has no PBS tooling (`proxmox-backup-client` absent), no production host access (`pve-prod-hv01` does not resolve), and no `/var/log/jol-deploy` directory
- Therefore, functional testing is **impossible** on this host
- Scripts must be tested on the PBS host before first use, and the verification documented in `rehearsals/records/`

**First-use verification:**
- Before any script is used in production, it must be:
  1. Reviewed for correctness against actual infrastructure
  2. Tested on a non-production environment (e.g., staging PBS)
  3. Documented in `rehearsals/records/REHEARSAL-YYYY-NN.md` with status "VERIFIED"
- Until then, the script carries the "UNVERIFIED" banner and must not be used in a real DR event without manual review

**Evidence retention:**
- All evidence JSON is committed to `rehearsals/records/` with a signed commit
- Evidence is retained for the duration specified in `DR-POL-002` (30 days for backup verification, 1 year for rehearsal records)
- Evidence is immutable once committed (no amendments; if a correction is needed, a new evidence file is created with a reference to the original)

### 3.4 CI Workflows

**`workflows/dr-verification.yml`** — runs on every PR and push to main:
- `bash -n` on all scripts
- `shellcheck` on all scripts
- `no_silent_failures` scan
- Fail-closed contract test (mock missing tools, verify abort)
- Verify that all scripts carry the UNVERIFIED banner (until first verification)

**`workflows/backup-freshness.yml`** — runs nightly (cron):
- Runs `verify-backups.sh` against `registers/backup-schedule.yml`
- If any backup is stale (older than RPO threshold), fails the workflow and opens an issue
- Emits evidence to `rehearsals/records/VERIFY-YYYY-MM-DD.json`

**`workflows/compliance-check.yml`** — runs on every PR and push to main:
- Every document has a Document Control block
- Every control in the matrix maps to at least one document
- No document claims a control is "effective" without an evidence artefact
- No fabricated evidence statuses (e.g., "completed" when not performed)

---

## Design Section 4: Cross-Repo Migration, jol-control Updates, and Publish Gates

### 4.1 Cross-Repo Migration (jol-deploy → jol-dr)

**Files to migrate** (authoritative DR content moves to jol-dr):

| jol-deploy (source) | jol-dr (destination) | Action |
|---------------------|----------------------|--------|
| `backup/backup-policy.md` | `policy/backup-policy.md` (DR-POL-002) | Migrate, add Document Control block, supersede jol-deploy copy |
| `backup/rto-rpo.md` | `registers/rto-rpo.yml` | Migrate, convert to YAML, add DB PITR overlay (RPO ≤1h) |
| `backup/disaster-recovery-test.sh` | — | DELETE (false-assurance stub, tracked in FINDING-DR-001) |
| `backup/restore-test.sh` | — | DELETE (false-assurance stub, tracked in FINDING-DR-002) |
| `docs/disaster-recovery/dr-plan.md` | `plans/disaster-recovery-plan.md` | Migrate, expand, fix 4h-RTO-vs-6h-phase overrun, remove "DR team" reference |
| `docs/runbooks/backup-restore.md` | `runbooks/RB-DR-04-single-tenant-restore.md` | Migrate, expand, add Document Control block |
| `docs/runbooks/emergency-procedures.md` | `runbooks/RB-DR-01-total-infrastructure-loss.md` | Migrate, expand, remove "DR team" reference |

**Files to keep in jol-deploy** (deployment-specific, not DR):
- `database/backup-before-migration.sh` — deployment rollback, not DR
- `database/rollback-policy.md` — deployment-specific
- `deployment/strategies/api-fallback-drill.sh` — deployment rollback drill, not DR

**Files to replace with pointers in jol-deploy**:
- `backup/backup-policy.md` → replace with: "Authoritative backup policy is in jol-dr. See `https://github.com/journeyoflife-org/jol-dr/blob/main/policy/backup-policy.md`"
- `backup/rto-rpo.md` → replace with pointer
- `docs/disaster-recovery/dr-plan.md` → replace with pointer
- `docs/runbooks/backup-restore.md` → replace with pointer
- `docs/runbooks/emergency-procedures.md` → replace with pointer

**Migration method**: copy content from jol-deploy to jol-dr staging dir, expand with Document Control blocks, control mappings, and evidence-integrity compliance, then delete jol-deploy originals and replace with short pointers.

### 4.2 jol-control Updates

**`repos/jol-dr.yml`**:
- Change `visibility: public` to `visibility: private`
- Update description to reflect authoritative DR system of record
- Add `compliance.data_classification: "Internal — Operational Security"`

**`policy/compliance-gates.yml`**:
- Add `backup_freshness` gate:
  - ID: `backup-freshness-check`
  - Description: "Fail if backups are stale beyond RPO threshold"
  - Enforcement: mandatory
  - Frameworks: soc2, iso27001
  - SOC2 controls: A1.2, A1.3
  - ISO controls: A.5.29, A.8.13
  - Implementation: `scripts/verify-backups.sh` (read-only)
  - Run on: scheduled (nightly)
  - Remediation: investigate backup failure, restore backup schedule

- Add `rehearsal_schedule` gate:
  - ID: `rehearsal-schedule-check`
  - Description: "Fail if DR rehearsal is overdue per schedule"
  - Enforcement: mandatory
  - Frameworks: soc2, iso27001
  - SOC2 controls: A1.3
  - ISO controls: A.5.29, A.5.30
  - Implementation: check `rehearsals/STATUS.md` for last rehearsal date vs schedule
  - Run on: scheduled (weekly)
  - Remediation: schedule and perform DR rehearsal

**`audit/iso27001-checklist.yml`**:
- Add A.5.29 (Information security during disruption):
  - A.5.29-01: DR plan documented and tested → evidence: `jol-dr/plans/disaster-recovery-plan.md`, `jol-dr/rehearsals/records/`
  - A.5.29-02: Business continuity plan documented → evidence: `jol-dr/plans/business-continuity-plan.md`
  - Status: `implemented` (plan documented), `not_tested` (rehearsal not yet performed)

- Add A.5.30 (Technology readiness for business continuity):
  - A.5.30-01: RTO/RPO targets defined → evidence: `jol-dr/registers/rto-rpo.yml`
  - A.5.30-02: Backup schedule defined and enforced → evidence: `jol-dr/registers/backup-schedule.yml`, `scripts/verify-backups.sh`
  - A.5.30-03: Recovery infrastructure tested → evidence: `jol-dr/rehearsals/records/`
  - Status: `implemented` (targets defined), `not_tested` (recovery not yet tested)

- Add A.8.13 (Information backup):
  - A.8.13-01: Backup policy documented → evidence: `jol-dr/policy/backup-policy.md`
  - A.8.13-02: Backups performed per schedule → evidence: `jol-dr/registers/backup-schedule.yml`, `scripts/verify-backups.sh`
  - A.8.13-03: Backup integrity verified → evidence: `scripts/verify-backups.sh`, `scripts/restore-test.sh`
  - A.8.13-04: Backups encrypted → evidence: `jol-dr/policy/backup-encryption-key-custody.md`
  - Status: `implemented` (policy documented), `not_verified` (integrity not yet verified)

**`audit/soc2-checklist.yml`**:
- Add A1.1 (Capacity planning):
  - A1.1-01: Service inventory with criticality tiers → evidence: `jol-dr/registers/service-inventory.yml`
  - Status: `implemented`

- Add A1.2 (Backup processes and recovery infrastructure):
  - A1.2-01: Backup policy documented → evidence: `jol-dr/policy/backup-policy.md`
  - A1.2-02: Backup schedule enforced → evidence: `jol-dr/registers/backup-schedule.yml`, `scripts/verify-backups.sh`
  - A1.2-03: Recovery infrastructure tested → evidence: `jol-dr/rehearsals/records/`
  - Status: `implemented` (policy documented), `not_tested` (recovery not yet tested)

- Add A1.3 (Recovery plan testing):
  - A1.3-01: Rehearsal schedule defined → evidence: `jol-dr/rehearsals/schedule.yml`
  - A1.3-02: Rehearsals performed per schedule → evidence: `jol-dr/rehearsals/records/`
  - Status: `not_tested` (no rehearsals performed yet)

- Add A1.4 (Effects of system changes during recovery):
  - A1.4-01: DR plan includes verification phase → evidence: `jol-dr/plans/disaster-recovery-plan.md` Phase 4
  - Status: `implemented` (plan documented), `not_tested` (verification not yet performed)

- Add A1.5 (System inventory for disaster recovery):
  - A1.5-01: Service inventory with recovery order → evidence: `jol-dr/registers/service-inventory.yml`, `jol-dr/registers/recovery-order.yml`
  - Status: `implemented`

**Fix CC6.3-02 false evidence**:
- Current: "Terraform state encrypted at rest (S3 SSE)" with evidence `backend 's3' { encrypt = true }`
- Correction: "Terraform state stored locally (terraform.tfstate)" with evidence `main.tf` (no backend block)
- Status: `not_encrypted` (local plaintext state)
- Note: this is a finding in jol-control, not jol-dr, but it must be corrected to maintain audit integrity

### 4.3 Publish Gates (Even Though Private)

Even though jol-dr is private, we still need QA gates before pushing:

**Phase 0: Preflight** (verify current state, tooling, permissions):
- Verify jol-dr repo state (empty, SSH origin, GPG signing enabled)
- Verify gh auth scopes (repo, workflow)
- Verify tooling (markdownlint-cli 0.41.0, pre-commit, detect-secrets, shellcheck)
- Verify jol-control repos/jol-dr.yml visibility (must be private before push)

**Phase 1: Proposal** (tree + control mapping matrix, await approval):
- DONE — tree approved, control mapping approved, evidence-integrity standard approved

**Phase 2: Generation** (write files in staging dir, then cp -a to jol-dr):
- Author all files in `.stage-dr/` inside jol-repo-template workspace
- Copy to jol-dr: `cp -a .stage-dr/. /opt/jol/repos/jol-dr/`
- Generate `.secrets.baseline` with `detect-secrets scan > .secrets.baseline`

**Phase 3: Local QA/Linting**:
- `markdownlint` on all `.md` files
- `pre-commit run --files <all>` across all files
- `shellcheck` on all `.sh` files
- `bash -n` on all `.sh` files
- `no_silent_failures` scan (grep for `|| true`)
- Verify Document Control blocks present in all policies and runbooks
- Verify control-mapping coverage (every control mapped to at least one document)
- Verify evidence-integrity compliance (no fabricated statuses)

**Phase 4: Publish Gate** (explicit user confirmation before push):
- Present QA results
- Present file count and commit summary
- Request explicit GO before `git commit` + `git push`

**Phase 5: Independent Verification**:
- Verify repo state via GitHub API (visibility: private, isEmpty: false, file count)
- Verify CI workflows green (docs-ci, compliance-check, dr-verification)
- Verify jol-control Terraform plan shows no drift (visibility: private matches)

### 4.4 Execution Order (Four Waves)

**Wave 1: Spine + registers + policy + plans + mapping + findings** (~35 files):
- Root governance files (README, SECURITY, CONTRIBUTING, CHANGELOG, LICENSE, Makefile, .editorconfig, .gitignore, .markdownlint.yaml, .pre-commit-config.yaml)
- .github/ (CODEOWNERS, PR template, dependabot, issue templates, workflows/docs-ci.yml, workflows/compliance-check.yml)
- docs/ (index, scope-statement, control-mapping-matrix, document-control, review-schedule, roles-and-responsibilities, evidence-integrity-standard)
- policy/ (DR-POL-001, DR-POL-002, DR-POL-003, DR-POL-004)
- registers/ (rto-rpo.yml, service-inventory.yml, backup-schedule.yml, recovery-order.yml, data-classification-dr.yml, external-dependencies.yml)
- plans/ (disaster-recovery-plan, business-continuity-plan, crisis-communication-plan, recovery-prioritisation)
- findings/ (README + FINDING-DR-001..008)
- QA gate: markdownlint + pre-commit + Document Control presence + control-mapping coverage
- Commit: "docs: seed DR system of record — governance spine, registers, policy, plans, findings"

**Wave 2: Runbooks + rehearsals** (~18 files):
- runbooks/ (README + RB-DR-01..08)
- rehearsals/ (README, schedule.yml, pass-criteria.yml, STATUS.md, templates/×3, records/REHEARSAL-2026-01.md)
- templates/ (runbook-template.md, dr-plan-template.md, control-mapping-template.md)
- QA gate: markdownlint + pre-commit + Document Control presence
- Commit: "docs: add scenario-led runbooks and honest rehearsal programme"

**Wave 3: Scripts harness + CI** (~10 files):
- scripts/ (README, lib/common.sh, dr-preflight.sh, verify-backups.sh, restore-test.sh, dr-drill.sh, emit-evidence.sh)
- .github/workflows/dr-verification.yml, .github/workflows/backup-freshness.yml
- QA gate: shellcheck + bash -n + no_silent_failures + fail-closed contract test
- Commit: "feat: add fail-closed DR harness (UNVERIFIED) and verification workflows"

**Wave 4: Cross-repo migration + jol-control updates + publish** (~15 files across 3 repos):
- jol-deploy: delete migrated files, replace with pointers
- jol-control: update repos/jol-dr.yml (visibility: private), add DR gates to compliance-gates.yml, add A.5.29/A.5.30/A.8.13 to iso27001-checklist.yml, add A1.1-A1.5 to soc2-checklist.yml, fix CC6.3-02 false evidence
- jol-dr: publish gate, commit, push, tag v1.0.0
- Independent verification: GitHub API, CI workflows green, Terraform plan no drift
- Final commit in jol-dr: "chore: publish DR system of record v1.0.0"

Each wave is independently verifiable and publishable. Total: ~79 files in jol-dr + ~15 files across jol-deploy and jol-control.

---

## Assumptions and Constraints

**Assumptions:**

1. The user is a solo operator with no teams. Runbooks must not reference fictional "DR teams" or multi-person workflows.
2. Production infrastructure is Proxmox-based (self-hosted), not AWS. Backup tooling is PBS (Proxmox Backup Server), not S3.
3. This workstation (`/opt/jol`) is a dev/admin box, not the production host. It has no PBS tooling and no network path to production hosts.
4. The user has GPG signing enabled and will sign commits with key `609F7926A8254CDB`.
5. The user has `gh` CLI authenticated with `repo` and `workflow` scopes.
6. The user will approve each wave before execution and will provide explicit GO before any push.

**Constraints:**

1. **No functional testing on this host.** Scripts cannot be functionally verified because PBS tooling is absent and production hosts do not resolve. Scripts must be marked UNVERIFIED and tested on the PBS host before first use.
2. **No fabricated evidence.** Rehearsal status is NOT PERFORMED. I will not generate a completed rehearsal record.
3. **Private repo.** jol-dr must be flipped to private before any content is pushed. The risk acceptance R6 expires at first content push.
4. **Fail-closed harness.** Scripts must abort non-zero on missing tooling, never print "PASSED" unless checks actually succeeded, and never use `|| true`.
5. **Evidence integrity.** No control may be claimed "effective" without evidence of testing. No script may emit a pass signal without performing the check it claims to perform.

---

## Related Documents

- `jol-control/repos/jol-dr.yml` — repository declaration (must be updated to `visibility: private`)
- `jol-control/policy/compliance-gates.yml` — compliance gates (must add DR gates)
- `jol-control/audit/iso27001-checklist.yml` — ISO checklist (must add A.5.29/A.5.30/A.8.13)
- `jol-control/audit/soc2-checklist.yml` — SOC 2 checklist (must add A1.1-A1.5)
- `jol-deploy/backup/*` — current DR content (must be migrated to jol-dr)
- `jol-policies/` — precedent governance repo (convention to mirror)
- `jol-infrastructure/docs/audit/backup-and-recovery-record.md` — git worktree preservation (not DR, do not conflate)

---

**End of spec.**
