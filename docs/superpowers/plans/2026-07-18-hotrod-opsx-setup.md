# HotROD-OPSX Setup Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Stand up the `hotrod-opsx` fork with Signadot-extended opsx-superpowers skills and both feature specs, ready for `/opsx:explore pickup-confirmation` in a fresh session.

**Architecture:** Two repos change. (1) `opsx-superpowers` (branch `signadot`): the schema template and four opsx command files gain Signadot plan/validate wiring per [signadot-integration-spec.md](../../signadot-integration-spec.md). (2) `hotrod-opsx` (new clone of the `austinxyz/hotrod` fork): gets opsx scaffolding installed, project config, CLAUDE.md, and the two feature specs. Feature development itself is NOT in this plan — it runs via `/opsx:*` in the new session.

**Tech Stack:** OpenSpec CLI 1.2.x, opsx-superpowers schema/commands (markdown), gh CLI, Go (hotrod, context only).

## Global Constraints

- opsx-superpowers work happens on branch `signadot` — never on `main`
- Command-file edits must keep the existing tone/format of those files (numbered steps, guardrails section)
- Signadot terminology must match the integration spec verbatim: plan ids are kebab-case behavior ids; the Contract binding line is `validated by signadot plan \`<behavior-id>\``
- Groups WITHOUT a bound plan must behave exactly as before (all Signadot steps are opt-in per group)
- Windows host: run `bin/opsx-install` via Git Bash (`bash bin/opsx-install`), not PowerShell

---

### Task 1: Fork and clone hotrod → hotrod-opsx

**Files:**
- Create: `C:\Users\lorra\projects\hotrod-opsx\` (git clone)

**Interfaces:**
- Produces: local repo `hotrod-opsx` with remote `origin` → `https://github.com/austinxyz/hotrod`, used by Tasks 6–7

- [ ] **Step 1: Fork on GitHub (no local clone from gh)**

```bash
gh repo fork signadot/hotrod --clone=false
```

Expected: `austinxyz/hotrod` created (or "already exists" — fine either way).

- [ ] **Step 2: Clone the fork as hotrod-opsx**

```bash
cd /c/Users/lorra/projects
git clone https://github.com/austinxyz/hotrod hotrod-opsx
```

- [ ] **Step 3: Verify remotes and add upstream**

```bash
cd /c/Users/lorra/projects/hotrod-opsx
git remote add upstream https://github.com/signadot/hotrod
git remote -v
```

Expected: `origin` → austinxyz/hotrod, `upstream` → signadot/hotrod.

---

### Task 2: Signadot wiring — tasks.md template

**Files:**
- Modify: `schemas/superpowers-driven/templates/tasks.md` (opsx-superpowers repo)

**Interfaces:**
- Produces: template lines `N.V VALIDATE` and Runtime plan-binding comment consumed by propose.md (Task 3) and apply.md (Task 4)

- [ ] **Step 1: Add plan-binding option to the group-2 Contract Runtime comment**

In `schemas/superpowers-driven/templates/tasks.md`, edit the group **2** Contract block (the feature-work exemplar). Replace:

```markdown
- **Runtime**: <!-- test command -->` → expected: <!-- passing state -->
```

(exact old string is `- **Runtime**: \`<!-- test command -->\` → expected: <!-- passing state -->`) with:

```markdown
- **Runtime**: `<!-- test command -->` → expected: <!-- passing state -->
  <!-- Integration-critical group (cross-service, user-visible behavior)? Use instead:
       validated by signadot plan `<behavior-id>` (plan yaml in signadot-plans/, authored at propose with unbound params) -->
```

- [ ] **Step 2: Add the optional N.V VALIDATE line to group 2**

In the same file, replace:

```markdown
- [ ] 2.4 VISUAL DIFF — bring up dev stack (use project.dev_stack_command from openspec/config.yaml); navigate to the route; eyeball rendered UI against the mock; fix any token/color/text drift
- [ ] 2.E EVAL — spawn evaluator subagent (haiku); reads contracts/group-2.md + spec + design + group diff; invokes superpowers:requesting-code-review (CRITICAL/HIGH = BLOCK); scores Spec/Runtime/Code; total ≥ threshold → PASS; < threshold → append FIX tasks + retry (max 3 attempts, plateau < 5pt = escalate)
```

