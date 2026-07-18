# OpenSpec × Signadot — Integration Spec

> **Status:** VALIDATED E2E 2026-07-18 on hotrod-opsx (`pickup-confirmation`) — see the
> "Validation results" section below. Skill-interface sections 1–2 are SUPERSEDED by
> Signadot's official agent skills; kept for history.  
> **Author:** Austin  
> **Companion doc:** [`workflow-signadot.md`](workflow-signadot.md) — conceptual overview  
> **This doc:** concrete skill interfaces and artifact formats for implementation

---

## Validation results (2026-07-18, hotrod-opsx / pickup-confirmation)

The full cycle ran end-to-end: explore → propose (plan draft) → apply (TDD +
`2.V VALIDATE` on real cluster austin-staging-1 → evaluator consumed the verdict
as Runtime evidence) → archive (plan registered into `openspec/specs/<cap>/plans/`).
Reference implementation: https://github.com/austinxyz/hotrod branch `opsx-setup`.

### Official skills replace the planned wrappers

Signadot ships official agent skills at https://github.com/signadot/agent-skills
(`npx skills add signadot/agent-skills`): **`signadot-plan`** (author/run plan specs)
and **`signadot-validate`** (sandbox + routing-key validation workflow). The
`signadot-plan` / `signadot-validate` wrappers this spec asked Joe to implement
already exist — opsx commands now invoke the official skills directly.

### Real plan model (supersedes the `kind: TestPlan` guess below)

A plan is an immutable compiled DAG of **action invocations**, discoverable at
runtime — not a static YAML schema:

- `signadot plan schema` returns the live JSON schema; top-level fields:
  `selectionHint`, `params`, `cluster`, `runner`, `steps`, `output`
- Steps reference org-catalog actions **by `action.actionID`** (from
  `signadot plan action get <name>`), not by name; catalog: `check`, `eval`,
  `request-http`, `k6`, `playwright`, `run-container`
- Params/eval-outputs need explicit `schema` for refs to drill in; step ordering
  is dependency-driven via `args.refs` / `extraInputs` (array order ≠ execution order)
- `selectionHint` is a first-class top-level field (open question 2: answered)
- Unbound params: declare without `default`; bind at run:
  `signadot plan run <plan-id> --param k=v` (open question 3: answered)
- Output/verdict: per-check outputs via `signadot plan x get-output <exec-id>
  <step>/result` (JSON: `{"check": name}` on pass, + `error.message` on fail);
  execution JSON carries per-step phase (open question 4: answered)
- Runtime prerequisite: **Managed Plan Runners** enabled per cluster in the
  dashboard (Platform → Managed Runners). A `jobrunnergroup` does NOT satisfy
  "no plan runner group on cluster".
- `request-http` auto-injects routing-key headers when the step has
  `routingContext` (open question on header plumbing: answered)

### Bugs / friction for Joe

1. **Image allowlist appears broken**: with run-container images allowlisted in
   Platform → Managed Runners → Configure Actions (tried both shorthand and
   fully-qualified `docker.io/...` forms, Apply saved), `plan create` still
   rejects every image — including k6's own system image `grafana/k6:1.7.1` —
   with "not in this org's allowed-images list". Only actionbox-backed actions
   (request-http / eval / check) are usable. Org: austinxyza.
2. **No sleep/wait primitive in plans**: async flows (Kafka processing) need
   polling. `request-http` exits 1 on transport timeout, so blackhole-URL delays
   fail the step; we resorted to chaining request-http steps against
   `httpbin.org/delay/4` as spacers. A `wait`/`sleep` action (or retry policy on
   request-http) would remove the hack.
3. **No Windows CLI build** (all releases darwin/linux; source build blocked by
   private `signadot/libconnect`). Docker image works as a wrapper, but
   `auth login` fails in-container (`dbus-launch` keyring) — config.yaml
   fallback works. A windows binary or documented docker-wrapper path would help.
4. Docs gap: "Enabling Plan Runner Groups" is referenced in release notes but
   the page 404s; the dashboard toggle was found by trial.

### Remaining open questions

- Dry-run mode (original question 5) — still unknown
- Plan ↔ group cardinality (original question 8) — unexercised; pickup-confirmation
  was 1 plan : 1 group

---

---

## Division of work

| Responsible | Deliverable |
|-------------|-------------|
| **Austin** (opsx-superpowers) | This spec; `signadot-plan` skill wrapper; `signadot-validate` skill wrapper; updated `workflow-signadot.md` |
| **Joe (Signadot)** | Implements skill wrappers against Signadot API; provides sandbox cluster; runs e2e to validate the flow |

---

## Core model (from Ani's clarification)

No plan stubs. A plan is authored at **propose time** with **unbound parameters** (URLs, payloads, service endpoints). Parameters are **bound at apply time** when the real implementation exists.

```
propose:  author plan with params: {baseUrl: ?, payload: ?}
            ↓
  apply:  bind params → run signadot-validate → verdict
```

---

## Skill 1: `signadot-plan`

**Phase:** propose  
**Trigger:** agent calls this skill for each integration-critical Scenario in `specs/<cap>/spec.md`

### Input

```yaml
# What the agent passes to the skill
behavior_id: <kebab-case-id>          # e.g. "ride-request-itinerary"
selection_hint: <prose>               # natural language: what behavior this plan validates
behavior_description: <prose>         # what the user-visible behavior is, end-to-end
task_groups: [N, M, ...]              # which task group numbers this plan covers
cap: <capability-name>                # which capability this belongs to
```

### What the skill does

1. Generates a parameterized plan YAML at `openspec/changes/<topic>/signadot-plans/<behavior-id>.yaml`
2. Updates the relevant `### Contract` block in `tasks.md`: sets `Runtime: validated by signadot plan <behavior-id>`
3. The plan YAML contains declared params with no bound values — filled at apply N.V VALIDATE

