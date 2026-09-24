# Design: Create & Configure 11 JOL Governance Repositories

- **Date:** 2026-09-24
- **Owner:** Gintaras Kazlauskas (JOL Platform)
- **Status:** Approved — ready for implementation
- **Target org:** `journeyoflife-org` (GitHub Free)
- **Local root:** `/opt/jol/repos`
- **Host:** Ubuntu 24.04, git 2.43.0, gh 2.100.0, Python 3.12.3

## 1. Objective

Create 11 new **empty** GitHub repositories under `journeyoflife-org`, create the
matching 11 local directories under `/opt/jol/repos`, and configure each local
working copy for GitHub (SSH), PyCharm, and Qoder — following the exact convention
already used by existing governance repos (`jol-compliance`, `jol-security`,
`jol-infrastructure`, `jol-devops`).

## 2. Scope — the 11 repositories

`jol-control`, `jol-policies`, `jol-docs`, `jol-iac`, `jol-dr`, `jol-privacy`,
`jol-incident-response`, `jol-payments-scope`, `jol-status`,
`jol-compliance-evidence`, `jol-secrets`.

Pre-flight state verified on 2026-09-24:
- **Local:** none of the 11 directories exist under `/opt/jol/repos` (30 other `jol-*` repos present).
- **GitHub:** none of the 11 repositories exist in `journeyoflife-org`.

## 3. Confirmed decisions

| Decision | Choice | Notes |
|---|---|---|
| Visibility | **All public** | User-accepted risk; see §4. |
| GitHub content | **Truly empty / bare** | No commits, no README, no `.gitignore` on GitHub. |
| Remote protocol | **SSH** | `git@github.com:journeyoflife-org/<repo>.git` (matches 25/30 existing repos). |
| Local depth | **Match existing repos** | `.venv` (py3.12) + `.idea/*` mirroring `jol-compliance`. |
| Push content | **No** | GitHub stays empty; `.idea/`+`.venv/` are gitignored (local-only). |
| Auto-open IDEs | **No** | User chose "Match existing repos", not "+ open all now". |

## 4. Risk register & acceptance

| ID | Risk | Severity | Recommendation | Decision |
|---|---|---|---|---|
| R1 | Public `jol-secrets` reveals secrets-management design/paths | High | Private | **Accepted public** by owner 2026-09-24 |
| R2 | Public `jol-privacy` (DPIA/GDPR processing) may create disclosure exposure | High | Private | **Accepted public** |
| R3 | Public `jol-payments-scope` reveals PCI-DSS CDE boundaries | High | Private | **Accepted public** |
| R4 | Public `jol-compliance-evidence` exposes audit findings | Medium | Private | **Accepted public** |
| R5 | Public `jol-incident-response` reveals IR process/runbooks | Medium | Private | **Accepted public** |
| R6 | Public `jol-control` / `jol-dr` / `jol-iac` reveal controls, DR posture, infra topology | Medium | Private | **Accepted public** |

**Mitigations / standing guidance:**
- Repos are created **empty**; risk only materialises once sensitive content is committed. Keep actual secrets out of git entirely (use a vault / SOPS); `.gitignore` blocks `*.pem`, `*.key`, `secrets.json`, `credentials.json`.
- **Private→public is one click; public→private after a leak is damage control.** Re-evaluate visibility before pushing content to R1–R6 repos.
- Enable GitHub **secret scanning / push protection** on the org.

## 5. Design — per-repository build steps (idempotent, re-runnable)

For each `<repo>` in the 11:

1. **Create GitHub repo (bare, public):**
   `gh repo create journeyoflife-org/<repo> --public`
   (no `--clone`, no `--add-readme`, no template → empty repo).
2. **Create local working copy linked via SSH.** Preferred: clone the empty repo so
   `origin` and branch tracking are set exactly by git:
   `git clone git@github.com:journeyoflife-org/<repo>.git /opt/jol/repos/<repo>`
   Fallback (if clone of empty repo is undesirable): `mkdir -p`, `git init -b main`,
   `git remote add origin git@github.com:journeyoflife-org/<repo>.git`.
3. **Python virtualenv:** `python3.12 -m venv /opt/jol/repos/<repo>/.venv`.
4. **PyCharm config** — write `.idea/` mirroring `jol-compliance` (SDK name
   `Python 3.12 (<repo>)`): `vcs.xml`, `misc.xml`, `modules.xml`, `<repo>.iml`,
   `.idea/.gitignore`. (See §6.)