with:

```markdown
- [ ] 2.4 VISUAL DIFF — bring up dev stack (use project.dev_stack_command from openspec/config.yaml); navigate to the route; eyeball rendered UI against the mock; fix any token/color/text drift
<!-- Only if this group's Contract Runtime binds a signadot plan: -->
- [ ] 2.V VALIDATE — bind params in openspec/changes/{{change}}/signadot-plans/<behavior-id>.yaml (URLs, payloads now known); run signadot-validate against the real cluster; append the structured verdict to eval-log.md; any failed assertion = Runtime floored → treat as BLOCK
- [ ] 2.E EVAL — spawn evaluator subagent (haiku); reads contracts/group-2.md + spec + design + group diff; invokes superpowers:requesting-code-review (CRITICAL/HIGH = BLOCK); scores Spec/Runtime/Code; if a 2.V verdict exists in eval-log.md, Runtime score = that verdict (pass=100, fail=0), not subagent judgment; total ≥ threshold → PASS; < threshold → append FIX tasks + retry (max 3 attempts, plateau < 5pt = escalate)
```

- [ ] **Step 3: Verify template renders no accidental breakage**

```bash
cd /c/Users/lorra/projects/opsx-superpowers
grep -n "2.V VALIDATE" schemas/superpowers-driven/templates/tasks.md
grep -n "validated by signadot plan" schemas/superpowers-driven/templates/tasks.md
```

Expected: one hit each, inside group 2.

- [ ] **Step 4: Commit**

```bash
git add schemas/superpowers-driven/templates/tasks.md
git commit -m "feat(schema): add optional signadot plan binding + N.V VALIDATE to tasks template"
```

---

### Task 3: Signadot wiring — propose.md

**Files:**
- Modify: `commands/opsx/propose.md`

**Interfaces:**
- Consumes: template markers from Task 2
- Produces: propose-time artifacts `signadot-plans/<behavior-id>.yaml` (unbound params) that apply.md (Task 4) binds and runs

- [ ] **Step 1: Add step 3b after step 3a (Contract blocks)**

In `commands/opsx/propose.md`, insert after the 3a section (immediately before the line `### 4. After proposal generation: branch on HAS_UI_SURFACE`):

```markdown
### 3b. Signadot plans for integration-critical groups (optional per group)

Skip this step entirely if `integrations.signadot.enabled` is absent or false in `openspec/config.yaml`.

A group is **integration-critical** when its behavior spans services and is user-visible end-to-end (the kind that passes unit tests but breaks the system). For each such group:

1. Pre-create the plans directory (parallel to `contracts/`):

   ```bash
   mkdir -p openspec/changes/<topic>/signadot-plans
   ```

2. Author `openspec/changes/<topic>/signadot-plans/<behavior-id>.yaml` — a **parameterized plan with unbound params** (no concrete URLs/payloads yet; they don't exist until apply):

   ```yaml
   apiVersion: signadot.com/v1
   kind: TestPlan
   metadata:
     name: <behavior-id>                # kebab-case, one user-visible behavior
     selectionHint: "<prose: what this plan validates — lets an agent match plan to diff>"
   spec:
     params:                            # declared, unbound — bound at apply N.V
       baseUrl: null
       # ...one entry per value unknown until implementation exists
     steps:
       - action: <request-http | playwright | k6>
         params:
           url: "{{ params.baseUrl }}<endpoint>"
         assertions:
           - <behavior-specific assertion>
   ```

3. Rewrite that group's Contract **Runtime** field to the binding form:

   ```
   - **Runtime**: validated by signadot plan `<behavior-id>`
   ```

4. Ensure the group has an `N.V VALIDATE` task between its last GREEN (or VISUAL DIFF) and `N.E EVAL` (the template shows the form at 2.V).

Groups that are NOT integration-critical keep the plain test-command Runtime and get no plan and no N.V task.
```

- [ ] **Step 2: Add guardrail**

In the same file's **Guardrails** list, append:

```markdown
- Signadot plans are propose-phase artifacts (what correct means) — author the yaml with unbound params here; NEVER fill in concrete URLs/payloads at propose. Binding happens at apply N.V.
```

- [ ] **Step 3: Verify**

```bash
grep -n "3b. Signadot plans" commands/opsx/propose.md
grep -n "unbound params" commands/opsx/propose.md
```

Expected: 3b heading present; ≥ 2 mentions of unbound params.

- [ ] **Step 4: Commit**

```bash
git add commands/opsx/propose.md
git commit -m "feat(propose): author parameterized signadot plans for integration-critical groups"
```

---

### Task 4: Signadot wiring — apply.md

**Files:**
- Modify: `commands/opsx/apply.md`

**Interfaces:**
- Consumes: `signadot-plans/<behavior-id>.yaml` (unbound) from Task 3; `N.V VALIDATE` task form from Task 2
- Produces: validate verdict entries in `eval-log.md` consumed by the evaluator subagent

- [ ] **Step 1: Add N.V dispatch to the task-prefix list**

In `commands/opsx/apply.md`, in the "For each task, dispatch by prefix" list, insert a new bullet immediately before the `- **\`- [ ] N.E EVAL — ...\`**` bullet:

```markdown
- **`- [ ] N.V VALIDATE — ...`** → only present when the group's Contract binds a signadot plan. (1) Open `openspec/changes/<name>/signadot-plans/<behavior-id>.yaml`; bind every declared param with the now-known concrete values (URLs, payloads, tokens). (2) Run `signadot-validate` with the bound plan — it self-summons an ephemeral env, runs against the real cluster, tears down. If the CLI contract is not yet available, run the documented manual/scripted equivalent and capture the same fields. (3) Append the verdict to `eval-log.md`:

  ```yaml
  - group: N
    validate:
      plan: <behavior-id>
      status: pass | fail
      env_url: <ephemeral env URL if available>
      assertions: [{name: "...", result: pass|fail}]
  ```

  (4) `status: fail` → treat exactly like an evaluator BLOCK: pause, report failed assertions, offer fix/skip/abort. Do not proceed to N.E with a failed verdict. Mark the checkbox only on pass or after the user chooses to proceed.
```

- [ ] **Step 2: Wire the verdict into the evaluator prompt**

In the **Evaluator Subagent** section, replace step 4 of the evaluator prompt:

```markdown
> 4. Score Runtime 0–100 (100 = all tests pass, 0 = test command fails to run).
```

with:

```markdown
> 4. Score Runtime 0–100. FIRST check `eval-log.md` for a `validate:` entry for this group: if present, Runtime = 100 when its status is pass, 0 when fail — the real-cluster verdict overrides your own run. Only if no validate entry exists: run the Runtime test command from the contract (100 = all tests pass, 0 = test command fails to run).
```

- [ ] **Step 3: Add guardrail**

Append to the Guardrails list:

```markdown
- DO run N.V VALIDATE before N.E EVAL in groups that bind a signadot plan — the evaluator reads the verdict as Runtime evidence; skipping N.V reverts Runtime to a guess, which defeats the binding.
```

- [ ] **Step 4: Verify**

```bash
grep -n "N.V VALIDATE" commands/opsx/apply.md
grep -n "real-cluster verdict overrides" commands/opsx/apply.md
```

Expected: dispatch bullet + guardrail hit; evaluator prompt hit.

- [ ] **Step 5: Commit**

```bash
git add commands/opsx/apply.md
git commit -m "feat(apply): N.V VALIDATE step binds signadot plan and feeds EVAL Runtime"
```

---

### Task 5: Signadot wiring — archive.md + config-template.yaml

**Files:**
- Modify: `commands/opsx/archive.md`
- Modify: `config-template.yaml`

**Interfaces:**
- Produces: durable plan library convention `openspec/specs/<capability>/plans/<behavior-id>.yaml`; config key `integrations.signadot.enabled` consumed by propose.md step 3b (Task 3)

- [ ] **Step 1: Add plan registration to archive**

In `commands/opsx/archive.md`, insert a new section between `### 6. Cleanup step 4 — conditional project README` and `### 7. Dev log check`:

```markdown
### 6b. Cleanup step 5 — register signadot plans (only if present)

If `<archived-dir>/signadot-plans/` contains plan yamls, move each validated plan into the owning capability's durable library:

```bash
mkdir -p openspec/specs/<capability>/plans
git mv openspec/changes/archive/<date>-<name>/signadot-plans/<behavior-id>.yaml openspec/specs/<capability>/plans/
```

The accumulating `selectionHint` catalog under `openspec/specs/*/plans/` is the versioned plan library — future changes touching the same behavior reuse these plans instead of authoring from scratch. If a plan's behavior failed final validation or was descoped, delete it instead of registering it; note why in the commit message.
```

- [ ] **Step 2: Update the step-8 cleanup commit to include plans**

In the same file, replace:

```bash
git add openspec/specs/ CLAUDE.md README.md docs/log/
```

with:

```bash
git add openspec/specs/ openspec/changes/archive/ CLAUDE.md README.md docs/log/
```

(the `git mv` above stages both sides; the broadened add keeps the atomic-commit guardrail intact).

- [ ] **Step 3: Add signadot block to config-template.yaml**

Append to `config-template.yaml` (top level, after the `project:` block, before `context:`):

```yaml
integrations:
  signadot:
    enabled: false               # true once Signadot Operator is installed on the cluster (one-time bootstrap, never a per-topic task)
    cluster: ""                  # Signadot cluster name, e.g. "austin-staging-1"
    # Prerequisites (bootstrap, like "CI is set up"):
    #  - Signadot Operator installed and cluster connected in app.signadot.com
    #  - action catalog available (request-http, playwright, k6, ...)
```

- [ ] **Step 4: Verify**

```bash
grep -n "6b. Cleanup step 5" commands/opsx/archive.md
grep -n "integrations:" config-template.yaml
```

Expected: one hit each.

- [ ] **Step 5: Commit and push the signadot branch**

```bash
git add commands/opsx/archive.md config-template.yaml
git commit -m "feat(archive,config): durable signadot plan library + integration config"
git push -u origin signadot
```

---

### Task 6: Install opsx scaffolding into hotrod-opsx

**Files:**
- Create (in `hotrod-opsx`): `.claude/commands/opsx/{explore,propose,apply,archive}.md` (via installer), `openspec/config.yaml`, `CLAUDE.md`, `docs/superpowers/specs/` (dir)
- Global: `%LOCALAPPDATA%/openspec/schemas/superpowers-driven` (via installer)

**Interfaces:**
- Consumes: Task 1's clone; Tasks 2–5's extended schema/commands (installer copies CURRENT opsx-superpowers state)
- Produces: working `/opsx:*` commands + `openspec/config.yaml` with `integrations.signadot.enabled: true` for the new session

- [ ] **Step 1: Run the installer from hotrod-opsx root**

```bash
cd /c/Users/lorra/projects/hotrod-opsx
bash /c/Users/lorra/projects/opsx-superpowers/bin/opsx-install
```

Expected output: `✓ superpowers-driven schema installed ...` and `✓ opsx commands installed to .claude/commands/opsx`.

- [ ] **Step 2: Verify install**

```bash
ls .claude/commands/opsx
grep -n "3b. Signadot plans" .claude/commands/opsx/propose.md
openspec schemas 2>/dev/null | grep superpowers-driven || echo "openspec CLI missing — install: npm i -g @fission-ai/openspec"
```

Expected: 4 files; the signadot section present (proves Tasks 3–5 landed before install); schema listed. If openspec CLI is missing, install it and re-check.

- [ ] **Step 3: Write openspec/config.yaml**

Create `openspec/config.yaml` in hotrod-opsx:

```yaml
schema: superpowers-driven

project:
  dev_stack_command: "kubectl -n hotrod port-forward svc/frontend 8080:8080"
  test_commands:
    - "go test ./..."
  e2e_command: ""
  design_system: "custom"        # hotrod ships its own embedded web UI; no external design system
  custom_verification_checks: []

integrations:
  signadot:
    enabled: true
    cluster: "austin-staging-1"

context: |
  HotROD — Signadot's fork of the Jaeger demo app. Go 1.x microservices:
  frontend (HTTP :8080, embedded web UI, Kafka producer, notification polling),
  driver (Kafka consumer, health :8082, calls location + route),
  location (HTTP :8081, MySQL), route (gRPC :8083), redis (notifications), kafka, mysql.
  Cross-service calls: services/frontend/dispatcher.go (location.Get, Kafka publish),
  services/driver/consumer.go processDispatchRequest (driverStore, bestETA, notifications),
  pkg/notifications/handler.go (Redis store/list, 30s SetEx TTL).
  Tests: go test ./... from repo root.
  Deploy: baseline runs on local kind cluster (context kind-hotrod) namespace hotrod,
  manifests k8s/overlays/prod/devmesh; Signadot Operator installed, cluster austin-staging-1.
  Sandbox pattern: fork the modified service (e.g. driver) in a Signadot sandbox against
  baseline frontend/redis — see signadot.com/docs.

rules: {}
```

- [ ] **Step 4: Write CLAUDE.md**

Create `CLAUDE.md` in hotrod-opsx:

```markdown
# hotrod-opsx

Fork of signadot/hotrod used as the reference example for the OpenSpec + Superpowers × Signadot
integration (collaboration with Signadot — Ani/Joe). Remote `origin` = austinxyz/hotrod,
`upstream` = signadot/hotrod.

## Workflow

All feature work goes through `/opsx:explore → /opsx:propose → /opsx:apply → /opsx:archive`.
Signadot plan/validate wiring is enabled (`openspec/config.yaml` → integrations.signadot).
Integration spec: https://github.com/austinxyz/opsx-superpowers/blob/signadot/docs/signadot-integration-spec.md

## Environment

- kind cluster `hotrod` (kubectl context `kind-hotrod`), baseline deployed in ns `hotrod`
  from `k8s/overlays/prod/devmesh`
- Signadot cluster name: `austin-staging-1` (Operator installed in ns `signadot`)
- Frontend UI: `kubectl -n hotrod port-forward svc/frontend 8080:8080` → http://localhost:8080
- Tests: `go test ./...`

## Planned changes

1. `pickup-confirmation` (Feature 2) — spec: docs/superpowers/specs/2026-07-18-pickup-confirmation-design.md (APPROVED)
2. `driver-status` (Feature 1) — spec: docs/superpowers/specs/2026-07-18-driver-status-design.md (DRAFT — brainstorm before development)

## Pitfalls

(populated by /opsx:archive)
```

- [ ] **Step 5: Commit (in hotrod-opsx, on a new branch)**

```bash
cd /c/Users/lorra/projects/hotrod-opsx
git checkout -b opsx-setup
git add .claude/ openspec/ CLAUDE.md
git commit -m "chore: install opsx-superpowers scaffolding with signadot integration"
```

---

### Task 7: Save both feature specs into hotrod-opsx

**Files:**
- Create (in `hotrod-opsx`): `docs/superpowers/specs/2026-07-18-pickup-confirmation-design.md`, `docs/superpowers/specs/2026-07-18-driver-status-design.md`

**Interfaces:**
- Consumes: approved design at `opsx-superpowers/docs/superpowers/specs/2026-07-18-hotrod-pickup-confirmation-design.md`
- Produces: the two specs `/opsx:explore` will draw requirements from in the new session

- [ ] **Step 1: Copy Feature 2 (approved) design in**

```bash
mkdir -p /c/Users/lorra/projects/hotrod-opsx/docs/superpowers/specs
cp /c/Users/lorra/projects/opsx-superpowers/docs/superpowers/specs/2026-07-18-hotrod-pickup-confirmation-design.md \
   /c/Users/lorra/projects/hotrod-opsx/docs/superpowers/specs/2026-07-18-pickup-confirmation-design.md
```

Then edit the copied file's `**Home:**` line to:

```markdown
**Home:** this repo (hotrod-opsx). Authored during brainstorm in opsx-superpowers on 2026-07-18.
```

- [ ] **Step 2: Write Feature 1 DRAFT spec**

Create `docs/superpowers/specs/2026-07-18-driver-status-design.md`:

```markdown
# Design: HotROD Driver Status Tracking (DRAFT)

**Status:** DRAFT — needs its own brainstorm pass before development. Do not /opsx:propose from this.
**Date:** 2026-07-18
**Depends on:** pickup-confirmation (reuses services/driver dispatchstore)

## Sketch (from service-map analysis, Option 1)

User-visible: frontend shows driver availability — "Driver busy" / "Driver accepting rides".

- Span: frontend ↔ driver ↔ location
- Driver publishes status transitions (dispatched → busy → available) building on the
  dispatch records introduced by pickup-confirmation
- Frontend surfaces status via the existing notification/polling mechanism or a new
  lightweight status endpoint (DECIDE at brainstorm)

## Open questions for the brainstorm

1. Status storage: extend `dispatch:<id>` records vs a separate `driver:<id>:status` key
2. Who transitions status back to available — timer, explicit API, or ride-complete event?
3. Does location service need on-duty/off-duty awareness, or is that scope creep? (YAGNI check)
4. Signadot plan: which cross-service assertion proves the behavior end-to-end?
```

- [ ] **Step 3: Commit**

```bash
cd /c/Users/lorra/projects/hotrod-opsx
git add docs/superpowers/specs/
git commit -m "docs: add pickup-confirmation (approved) and driver-status (draft) specs"
```

---

### Task 8: Update tracking README + handoff instructions

**Files:**
- Modify: `signadot/README.md` (opsx-superpowers repo)

**Interfaces:**
- Consumes: everything above
- Produces: the kickoff card for the new session

- [ ] **Step 1: Update the checklist and add next-session kickoff**

In `opsx-superpowers/signadot/README.md`, replace:

```markdown
- [ ] Feature candidate chosen
- [ ] OpenSpec change opened for the feature
```

with:

```markdown
- [x] Feature candidate chosen — Feature 2: Pickup Confirmation (approved design), then Feature 1: Driver Status (draft)
- [x] Fork created: austinxyz/hotrod → local clone C:\Users\lorra\projects\hotrod-opsx (branch opsx-setup)
- [x] opsx skills extended for signadot plan/validate (opsx-superpowers branch signadot)
- [x] opsx scaffolding installed in hotrod-opsx (config: integrations.signadot.enabled)
- [ ] OpenSpec change opened for the feature

## Next session (in C:\Users\lorra\projects\hotrod-opsx)

1. `/opsx:explore pickup-confirmation` — requirements distill from
   docs/superpowers/specs/2026-07-18-pickup-confirmation-design.md (design already approved;
   explore should be fast: draft requirements → review → REVIEWED)
2. `/opsx:propose pickup-confirmation` — expect step 3b to author
   signadot-plans/pickup-confirmation-arrival.yaml with unbound params
3. `/opsx:apply pickup-confirmation` — N.V VALIDATE binds params and runs against
   austin-staging-1 (manual/scripted until Joe's signadot-validate CLI answers land)
4. `/opsx:archive pickup-confirmation` — registers the plan into openspec/specs/*/plans/
```

- [ ] **Step 2: Commit and push**

```bash
cd /c/Users/lorra/projects/opsx-superpowers
git add signadot/README.md
git commit -m "docs: mark setup complete; add next-session kickoff for pickup-confirmation"
git push
```

---

## Self-Review Notes

- **Spec coverage:** 5-step plan → Task 1 (step 1), Tasks 2–5 (step 3), Task 6 (step 2), Task 7 (step 4), Task 8 (handoff to step 5). Step 5 itself (feature development) is intentionally out of scope — it runs via /opsx in the new session.
- **Ordering:** Tasks 2–5 (extend skills) MUST precede Task 6 (install copies current state). Task 1 is independent and can run any time before Task 6.
- **Type consistency:** binding line `validated by signadot plan \`<behavior-id>\`` identical across Tasks 2/3/4; verdict YAML shape in Task 4 matches integration-spec's output section (subset: same field names).
- **No placeholders:** every edit shows exact old/new text; config and CLAUDE.md contents complete.
