# jol-iac — Versioned OpenTofu Library + Policy-as-Code (Design Spec)

- **Date:** 2026-09-25
- **Status:** Approved — spec reviewed; D6 (public visibility with placeholder-only mitigation) ratified by owner 2026-09-25; ready for implementation planning
- **Owner:** Journey of Life (JOL) — journeyoflife-org (sole owner, `solo_mode`)
- **Repo:** `jol-iac` (Governance tier, `visibility: public`, `data_classification: confidential`)
- **Anchor controls:** ISO 27001:2022 **A.8.9** (configuration management) · SOC 2 **CC8.1** (change management)
- **Secondary controls:** SOC 2 CC6.1/CC7.1 · ISO A.8.24/A.8.13/A.8.8/A.8.19 · ISO A.5.29/A.5.30 + SOC 2 A1.x (availability/backup gap) · GDPR Art. 5(1)(f)/25/32, Art. 9 (processor role)
- **Approach:** B — declarative library (modules + templates + policy-as-code + full CI); SOPS Gate 4–6 delivered as **runbooks/checklists/templates only** (no executable scripts).

## 1. Context & Problem

`jol-iac` is provisioned but **empty** (`.git` with no commits, `.gitignore`, `.idea`, `.venv`). Its control-plane definition ([jol-control/repos/jol-iac.yml](file:///opt/jol/repos/jol-control/repos/jol-iac.yml)) declares it "Infrastructure as Code templates and shared modules", governance tier, frameworks `soc2/gdpr/iso27001`.

Three gaps this repo closes:

1. **No canonical, versioned, reviewable IaC home.** `jol-infrastructure` is Ansible/Shell-first; its `terraform/` + `helm/charts/` are **legacy AWS EKS** ([AGENTS.md §2.5](file:///opt/jol/repos/jol-infrastructure/AGENTS.md) drift alert: "reality is 100% on-prem Proxmox"), and it has **committed `.terraform/` provider binaries + vendored `.external_modules`** — a CC6.1/CC8.1 hygiene violation.
2. **The SOPS Gate 4–6 infrastructure has no operational home.** The SOPS+age rollout is defined in [adr-003-amendment-sops-age.md](file:///opt/jol/repos/jol-infrastructure/docs/architecture/adr-003-amendment-sops-age.md) (**status PROPOSED**) with `sops-recipients-registry.md` as Gate 7 and gate audits under `jol-infrastructure/docs/audit`. Gate 4 (install), Gate 5 (age identity), Gate 6 (Shamir DR) need reviewable runbooks/templates in a versioned governance repo.
3. **Availability/backup controls are unmanaged.** jol-control's `iso27001-checklist.yml` has no A.5.29/A.5.30/A.8.13 entries and `soc2-checklist.yml` has no A1.1–A1.5 despite claiming "SOC 2 Type II". A reusable Proxmox backup/DR module + control mapping directly addresses this.

## 2. Goals / Non-Goals

**Goals**
- Reusable, **empirically validated** Proxmox-native OpenTofu modules + copy-to-consume templates.
- Policy-as-code (OPA/conftest rego **+ unit tests**) encoding the org baseline (no secrets in HCL, required tags, no public ingress, resource limits).
- Canonical SOPS Gate 4–6 **operational artifacts** (runbooks, checklists, templates, tool pins) — inert by construction.
- Full governance wrapper + CI mapped to the governance-tier gates in [compliance-gates.yml](file:///opt/jol/repos/jol-control/policy/compliance-gates.yml).
- Control-traceability matrix (A.8.9 / CC8.1 / A1.x / Art. 32 …).

**Non-Goals (YAGNI)**
- No deployable stack that provisions the live fleet (host provisioning stays in `jol-infrastructure` Ansible).
- No executable SOPS scripts, no age key generation, no tool installation.
- No edits to any other repo (`jol-control`, `jol-infrastructure`); supersession is documented by ADR only.
- No AWS resources.

## 3. Decisions

| # | Decision | Choice | Rationale |
|---|---|---|---|
| D1 | IaC tool | **OpenTofu** (`.tofu-version`), `required_version >= 1.8.0`, Terraform-compatible HCL | Terraform is BUSL-1.1 (absent from `allowed_licenses`); OpenTofu is MPL-2.0 + supports client-side state encryption |
| D2 | Provider | **`bpg/proxmox`**, version pinned at generation, recorded in `docs/tool-versions.md` + committed `.terraform.lock.hcl` | Proxmox is the real platform; bpg/proxmox is actively maintained |
| D3 | State | **Local-first**; examples/templates validated via `tofu init -backend=false && tofu validate`, **never applied**; documented path to self-hosted **MinIO/S3-compatible** remote + locking for consumers | Matches fleet convention; no AWS; state never in git |
| D4 | License | **MIT** (matches [jol-repo-template/LICENSE](file:///opt/jol/repos/jol-repo-template/LICENSE); in `allowed_licenses`) | Reusable library; permissive |
| D5 | CODEOWNERS | **Mirror template** team refs (`platform-core`/`security`/`docs`) | Established governance convention, even though teams don't exist for the solo owner |
| D6 | Visibility handling | Repo stays **public** per `jol-iac.yml` (**ratified by owner 2026-09-25**); therefore **placeholders only** (no real hostnames/IPs/VMIDs/PBS paths/key locations) | `jol-dr` was flipped private for exactly this exposure risk; jol-iac mitigates by strict placeholder policy |
| D7 | SOPS scope | **Docs/templates only** (Approach B); amendment is PROPOSED, so gates stay human-executed behind their own approval Conditions | Prevents building automation for unapproved, human-gated key ceremonies |

## 4. Repository Structure

```text
jol-iac/
├── README.md · LICENSE (MIT) · SECURITY.md · CONTRIBUTING.md · CHANGELOG.md · CODE_OF_CONDUCT.md
├── .editorconfig · .gitignore · .markdownlint.yaml · .secrets.baseline · .pre-commit-config.yaml
├── Makefile                 # install validate fmt lint policy scan docs secrets check clean
├── .tofu-version            # pinned OpenTofu version
├── modules/                 # Proxmox-native reusable OpenTofu modules (empirically validated)
│   ├── proxmox-vm/          # proxmox_virtual_environment_vm + cloud-init + MANDATORY tags
│   ├── proxmox-firewall/    # firewall rules: default-deny, VLAN-scoped ingress
│   ├── proxmox-pool/        # resource pool + tag inheritance (governance grouping)
│   └── proxmox-backup/      # PBS backup job/datastore (see §6.1 empirical rule)
│       └── each: main.tf variables.tf outputs.tf versions.tf README.md examples/{basic,complete}/
├── templates/
│   ├── env-stack/           # consumer copy-paste root: backend.tf.tmpl providers.tf.tmpl tfvars.example main.tf
│   └── module-template/     # skeleton that passes policy by construction (versions.tf, tags, README, examples)
├── policy/
│   └── opa/                 # conftest rego + *_test.rego (CI-runnable, no infra needed)
│       ├── no_secrets_in_tf.rego · required_tags.rego · no_public_ingress.rego · resource_limits.rego
├── sops/                    # Gate 4–6 — DOCS/TEMPLATES ONLY (Approach B)
│   ├── README.md            # banner: amendment PROPOSED; human-executed; private keys NEVER committed
│   ├── runbooks/gate-4-install-sops.md · gate-5-age-identity.md · gate-6-shamir-dr.md
│   ├── checklists/gate-4-6-conditions.md
│   ├── templates/sops.yaml.tmpl · age-jol.recipients.tmpl   # PUBLIC key placeholders only
│   └── tool-versions.md     # pinned sops version + SHA256, age version
├── docs/
│   ├── architecture.md · consuming-modules.md · state-and-backends.md · tool-versions.md
│   ├── control-mapping-matrix.md
│   └── adr/  README.md · ADR-001-opentofu-over-terraform.md · ADR-002-proxmox-target.md
│             · ADR-003-state-strategy.md · ADR-004-supersedes-legacy-aws-terraform.md
└── .github/
    ├── CODEOWNERS · dependabot.yml · PULL_REQUEST_TEMPLATE.md
    ├── ISSUE_TEMPLATE/{bug_report.yml, feature_request.yml}
    └── workflows/  compliance-scan.yml · codeql.yml
```

Excluded per governance convention: `pyproject.toml`, `src/`, `tests/`, `qodana.yaml`. `.idea/`, `.venv/` stay local (gitignored).

## 5. Governance Wrapper

Mirror the seeded reference (`jol-policies`) + template:
- **README.md** — purpose, scope/non-goals, structure, module + policy catalog, consumer quick-start, SOPS Gate 4–6 pointer, compliance summary.
- **SECURITY.md** — private vuln reporting; secret handling (SOPS+age dual identity, private keys never committed); public+confidential caveat.
- **CONTRIBUTING.md** — module-authoring standard (versions.tf pins, required tags, README, examples), policy contribution, PR + change-control checklist, signed commits.
- **CHANGELOG.md** — Keep-a-Changelog + evidence references (§0.3 convention); initial seed entry.
- **CODE_OF_CONDUCT.md**, **.editorconfig**, **.markdownlint.yaml**, **.secrets.baseline** — mirror `jol-policies`.
- **Makefile** — venv-based, targets: `install validate fmt lint policy scan docs secrets check clean`; markdownlint via `npx markdownlint-cli@0.41.0` (Node 20 pin).
- **.pre-commit-config.yaml** — drop ruff/mypy (no Python); keep pre-commit-hooks (trailing-whitespace, end-of-file-fixer, check-yaml, check-json, check-merge-conflict, check-added-large-files 500kb, detect-private-key, mixed-line-ending=lf), markdownlint v0.41.0, detect-secrets (baseline), forbid-crlf, forbid-tabs (exclude Makefile). Heavy IaC scanning is authoritative in CI (keeps local deps light).
- **dependabot.yml** — ecosystems `github-actions` + `terraform`.

## 6. Components

### 6.1 modules/ (empirical-validation rule)
Target set: `proxmox-vm`, `proxmox-firewall`, `proxmox-pool`, `proxmox-backup` (PBS). **Every module must pass `tofu init -backend=false && tofu validate` against the live `bpg/proxmox` schema before commit** (Empirical Terraform Validation Requirement — never assume resource args). If a resource (notably the PBS backup job) is not supported by the pinned provider schema, that module is **reduced to a `templates/` scaffold + tracked item** rather than committed as a fake module (fail-closed).
- Common module contract: `versions.tf` (OpenTofu + provider pin), required-tag variables (`CostCenter`, `Environment`, `Owner`) with `validation` blocks, no hardcoded secrets, `README.md` (generated by `tofu-docs`), `examples/{basic,complete}/` validated in CI but never applied, example values are placeholders (VMID `9000+`, RFC1918 doc ranges, `CHANGEME`).

### 6.2 templates/
- **env-stack/** — consumer root wiring modules; `backend.tf.tmpl` documents local-first + MinIO/S3 remote with locking/encryption/versioning and a bold "NEVER commit state" note; `providers.tf.tmpl`; `terraform.tfvars.example`.
- **module-template/** — authoring skeleton that satisfies policy + validation by construction (A.8.9 consistency).

### 6.3 policy/opa/
conftest rego mirroring `jol-infrastructure`'s rule set, adapted to HCL:
- `no_secrets_in_tf.rego` — deny hardcoded passwords/tokens/private keys in `.tf`.
- `required_tags.rego` — require `CostCenter`/`Environment`/`Owner` (matches `iac_policy` gate rule).
- `no_public_ingress.rego` — deny `0.0.0.0/0` on sensitive ports.
- `resource_limits.rego` — require cpu/memory bounds on VMs.
- Each ships `*_test.rego` fixtures; CI runs `opa test policy/opa -v --coverage` (no infra needed) + `conftest test --parser hcl` over examples. Plan-based conftest is documented as the **consumer's** responsibility (needs live Proxmox creds).

### 6.4 sops/ (Gate 4–6, docs/templates only)
Prominent banner: the ADR-003 amendment is **PROPOSED**; no gate executes until its Conditions are met; **private key material is never committed**; runbooks are **human-executed under change control** with fail-closed verification + evidence capture.
- **runbooks/gate-4-install-sops.md** — install with version + SHA256 checksum verification, preflight conditions, evidence, rollback.
- **runbooks/gate-5-age-identity.md** — `age-jol` identity (dual-identity segregation: `age-jol` for `/opt/jol`, `age-jolarca` for `/opt/jolarca`; no cross-tree recipients); Vaultwarden = source of truth; workstation cache perms `600` file / `700` dir; recipients published via security-reviewed PR.
- **runbooks/gate-6-shamir-dr.md** — Shamir **M=2/N=3** (`ssss`); custody log records locations only; quarterly DR rehearsal (reconstruct from 2, verify pubkey, decrypt fixture).
- **checklists/gate-4-6-conditions.md** — the amendment's go/no-go Conditions (amendment APPROVED; CC8.1 issue w/ rollback; DPIA re-check; two-identity custody plan decided BEFORE keygen; reconciled with jol-hub prior SOPS work).
- **templates/sops.yaml.tmpl** — `creation_rules` restricted to `*.enc.yaml`, `secrets/**`; age recipients = **public** key placeholder; segregation note.
- **templates/age-jol.recipients.tmpl** — public key placeholder only.
- **tool-versions.md** — pinned `sops` version + SHA256, `age` version.
- Cross-references (not copies): `adr-003-amendment-sops-age.md`, `sops-recipients-registry.md` (Gate 7), `jol-infrastructure/docs/audit/gate*`.

### 6.5 docs/ + ADRs
`architecture.md`, `consuming-modules.md`, `state-and-backends.md`, `tool-versions.md`, `control-mapping-matrix.md`. ADRs follow the **immutability rule** (new ADRs; supersede don't rewrite; jol-infrastructure ADRs are referenced, never edited):
- **ADR-001** OpenTofu over Terraform (BUSL vs MPL; state encryption; LF governance).
- **ADR-002** Proxmox target (reality per AGENTS.md §2.5; no AWS).
- **ADR-003** State strategy (local-first + MinIO remote path; never in git).
- **ADR-004** Supersedes legacy AWS terraform direction (non-destructive; migration is separate change-controlled work).

## 7. Framework / Control Mapping

| Control | Implementation in jol-iac |
|---|---|
| ISO **A.8.9** config mgmt | Versioned modules/templates, pinned tool+provider versions, committed lock file, policy-enforced baselines, ADRs |
| SOC 2 **CC8.1** change mgmt | PR-based flow + signed commits + CHANGELOG w/ evidence + ADRs; SOPS gates require a CC8.1 change-issue; CI gates |
| SOC 2 **CC6.1** logical access | No secrets in git (detect-secrets + trufflehog + OPA `no_secrets_in_tf`); SOPS dual identity |
| SOC 2 **CC7.1** / ISO A.8.8 | IaC SAST: checkov, tflint, trivy, conftest; CodeQL on `actions` |
| ISO **A.5.29/A.5.30** + SOC 2 **A1.x** availability/backup | `proxmox-backup` (PBS) module + DR/backup control rows in `control-mapping-matrix.md` (fills the jol-control gap) |
| ISO **A.8.24** cryptography / **A.8.13** segregation / GDPR Art. 32 | SOPS+age templates/runbooks; dual-identity segregation; SHA256-pinned tooling |
| ISO **A.8.19** patch mgmt | Dependabot (`github-actions` + `terraform`), provider pins |
| GDPR **Art. 9** (processor role) | No personal data in examples; DPIA-trigger note; JOL=processor, tenant=controller documented |

## 8. CI / Required Status Contexts

`jol-iac.yml` declares required contexts `compliance-scan / validate` and `codeql-analysis`. **Under active `solo_mode`, `required_status_checks` and `required_pull_request_reviews` are omitted** (verified: `status_checks: NONE`, `pr_reviews: NONE`, `signed_commits: true`), so the sole owner pushes directly to `main`; CI still runs on push for evidence. We build CI to the **full governance standard** so flipping `solo_mode=false` restores enforcement with no rework.

- **compliance-scan.yml** (`name: compliance-scan`): job `validate` → emits `compliance-scan / validate` (required-files check, license headers, `tofu fmt -check`, `tofu validate` on examples/templates, `tflint`, `checkov`); sibling jobs `policy` (`opa test` + `conftest`), `secrets` (trufflehog + detect-secrets), `iac-scan` (`trivy config`), `dependencies` (dependency-review-action).
- **codeql.yml**: emits `codeql-analysis` via **`language: actions`** (CodeQL natively scans GitHub Actions workflows for script-injection) — meaningful with zero Python; no jol-control change needed. Follow-up note: add `actions` to `compliance-gates.yml` `sast.languages`.

## 9. Secrets, SOPS Boundary & .gitignore Spec

`.gitignore` must **commit** `.terraform.lock.hcl` and `.sops.yaml` (reproducibility/public config) and **strictly prohibit**:
`.terraform/`, `*.tfstate`, `*.tfstate.backup`, `*.tfvars` (allow `*.tfvars.example`), `crash*.log`, provider binaries; age private keys (block `keys.txt`, `age-*.txt`; allow `*.pub`); decrypted outputs (`*.dec.*`, non-`*.enc.*` under `secrets/`); backup artifacts (`*.bundle`, `*.sql.gz`, `*.tar.gz`, `*.zst`, `recovery-artifacts/`); plus `.venv/`, `.idea/`, `__pycache__/`.
Examples use **placeholders only** (public + confidential). This is the inverse of the two live defects (§10) — jol-iac must not repeat them.

## 10. Cross-Repo Boundaries, Non-Destructive Rule & Flagged Defects

- `jol-control` owns the GitHub org, branch protection (`solo_mode`), and `repos/jol-iac.yml` (source of truth — **not edited here**).
- `jol-infrastructure` keeps Ansible/inventory/host docs + ADR-003/amendment + gate audits; its AWS `terraform/`+`helm/` are legacy, superseded-by-note (ADR-004) only.
- **Flagged, NOT touched (separate change-controlled remediation):** (a) `jol-control` committed `terraform.tfstate`/`.tfstate.backup`/`terraform.tfvars`; (b) `jol-infrastructure/terraform` committed `.terraform/` provider binaries + `.external_modules`.
- All automation follows the **fail-closed contract**: abort non-zero on missing prerequisites, no `|| true`, never print PASSED without real verification, read-only defaults, UNVERIFIED banners until first functional test, machine-readable evidence.

## 11. Validation / QA Strategy (Phase 3)

Local (`make check`) + CI: `tofu fmt -check -recursive`; per example/template `tofu init -backend=false && tofu validate` (**empirical, against real bpg/proxmox schema**); `tflint --recursive`; `checkov -d .`; `opa test policy/opa -v --coverage`; `conftest test --parser hcl examples/`; `trivy config .`; `markdownlint` (0.41.0) + link-check; `detect-secrets`/`trufflehog`; `pre-commit run --all-files`. **Examples/templates are validated, never applied.** Validation failures are critical blockers. <!-- pragma: allowlist secret -->

## 12. Execution Model & Seeding (Phases 2–5)

- **Subagent-driven execution**: implement in waves (wrapper → modules → policy → sops/docs → CI), dispatching `code-reviewer` after each wave, with a **user checkpoint between waves**.
- **Preflight (Phase 0/1)**: verify OpenTofu + bpg/proxmox availability, `tofu`/`tflint`/`checkov`/`opa`/`conftest`/`trivy` presence; confirm `jol-iac` still empty; resolve provider/tool versions.
- **Publish (Phase 4 gate)**: commit **GPG-signed** to `main` and push (direct push permitted under `solo_mode`) **only after explicit user confirmation**; then **Phase 5 independent verification** (`gh api` repo state, CI runs green, live-clone `tofu validate`).

## 13. Risks & Open Items

1. **Public + confidential** — mitigated by strict placeholder-only policy + OPA secret rules; if real topology must be documented, revisit visibility in `jol-control/repos/jol-iac.yml` (separate change).
2. **bpg/proxmox backup-job schema** — verify empirically; fall back to template + tracked item (§6.1).
3. **CODEOWNERS teams don't exist** — mirrored per convention; code-owner review is inert for the solo owner (acceptable under `solo_mode`).
4. **SOPS amendment unapproved** — repo ships inert artifacts only; cannot itself execute Gate 4–6.

## 14. Acceptance Criteria

- Tree matches §4; governance required-files present; no Python packaging.
- Every module/example passes `tofu fmt -check` + `tofu init -backend=false && tofu validate` (evidence captured).
- `opa test` passes with coverage; `conftest`, `checkov`, `tflint`, `trivy` clean (or baselined with justification).
- `.gitignore` blocks state/keys/binaries/backup artifacts; `detect-secrets`/`trufflehog` clean; `.terraform.lock.hcl` committed.
- CI emits `compliance-scan / validate` and `codeql-analysis`; `markdownlint` clean.
- Control-mapping matrix present; ADRs follow immutability; CHANGELOG has evidence-linked seed entry.
- Commits GPG-signed; push only after Phase 4 confirmation; Phase 5 independent verification green.

## 15. References

- [jol-control/repos/jol-iac.yml](file:///opt/jol/repos/jol-control/repos/jol-iac.yml) · [compliance-gates.yml](file:///opt/jol/repos/jol-control/policy/compliance-gates.yml)
- [adr-003-amendment-sops-age.md](file:///opt/jol/repos/jol-infrastructure/docs/architecture/adr-003-amendment-sops-age.md) · [AGENTS.md §0.1/§0.2/§2.5](file:///opt/jol/repos/jol-infrastructure/AGENTS.md)
- [jol-repo-template](file:///opt/jol/repos/jol-repo-template) (wrapper + CI) · [jol-policies](file:///opt/jol/repos/jol-policies) (seeded governance reference)
- Conventions: OpenTofu/Proxmox stack · local-first state · IaC gitignore hygiene · ADR immutability · fail-closed scripts · empirical Terraform validation · `solo_mode` branch protection · subagent-driven execution · staged publish workflow.
