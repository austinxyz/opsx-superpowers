# OpenSpec × Signadot — Integration Spec

> **Status:** DRAFT — for review with Joe (Signadot Engineering), week of 2026-06-30  
> **Author:** Austin  
> **Companion doc:** [`workflow-signadot.md`](workflow-signadot.md) — conceptual overview  
> **This doc:** concrete skill interfaces and artifact formats for implementation

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