### Output artifact: `signadot-plans/<behavior-id>.yaml`

```yaml
# Questions for Joe: is this the right schema? What fields are required?
apiVersion: signadot.com/v1
kind: TestPlan
metadata:
  name: <behavior-id>
  selectionHint: "<prose from behavior_description — used by agent to match plan to diff>"
spec:
  params:
    baseUrl: ""          # unbound at propose; bound at apply
    authToken: ""        # unbound at propose; bound at apply
    # ... other params TBD based on action types used
  steps:
    - action: request-http           # or playwright, k6, etc.
      params:
        url: "{{ params.baseUrl }}/endpoint"
        method: POST
        body: "{{ params.payload }}"
      assertions:
        - statusCode: 200
        # ... behavior-specific assertions
```

**Open question for Joe:** Is `selectionHint` a first-class field in Signadot plan schema, or is it metadata/annotation?

---

## Skill 2: `signadot-validate`

**Phase:** apply — step `N.V VALIDATE`, runs after `N.X+1 GREEN`, before `N.E EVAL`  
**Trigger:** agent calls this skill for each task group that has a bound plan in its Contract block

### Input

```yaml
plan_id: <behavior-id>               # matches the plan file authored at propose
topic: <topic>                       # to locate signadot-plans/<id>.yaml
param_bindings:                      # concrete values, known now that implementation exists
  baseUrl: "https://..."
  authToken: "..."
  payload: "..."
  # ... other params from the plan
```

### What the skill does

1. Reads `openspec/changes/<topic>/signadot-plans/<plan-id>.yaml`
2. Merges `param_bindings` into the plan
3. Invokes `signadot-validate` CLI with the materialized plan
4. Signadot self-summons an ephemeral env, runs the plan against the real cluster, tears down
5. Returns a structured verdict to the agent

### Output: validate verdict (written to `eval-log.md`)

```yaml
# Questions for Joe: what does the actual signadot-validate output look like?
plan_id: <behavior-id>
timestamp: <iso8601>
env_url: "https://..."               # ephemeral env URL (for audit trail)
run_url: "https://..."               # link to Signadot dashboard run
status: pass | fail
assertions:
  - name: "statusCode is 200"
    result: pass | fail
    expected: 200
    actual: 404
  # ...
summary: "<prose from Signadot>"
```

### Gate semantics

| Verdict | N.E EVAL Runtime score | Action |
|---------|------------------------|--------|
| All assertions pass | 100% (full 40% weight) | Continue to EVAL |
| Any assertion fails | 0% → Runtime floored | Immediate BLOCK → FIX tasks → retry (max 3) |

The evaluator subagent reads the verdict from `eval-log.md` as the Runtime evidence. No guessing.

---

## Artifact layout

```
openspec/changes/<topic>/
  tasks.md                          ← Contract.Runtime = "validated by signadot plan <id>"
  contracts/
    group-N.md
  signadot-plans/                   ← NEW directory, pre-created at propose
    <behavior-id>.yaml              ← authored at propose (unbound params)
  eval-log.md                       ← N.V VALIDATE verdict appended here

openspec/specs/<capability>/
  spec.md
  plans/                            ← NEW: durable plan library (registered at archive)
    <behavior-id>.yaml              ← validated plan, moved here after archive
```

---

## Workflow delta (tasks.md template additions)

At propose, groups with integration-critical behaviors get:

```markdown
### Contract
Spec:      SHALL statements covered by this group
Runtime:   validated by signadot plan `<behavior-id>`
Code:      files / interfaces touched
Threshold: 80

- [ ] N.0 CONTRACT — write contracts/group-N.md; confirm signadot plan id matches
- [ ] N.1 RED — ...
- [ ] N.2 GREEN — ...
- [ ] N.V VALIDATE — run `signadot-validate` skill with param bindings; append verdict to eval-log.md
- [ ] N.E EVAL — evaluator reads validate verdict as Runtime evidence; score Spec/Runtime/Code
```

Groups without a plan keep `Runtime: <prose>` and the N.V step is omitted.

---

## Project prerequisite (one-time)

Not a per-topic task — treated like "CI is configured":

- Signadot account connected to Kubernetes cluster
- Action catalog present (typed building blocks: `request-http`, `playwright`, `k6`, etc.)
- Documented in `openspec/config.yaml`:

```yaml
# openspec/config.yaml addition
integrations:
  signadot:
    enabled: true
    cluster: <cluster-name>
    action_catalog: signadot-actions/
```

---

## Open questions for Joe

1. **Plan schema** — Is the YAML format above close to Signadot's actual plan schema? What fields are required vs optional?
2. **`selectionHint`** — First-class field or annotation/label?
3. **Unbound parameters** — What's the syntax for declaring a param with no value? Empty string `""`? Sentinel like `null` or `~`?
4. **`signadot-validate` CLI** — What does the actual output look like? JSON? YAML? What fields are stable?
5. **Dry-run mode** — Is there a `--dry-run` flag to test the plan without spinning up an ephemeral env? (Useful for developing the skill wrappers.)
6. **Sandbox cluster** — Can Signadot provide a sandbox for Austin to test against during implementation?
7. **Action catalog location** — Per-project or shared/global? How does a new project bootstrap one?
8. **Plan ↔ group cardinality** — A plan may cover behaviors across multiple task groups. When the verdict comes back, it feeds the Runtime of all covered groups. Does Signadot have a concept of "this run covered groups N and M"? Or does Austin need to track that mapping explicitly?

---

> Next step: Joe reviews this doc, answers open questions, and implements the skill wrappers.  
> Austin updates the spec based on Joe's answers, then wires the skills into the opsx-superpowers slash commands.