5. **Root `.gitignore`** — copy from `jol-repo-template/.gitignore` (ignores `.idea/`,
   `.venv/`, caches, and secret file patterns).
6. **No commit, no push.** Local files stay uncommitted so GitHub remains empty;
   `.idea/`+`.venv/` are gitignored regardless.
7. **Qoder** — no hand-authored config: Qoder auto-registers the project under
   `~/.qoder/shared_client/projects/-opt-jol-repos-<repo>` on first open.

**Safety/idempotency:** before creating, check `gh repo view` (skip/​report if the
GitHub repo exists) and `[ -d /opt/jol/repos/<repo> ]` (skip/​report if the local dir
exists). Never delete or overwrite existing data. GPG signing is available
non-interactively, but no commits are made, so signing is not triggered.

## 6. Exact PyCharm config templates

`.idea/vcs.xml`
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project version="4">
  <component name="VcsDirectoryMappings">
    <mapping directory="$PROJECT_DIR$" vcs="Git" />
  </component>
</project>
```

`.idea/misc.xml`
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project version="4">
  <component name="Black">
    <option name="sdkName" value="Python 3.12 (REPO)" />
  </component>
  <component name="ProjectRootManager" version="2" project-jdk-name="Python 3.12 (REPO)" project-jdk-type="Python SDK" />
</project>
```

`.idea/modules.xml`
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project version="4">
  <component name="ProjectModuleManager">
    <modules>
      <module fileurl="file://$PROJECT_DIR$/.idea/REPO.iml" filepath="$PROJECT_DIR$/.idea/REPO.iml" />
    </modules>
  </component>
</project>
```

`.idea/REPO.iml`
```xml
<?xml version="1.0" encoding="UTF-8"?>
<module type="PYTHON_MODULE" version="4">
  <component name="NewModuleRootManager">
    <content url="file://$MODULE_DIR$">
      <excludeFolder url="file://$MODULE_DIR$/.venv" />
    </content>
    <orderEntry type="jdk" jdkName="Python 3.12 (REPO)" jdkType="Python SDK" />
    <orderEntry type="sourceFolder" forTests="false" />
  </component>
</module>
```

`.idea/.gitignore` — standard JetBrains ignore (`/workspace.xml`, `/shelf/`, `/httpRequests/`, `/queries/`, `/dataSources/`, `/dataSources.local.xml`).

> `REPO` is replaced with the repository name (e.g. `jol-control`). The interpreter
> SDK `Python 3.12 (REPO)` is registered by PyCharm automatically when the folder is
> opened and `.venv` is detected; until then PyCharm may briefly show the interpreter
> as unresolved (self-heals on first open).

## 7. Verification plan (end-to-end)

For all 11, assert and tabulate:
1. **GitHub:** `gh repo view journeyoflife-org/<repo> --json url,visibility,isEmpty` → exists, `visibility=PUBLIC`, empty (no commits).
2. **Local dir:** exists at `/opt/jol/repos/<repo>`.
3. **Git:** `git -C <repo> remote get-url origin` == SSH URL; branch `main`.
4. **Venv:** `<repo>/.venv/bin/python --version` == `Python 3.12.3`.
5. **PyCharm:** `.idea/{vcs.xml,misc.xml,modules.xml,<repo>.iml,.gitignore}` exist and are well-formed XML.
6. Produce a summary table: repo × {github url, visibility, remote, venv, idea} and an overall PASS/FAIL.

## 8. Out of scope

- No repository content (README, LICENSE, CI, code) — repos stay empty.
- No `git push` / no commits.
- No auto-launching of PyCharm or Qoder windows.
- No changes to the existing 30 repos or to `jol-repo-template`.
- No org-level GitHub settings changes (secret scanning recommended separately, not applied here).

## 9. Rollback

- **GitHub:** `gh repo delete journeyoflife-org/<repo>` (requires `delete_repo` scope; current token lacks it — deletion would be manual via web UI or a token with the scope).
- **Local:** remove `/opt/jol/repos/<repo>` directories created by this task.
- Because nothing is pushed and repos are empty, rollback carries no data-loss risk.
