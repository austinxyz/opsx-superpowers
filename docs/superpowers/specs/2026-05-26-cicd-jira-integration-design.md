# CI/CD + JIRA Integration — Design

**Date:** 2026-05-26
**Status:** REVIEWED
**Topic:** cicd-jira-integration
**Scope:** Jenkins + JIRA integration with the multi-engineer OpenSpec workflow

---

## Context

The multi-engineer OpenSpec workflow (explore → propose → apply → archive) runs on teams of 2-6 engineers with Jenkins CI and JIRA for ticket tracking. Work originates from both directions: PM creates JIRA tickets that engineers pick up, and engineers self-initiate topics that later get linked to tickets.

Three goals drive this integration:
1. **Visibility** — PM/manager sees which OpenSpec phase a ticket is in without asking engineers
2. **Automation** — JIRA status updates happen automatically; zero manual updates
3. **DoD enforcement** — ticket cannot close until eval passes, CI is green, and archive is complete

---

## Approach: opsx CLI as Integration Layer

Each OpenSpec phase command (`opsx explore`, `opsx propose`, `opsx apply`, `opsx archive`) calls a `jira_hook.py` post-step that updates JIRA. Jenkins is a runner — it executes opsx commands. JIRA integration logic lives in the opsx layer, not the Jenkinsfile.

This makes the integration:
- Testable locally without Jenkins
- Consistent across local dev and CI (same code path)
- Resilient: JIRA API errors are non-fatal; JIRA outage never breaks a build

---

## 1. JIRA Status Mapping

Seven statuses cover the key gates. Blocked is a cross-cutting state reachable from Applying or In Review:

| JIRA Status | Triggered by | Notes |
|---|---|---|
| **Exploring** | `opsx explore` | requirements.md remote link added |
| **Proposing** | `opsx propose` | proposal.md, design.md, tasks.md links added |
| **Spec Review** | PR1 opened (large track) | spec locked label set |
| **Applying** | `opsx apply` | eval-log.md link added (live) |
| **In Review** | PR2 opened | PR URL remote link added |
| **Blocked** | DoD check fails at archive | comment added explaining which check failed; `dod-blocked` label set |
| **Done** | `opsx archive` DoD pass | pitfalls.md + spec archive links added |

Small-track features skip Spec Review (no PR1). The hook reads `track: small|large` from `.meta.yaml` and omits that transition.

---

## 2. Ticket Linking Convention

### `.meta.yaml` — per-topic metadata file

Written by `opsx explore` at topic creation time. Stored at `openspec/topics/<topic>/.meta.yaml`.

```yaml
topic: auth-refactor
created: 2026-05-26
jira_key: PROJ-123        # optional — absent means all JIRA hooks skip silently
jira_project: PROJ
track: large              # small | large
phase: applying           # auto-updated by each opsx command
pr1_url: https://git/...  # set when PR1 opens
pr2_url: null             # set when PR2 opens
```

**`jira_key` is optional.** If absent, `jira_hook.py` exits 0 silently. This supports:
- JIRA-first flow: PM creates ticket → engineer sets `jira_key` at explore time
- OpenSpec-first flow: engineer starts explore → fills `jira_key` later (or never, for internal work)

### JIRA Remote Links added per phase

Each phase adds a remote link to the JIRA ticket pointing to the relevant artifact:

| Phase | Remote links added |
|---|---|
| explore | `requirements.md` |
| propose | `proposal.md`, `design.md`, `tasks.md` |
| apply | `eval-log.md` |
| PR opened | PR URL |
| archive | `pitfalls.md`, spec archived URL |

Links are added via `POST /rest/api/3/issue/{key}/remotelink`.

---

## 3. DoD Enforcement (archive gate)

`opsx archive` runs four checks before calling the JIRA Done transition. Any failure transitions JIRA to Blocked instead.

### Checks

| # | Check | Pass condition | Implementation |
|---|---|---|---|
| 1 | **eval-log.md** | No CRITICAL or HIGH findings | `grep "CRITICAL\|HIGH" eval-log.md` → exit 1 if found |
| 2 | **CI green** | Build passed | `$JENKINS_BUILD_STATUS == SUCCESS` |
| 3 | **PR merged** | On main branch or PR_MERGED flag | `git branch --show-current == main` OR `$PR_MERGED == true` |
| 4 | **Archive files** | pitfalls.md exists + spec.md has DONE status | `test -f pitfalls.md && grep "DONE" spec.md` |

MEDIUM and LOW eval findings are warnings only — they do not block Done.

### On pass

```
POST /rest/api/3/issue/{key}/transitions  → Done
POST /rest/api/3/issue/{key}/remotelink   → pitfalls.md, spec archived
```

### On fail

```
POST /rest/api/3/issue/{key}/transitions  → Blocked
POST /rest/api/3/issue/{key}/comment      → which check failed + details
POST /rest/api/3/issue/{key}/labels       → add dod-blocked
exit 1                                    → Jenkins stage fails
```

