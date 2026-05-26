# Harness Validation Design

**Date:** 2026-05-25  
**Branch:** feat/harness-validation  
**Status:** APPROVED  
**Reference:** [Anthropic Engineering — Harness Design for Long-Running Apps](https://www.anthropic.com/engineering/harness-design-long-running-apps)

---

## Problem

Current `apply` flow runs a single agent for both generation and validation. The article's core finding: generators self-assess poorly — confident praise even for mediocre output. `requesting-code-review` and `verification-before-completion` both run inside the same agent context, so the same context bias affects them.

---

## Solution Overview

Introduce a **per-group harness loop** inside `apply`:

1. **Contract** — apply agent writes explicit success criteria before implementation starts
2. **Generate** — RED → GREEN as before
3. **Evaluate** — separate evaluator subagent (fresh context, skeptical lens) scores the output
4. **Retry** — if score below threshold, evaluator generates FIX tasks; apply agent executes them and re-evaluates
5. **Escalate** — after 3 attempts or score plateau, pause for human decision

---

## Architecture

### Flow per task group

```
Group N:

  [N.0 CONTRACT]
    apply agent writes contracts/group-N.md
    content: Spec SHA statements, Runtime test commands, Code review points, Threshold

  [N.1 RED → N.X GREEN]
    apply agent implements as before (TDD enforced by superpowers:test-driven-development)

  [N.E EVAL]  ← replaces N.Z requesting-code-review
    spawn evaluator subagent (haiku model)
    context: contracts/group-N.md + spec.md + design.md + git diff (group only)
    evaluator steps:
      1. invoke superpowers:requesting-code-review → CRITICAL/HIGH = immediate BLOCK
      2. run Runtime test commands from contract
      3. diff vs Spec SHA statements → Spec score
      4. aggregate: Runtime score + Code score (no CRITICAL/HIGH)
      5. write to eval-log.md: {group, attempt, scores, findings, fix_tasks}
      6. return BLOCK | PASS | RETRY

  on BLOCK:
    pause immediately, report CRITICAL/HIGH findings, wait for human

  on PASS (total ≥ threshold):
    continue to next group

  on RETRY (total < threshold):
    append N.F1 FIX, N.F2 FIX … tasks to tasks.md
    apply agent executes FIX tasks
    re-run N.E EVAL (attempt + 1)

  escalate when:
    attempt >= 3
    OR (last_score - prev_score) < 5
    → write eval-log.md summary, pause with score history
```

### Retry loop (apply agent manages)

```
attempt = 1
while attempt <= 3:
  result = spawn_evaluator(group=N, attempt=attempt)
  if result == BLOCK:
    pause → report to user
    break
  if result == PASS:
    advance to group N+1
    break
  # RETRY
  if attempt >= 3 or score_delta < 5:
    pause → "Group N failed after {attempt} attempts. Last score: {score}. Options: manual fix / skip / abort."
    break
  append FIX tasks to tasks.md
  attempt += 1
```

---

## Scoring

### Dimensions

| Dimension | What it checks | Weight |
|---|---|---|
| Spec | diff satisfies all SHALL statements in contract | 40% |
| Runtime | test commands from contract pass independently | 40% |
| Code | requesting-code-review findings (no CRITICAL/HIGH) | 20% |

### Thresholds

- Default: **80**
- UI-touching groups: **70** (visual judgment has inherent subjectivity)
- CRITICAL/HIGH code finding: **immediate BLOCK** regardless of total score

### Score output format (eval-log.md entry)

```yaml
- group: 1
  attempt: 1
  scores:
    spec: 85
    runtime: 60
    code: 90
  total: 73
  status: RETRY
  findings:
    - "runtime: test_auth_flow fails — token not persisted across requests"
  fix_tasks:
    - "1.F1 FIX — persist token in session store, not local variable"
```

---

## File Artifacts

```
openspec/changes/<topic>/
  contracts/
    group-1.md        ← written by N.0 CONTRACT task
    group-2.md
    ...
  eval-log.md         ← appended after each evaluator run
```

### contracts/group-N.md format

```markdown
# Group N Contract

## Spec
- SHALL <statement from spec.md>
- SHALL <statement from spec.md>

## Runtime
- Command: `<from openspec/config.yaml test_commands>`
- Expected: <description of passing state>

## Code
- <key design decision from design.md>
- <risk point from design.md>

## Threshold
80
```

---

## Changes Required

### 1. `commands/opsx/apply.md`

- Add dispatch cases for `N.0 CONTRACT`, `N.E EVAL`, `N.X FIX` prefixes
- Remove `N.Z requesting-code-review` dispatch (evaluator subsumes it)
- Add retry loop logic (attempt counter, plateau detection, escalate path)
- Document new file outputs (contracts/, eval-log.md)

### 2. `commands/opsx/propose.md`

- For each task group, generate `### Contract` block by extracting:
  - Spec: SHA statements from `specs/<cap>/spec.md` relevant to the group
  - Runtime: test commands from `openspec/config.yaml`, scoped to group's test files
  - Code: design decisions and risks from `design.md` relevant to the group
  - Threshold: 80 (70 if group has VISUAL DIFF tasks)
- Replace `N.Z Run superpowers:requesting-code-review` with `N.0 CONTRACT` + `N.E EVAL`

### 3. `schemas/superpowers-driven/templates/tasks.md`

- Add `### Contract` template block before each group's task list
- Replace `N.Z` placeholder with `N.0 CONTRACT` and `N.E EVAL` placeholders

### 4. `commands/opsx/archive.md`

- In Cleanup 3 (CLAUDE.md Pitfalls): read `eval-log.md` before relying on memory
- Groups with `attempt > 1` are automatic pitfall candidates — surface them for CLAUDE.md
- No other archive changes needed (contracts/ and eval-log.md archive automatically with other artifacts)

---

## What Does NOT Change

- `commands/opsx/explore.md` — no changes
- `superpowers:test-driven-development` invocation at session start — unchanged
- `superpowers:verification-before-completion` as final task — unchanged
- RED/GREEN/MOCK/VISUAL DIFF task dispatch — unchanged
- `openspec/config.yaml` structure — unchanged

---

## Cost Model

- Evaluator model: **haiku** (cost-effective for structured evaluation tasks)
- Worst case per group: 3 evaluator runs × haiku cost
- Evaluator context is small by design: contract file + spec + design + diff only (no apply agent conversation)
- `requesting-code-review` skill is invoked inside evaluator, not in addition to it

---

## Traceability

`eval-log.md` provides a complete audit trail:
- Which groups needed retries
- Why they failed (findings per attempt)
- What FIX tasks were generated
- Final score that passed

Archive phase uses this to auto-surface CLAUDE.md pitfall candidates (groups with attempt > 1).