---

## 4. Jenkins Integration

### Where each phase runs

| Phase | Runs where | Why |
|---|---|---|
| `opsx explore` | **Local** — engineer's machine | Interactive; requires conversation with Claude |
| `opsx propose` | **Local** — engineer's machine | Interactive; spec authoring is human-driven |
| `opsx apply` | **Jenkins** (or local) | Claude executes on the feat/* branch; can run headless in CI agent |
| `opsx archive` | **Jenkins** post-merge | DoD enforcement runs after PR merges to main |

JIRA hooks fire in both local and Jenkins contexts — `jira_hook.py` reads credentials from env vars that exist in both environments.

### Jenkinsfile skeleton (apply + archive stages only)

```groovy
pipeline {
  parameters {
    string(name: 'TOPIC', defaultValue: '', description: 'OpenSpec topic name')
  }
  environment {
    JIRA_TOKEN    = credentials('jira-api-token')   // Jenkins credentials store
    JIRA_BASE_URL = 'https://yourco.atlassian.net'
    JIRA_USER     = 'ci-bot@yourco.com'
  }

  stages {
    stage('apply') {
      when { branch 'feat/*' }
      environment {
        // TOPIC auto-derived from branch name if not set as parameter
        TOPIC = "${params.TOPIC ?: env.BRANCH_NAME.replaceFirst('feat/', '')}"
      }
      steps { sh 'opsx apply $TOPIC' }
    }
    stage('test') {
      steps {
        sh 'make test'      // unit + integration
        sh 'make e2e'       // E2E (CD pipeline)
      }
    }
    stage('archive') {
      when { branch 'main' }
      environment {
        JENKINS_BUILD_STATUS = "${currentBuild.result ?: 'SUCCESS'}"
        PR_MERGED            = 'true'
      }
      steps { sh 'opsx archive $TOPIC' }
    }
  }
}
```

`TOPIC` is derived from the branch name (`feat/auth-refactor` → `auth-refactor`) or set as an explicit pipeline parameter.

### Credentials

- `JIRA_TOKEN` stored in Jenkins credentials store as Secret Text (`jira-api-token`)
- Injected automatically as env var; same var name works locally if set in shell profile
- No `JIRA_TOKEN` locally → hook exits 0 silently (same as no `jira_key`)

### `jira_hook.py` interface

Called internally by each opsx phase command as a post-step — not called directly from the Jenkinsfile. Engineers do not need to know the hook exists.

```
jira_hook.py <phase> <topic>
```

Reads from:
- `openspec/topics/<topic>/.meta.yaml` — jira_key, pr_urls, track
- `openspec/topics/<topic>/eval-log.md` — severity counts (archive only)
- `$JIRA_TOKEN`, `$JIRA_BASE_URL`, `$JIRA_USER`
- `$JENKINS_BUILD_STATUS`, `$PR_MERGED` (archive only)

Exit codes:
- `0` — success, or `jira_key` absent, or `JIRA_TOKEN` absent
- `1` — DoD check failed (archive only) — Jenkins stage fails
- `2` — JIRA API error (non-fatal) — logged, pipeline continues

---

## 5. File Layout

```
openspec/
├── topics/
│   └── auth-refactor/
│       ├── .meta.yaml          ← jira_key + phase state
│       ├── requirements.md
│       ├── proposal.md
│       ├── design.md
│       ├── tasks.md
│       ├── eval-log.md
│       └── spec.md
├── hooks/
│   └── jira_hook.py            ← single entry point for all JIRA calls
└── config.yaml                 ← JIRA_BASE_URL, JIRA_USER (non-secret defaults)
```

`jira_hook.py` is the single file that owns all JIRA REST calls. Credentials never appear in opsx command files or Jenkinsfile content.

---

## 6. Design Constraints

- **JIRA outage is non-fatal.** Exit code 2 from API errors never breaks a Jenkins build.
- **DoD fail IS fatal at archive only.** No other phase blocks the pipeline on JIRA state.
- **Hook runs after phase files are written.** JIRA sees the new status only after spec artifacts exist.
- **Local dev unaffected.** No `JIRA_TOKEN` or no `jira_key` → silent skip. No configuration required for local OpenSpec usage.
- **Small track skips Spec Review status.** Hook reads `track` from `.meta.yaml` and omits that transition.

---

## Summary

| Component | Role |
|---|---|
| `jira_hook.py` | All JIRA REST calls; reads .meta.yaml + eval-log; enforces DoD |
| `.meta.yaml` | Per-topic state: jira_key (optional), phase, track, PR URLs |
| Jenkinsfile | Pure runner: sets credentials env vars, calls opsx commands |
| JIRA workflow | 6 statuses: Exploring → Proposing → Spec Review → Applying → In Review → Done (+ Blocked) |
| DoD gate | 4 checks at archive: eval-log + CI green + PR merged + archive files |
