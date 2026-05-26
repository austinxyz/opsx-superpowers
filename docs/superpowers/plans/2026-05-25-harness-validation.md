# Harness Validation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Wire a per-group evaluator subagent into the apply flow — contract-before-work, fresh-context scoring (Spec + Runtime + Code), score-gated retry loop, eval-log.md traceability, and archive-phase pitfall surfacing from retry history.

**Architecture:** Four markdown command files modified; no new code files. `tasks.md` schema template grows a `### Contract` block and `N.0 CONTRACT` + `N.E EVAL` tasks per group. `propose.md` fills those blocks from spec/design/config at artifact-generation time. `apply.md` dispatches CONTRACT/EVAL/FIX and manages the retry loop. `archive.md` reads `eval-log.md` to auto-surface pitfall candidates.

**Tech Stack:** Markdown, openspec CLI, superpowers skills (requesting-code-review, subagent dispatch via Agent tool)

**Spec:** `docs/superpowers/specs/2026-05-25-harness-validation-design.md`

---

### Task 1: Update tasks.md schema template

**Files:**
- Modify: `schemas/superpowers-driven/templates/tasks.md`

The template is what `openspec instructions tasks` injects as the starting skeleton. Every future change will inherit this structure.

- [ ] **Step 1: Open the current template and understand the structure**

Read `schemas/superpowers-driven/templates/tasks.md`. Note: group 1 = setup/scaffold, group 2 = feature with optional UI sandwich, group 3 = verification. The `N.Z` slot is what gets replaced.

- [ ] **Step 2: Replace the template with the new harness-aware version**

Write the complete new content to `schemas/superpowers-driven/templates/tasks.md`:

```markdown
## 1. <!-- First task group: setup or scaffold -->

### Contract
- **Spec**: <!-- SHALL statements from specs/<cap>/spec.md satisfied by this group — copy verbatim -->
- **Runtime**: `<!-- test command from openspec/config.yaml test_commands -->` → expected: <!-- what passing looks like -->
- **Code**: <!-- key design decisions / risk points from design.md relevant to this group -->
- **Threshold**: 80

- [ ] 1.0 CONTRACT — write openspec/changes/{{change}}/contracts/group-1.md with the ### Contract block above; confirm content is complete
- [ ] 1.1 RED — <!-- failing test for first behavior -->
- [ ] 1.2 GREEN — <!-- minimal implementation to pass 1.1 -->
- [ ] 1.3 RED — <!-- next failing test -->
- [ ] 1.4 GREEN — <!-- minimal impl -->
- [ ] 1.E EVAL — spawn evaluator subagent (haiku); reads contracts/group-1.md + spec + design + group diff; invokes superpowers:requesting-code-review (CRITICAL/HIGH = BLOCK); scores Spec/Runtime/Code; total ≥ 80 → PASS; < 80 → append FIX tasks + retry (max 3 attempts, plateau < 5pt = escalate)

## 2. <!-- Next task group: feature work -->

### Contract
- **Spec**: <!-- SHALL statements covered by this group -->
- **Runtime**: `<!-- test command -->` → expected: <!-- passing state -->
- **Code**: <!-- design decisions / risks for this group -->
- **Threshold**: <!-- 80; use 70 if this group contains VISUAL DIFF tasks -->

<!-- For frontend tasks that touch a VIEW / MODAL / named LAYOUT (>50 lines):
     sandwich the GREEN with MOCK + VISUAL DIFF tasks. Example: -->

- [ ] 2.0 CONTRACT — write openspec/changes/{{change}}/contracts/group-2.md with the ### Contract block above
- [ ] 2.1 MOCK — open docs/superpowers/specs/mocks/{{date}}-{{change}}-mocks.html#<anchor>; note design system tokens (see project.design_system in openspec/config.yaml) and verbatim text strings
- [ ] 2.2 RED — <!-- vitest case asserting wrapper.classes() includes the tokens -->
- [ ] 2.3 GREEN — <!-- implement the view -->
- [ ] 2.4 VISUAL DIFF — bring up dev stack (use project.dev_stack_command from openspec/config.yaml); navigate to the route; eyeball rendered UI against the mock; fix any token/color/text drift
- [ ] 2.E EVAL — spawn evaluator subagent (haiku); reads contracts/group-2.md + spec + design + group diff; invokes superpowers:requesting-code-review (CRITICAL/HIGH = BLOCK); scores Spec/Runtime/Code; total ≥ threshold → PASS; < threshold → append FIX tasks + retry

## 3. <!-- Verification + ship -->

- [ ] 3.1 Run backend test suite — ensure no regressions (use project.test_commands from openspec/config.yaml)
- [ ] 3.2 Run frontend test suite — ensure no regressions (use project.test_commands from openspec/config.yaml)
- [ ] 3.3 Run e2e suite if applicable (use project.e2e_command from openspec/config.yaml)
- [ ] 3.4 Run superpowers:verification-before-completion (run project.test_commands from openspec/config.yaml; grep -r console.log on frontend src if applicable; run project.custom_verification_checks from openspec/config.yaml)
```

- [ ] **Step 3: Verify the template is well-formed**

Read the file back. Confirm:
- Both group 1 and group 2 have a `### Contract` block above their task list
- `N.Z Run superpowers:requesting-code-review` is gone from both groups
- `N.0 CONTRACT` and `N.E EVAL` are present in both groups
- Group 3 is unchanged (no CONTRACT/EVAL — verification group has no per-group harness)
- `{{change}}` and `{{date}}` placeholders are preserved (openspec substitutes these)

- [ ] **Step 4: Commit**

```bash
git add schemas/superpowers-driven/templates/tasks.md
git commit -m "feat: update tasks.md template with CONTRACT/EVAL harness structure"
```

---

### Task 2: Update propose.md — contract generation

**Files:**
- Modify: `commands/opsx/propose.md`

`propose.md` generates `tasks.md` via `openspec instructions tasks`. After generation, it must fill in each group's `### Contract` block with real content extracted from spec/design/config. The N.0 CONTRACT task in apply just copies this to a file — the content is already decided at propose time.

- [ ] **Step 1: Locate the insertion point in propose.md**

Read `commands/opsx/propose.md`. Find step 3 ("Generate artifacts in dependency order"). The tasks artifact is last in the order: `proposal → specs → design → mocks → tasks`. The new contract-filling logic goes immediately after tasks generation, as step 3a.

- [ ] **Step 2: Add step 3a after the artifact generation loop**

In `commands/opsx/propose.md`, after the sentence "Order: `proposal` → `specs` → `design` → `mocks` → `tasks`." and before "### 4. After proposal generation: branch on HAS_UI_SURFACE", insert the following new section:

```markdown
### 3a. Fill in Contract blocks in tasks.md

After the `tasks` artifact is generated, the `### Contract` blocks contain placeholder comments. Fill them in now — the N.0 CONTRACT task in apply will copy this content to a file; the content decisions happen here.

For each `## N` group in `openspec/changes/<topic>/tasks.md`:

**Spec field:** Read `openspec/changes/<topic>/specs/<cap>/spec.md`. Identify which SHALL statements this group's tasks implement (by reading the task descriptions). Copy those SHALL statements verbatim into the Contract's Spec field. If multiple capabilities are touched, include statements from each.

**Runtime field:** Read `openspec/config.yaml` → `project.test_commands`. Choose the command most relevant to this group's tests (e.g., for a backend group: `pytest tests/<module>/`; for a frontend group: `vitest run src/<module>/`). Scope the path to the files this group touches if possible. Set expected to a plain-language description of what passing looks like (e.g., "all 4 tests pass, no import errors").

**Code field:** Read `openspec/changes/<topic>/design.md`. Extract the design decisions and risk points that apply to this group. 1–3 bullet points. Examples: "must use repository pattern, no direct DB calls in route handler", "token must be validated before any capability check".

**Threshold field:**
- Default: `80`
- If the group contains a `VISUAL DIFF` task: `70` (visual judgment has inherent subjectivity)

After filling all groups, also pre-create the `contracts/` directory:

```bash
mkdir -p openspec/changes/<topic>/contracts
```

Leave the contract files empty for now — N.0 CONTRACT in apply writes them. The directory must exist before apply starts.

Also pre-create `eval-log.md` with a header:

```bash
cat > openspec/changes/<topic>/eval-log.md << 'EOF'
# Eval Log — <topic>

<!-- Appended by evaluator subagent after each N.E EVAL run -->
EOF
```
```

- [ ] **Step 3: Update step 6 commit to include the new files**

In step 6 of `propose.md`, the `git add` line currently reads:
```bash
git add openspec/changes/<topic>/
```
This already covers contracts/ and eval-log.md since they're under `openspec/changes/<topic>/`. No change needed here — just verify this is still accurate.

- [ ] **Step 4: Update the guardrails to mention contract filling**

At the bottom of `propose.md`, in the **Guardrails** section, add after the last bullet:

```markdown
- ALWAYS fill in `### Contract` blocks in tasks.md before committing. Placeholder comments in Contract blocks are plan failures — the evaluator cannot score against empty criteria.
```

- [ ] **Step 5: Verify the changes read correctly**

Read `commands/opsx/propose.md` from line 58 onward. Confirm:
- Step 3a appears between step 3 and step 4
- Step 3a explains how to extract Spec (from spec.md), Runtime (from config.yaml), Code (from design.md)
- Threshold logic (80 default, 70 for VISUAL DIFF groups) is present
- `mkdir -p contracts/` and `eval-log.md` pre-creation are present
- New guardrail bullet is at the bottom

- [ ] **Step 6: Commit**

```bash
git add commands/opsx/propose.md
git commit -m "feat: propose.md generates contract blocks and pre-creates eval artifacts"
```

---

### Task 3: Update apply.md — harness dispatch and retry loop

**Files:**
- Modify: `commands/opsx/apply.md`

This is the main change. Three additions: (1) new dispatch cases, (2) retry loop logic, (3) updated guardrails. One removal: N.Z requesting-code-review dispatch.

- [ ] **Step 1: Update the Setup section to document new file outputs**

In `commands/opsx/apply.md`, the **Setup** section currently lists `project.dev_stack_command`, `project.test_commands`, `project.e2e_command`, `project.custom_verification_checks`, `project.design_system`. After reading config, add a note about the new artifacts:

After the `project.design_system` bullet, add:
```markdown
Also note these new artifact paths for this change:
- `openspec/changes/<name>/contracts/` — contract files written by N.0 CONTRACT tasks
- `openspec/changes/<name>/eval-log.md` — evaluator score history (pre-created by propose)
```

- [ ] **Step 2: Replace the N.Z dispatch case with CONTRACT, EVAL, and FIX cases**

In the **Walk task groups** section, find this block:

```markdown
- **`- [ ] N.Z Run superpowers:requesting-code-review on the diff for group N — ...`** → invoke `superpowers:requesting-code-review` via the **Skill** tool. Pass the group's diff as input. Address CRITICAL/HIGH findings inline before moving on; MEDIUM/LOW go to a follow-up note in the change directory.
```

Replace it with:

```markdown
- **`- [ ] N.0 CONTRACT — ...`** → read the `### Contract` block above group N in `tasks.md`. Write its content verbatim to `openspec/changes/<name>/contracts/group-N.md`. Confirm the file is written. Mark the checkbox.

- **`- [ ] N.E EVAL — ...`** → spawn evaluator subagent. See **Evaluator Subagent** and **Retry Loop** sections below. Do NOT mark the checkbox until the evaluator returns PASS. On BLOCK, pause immediately. On PASS, mark the checkbox and continue.

- **`- [ ] N.X FIX — ...`** → execute like a GREEN task: write the minimal code change described in the task. Run the relevant test to confirm the fix takes effect. Mark the checkbox. The next task will be another N.E EVAL — the retry loop re-fires automatically.
```

- [ ] **Step 3: Add the Evaluator Subagent section**

After the dispatch table (before the "Final group's verification task" bullet), add:

```markdown
### Evaluator Subagent

Spawn via the **Agent** tool with `model: haiku`. Pass ONLY these files as context — do not pass the full apply-session conversation:

- `openspec/changes/<name>/contracts/group-N.md`
- `openspec/changes/<name>/specs/<cap>/spec.md` (all capability specs for this change)
- `openspec/changes/<name>/design.md`
- The git diff for files modified in group N: `git diff HEAD~<n>..HEAD -- <files changed in group N>`

Evaluator prompt (pass this verbatim):

> You are an external evaluator with a skeptical lens. You have no knowledge of the implementation decisions made during this session.
>
> 1. Invoke `superpowers:requesting-code-review` on the provided diff. If you find CRITICAL or HIGH severity issues, return immediately: `STATUS: BLOCK` with the findings. Do not score.
> 2. Run the Runtime test command from the contract. Record pass/fail and output.
> 3. Compare the diff against each SHALL statement in the contract's Spec section. Score 0–100.
> 4. Score Runtime 0–100 (100 = all tests pass, 0 = test command fails to run).
> 5. Score Code 0–100 based on requesting-code-review findings (no CRITICAL/HIGH assumed at this point).
> 6. Compute total = (Spec × 0.4) + (Runtime × 0.4) + (Code × 0.2).
> 7. Read the Threshold from the contract.
> 8. Append to `openspec/changes/<name>/eval-log.md`:
>    ```yaml
>    - group: N
>      attempt: <attempt_number>
>      scores: {spec: X, runtime: Y, code: Z}
>      total: T
>      status: PASS | RETRY
>      findings:
>        - "<dimension>: <specific finding>"
>      fix_tasks:
>        - "N.F1 FIX — <specific actionable fix>"
>    ```
> 9. If total ≥ threshold: return `STATUS: PASS`.
>    If total < threshold: append fix_tasks to group N in `tasks.md` as `- [ ] N.F1 FIX — ...`, then return `STATUS: RETRY` with the fix_tasks list.
```

- [ ] **Step 4: Add the Retry Loop section**

After the Evaluator Subagent section, add:

```markdown
### Retry Loop

The apply agent manages attempt counting. Per group:

```
attempt = 1
score_history = []

loop:
  spawn evaluator → result

  if result == BLOCK:
    pause: "Group N BLOCKED — CRITICAL/HIGH code issue. Fix and resume from N.E EVAL."
    stop loop

  if result == PASS:
    mark N.E EVAL checkbox [x]
    advance to next group
    stop loop

  # RETRY path
  score_history.append(result.total)
  attempt += 1

  escalate if:
    attempt > 3
    OR (len(score_history) >= 2 AND score_history[-1] - score_history[-2] < 5)

  if escalate:
    pause: "Group N failed after {attempt-1} attempts. Score history: {score_history}. Last findings: {findings}. Options:
      1. Fix manually, then continue (type 'resume')
      2. Skip group N (type 'skip')
      3. Abort apply (type 'abort')"
    stop loop

  # otherwise: FIX tasks already appended to tasks.md by evaluator
  # execute them, then loop back to spawn evaluator
```
```

- [ ] **Step 5: Update the Guardrails section**

Find and remove:
```markdown
- DO invoke `superpowers:requesting-code-review` at every group's `N.Z` checkpoint. Don't batch all reviews to the end.
```

Add in its place:
```markdown
- DO spawn an evaluator subagent at every group's `N.E EVAL` checkpoint. Never merge evaluator into the apply agent's context — the fresh context is the point.
- DO write `contracts/group-N.md` at the `N.0 CONTRACT` step before any RED/GREEN tasks in the group. Never skip this.
- DO manage the retry loop per group. Three attempts max; plateau < 5pt triggers escalation.
- NEVER invoke `superpowers:requesting-code-review` directly during apply — the evaluator subagent handles it internally.
```

- [ ] **Step 6: Verify the full apply.md reads correctly**

Read `commands/opsx/apply.md` in full. Confirm:
- Setup section mentions contracts/ and eval-log.md
- Dispatch table has N.0 CONTRACT, N.E EVAL, N.X FIX — no N.Z
- Evaluator Subagent section exists with the verbatim evaluator prompt
- Retry Loop section exists with the pseudocode
- Guardrails: no reference to requesting-code-review as a direct call; evaluator guardrails present

- [ ] **Step 7: Commit**

```bash
git add commands/opsx/apply.md
git commit -m "feat: apply.md harness — CONTRACT/EVAL/FIX dispatch + retry loop"
```

---

### Task 4: Update archive.md — eval-log pitfall surfacing

**Files:**
- Modify: `commands/opsx/archive.md`

Small targeted change: Cleanup step 3 (CLAUDE.md pitfalls) currently relies on the agent's memory of gotchas. Add an explicit step to read `eval-log.md` first — groups with attempt > 1 are structured signals of non-obvious complexity.

- [ ] **Step 1: Locate Cleanup step 3 in archive.md**

Read `commands/opsx/archive.md`. Find step 5 ("Cleanup step 3 — update CLAUDE.md pitfalls"). It currently reads:

> Read the dev log entry at `docs/log/<date>.md` (if it exists) and the change diff via `git log ...`

- [ ] **Step 2: Add eval-log.md read before the dev log read**

Replace the opening sentence of step 5 with:

```markdown
### 5. Cleanup step 3 — update `CLAUDE.md` pitfalls

First, read `openspec/changes/archive/<date>-<name>/eval-log.md` (it was archived with the other artifacts). Find any entries where `attempt > 1` — these groups needed multiple evaluator passes, which is a structural signal that something non-obvious happened. For each such group, read the `findings` from the failed attempts: if they describe a foot-gun worth documenting (timing-sensitive behavior, env-var ordering, schema migration edge case, boundary condition that the spec didn't make explicit), that's a CLAUDE.md pitfall candidate.

Then read the dev log entry at `docs/log/<date>.md` (if it exists) and the change diff via `git log --oneline <change-base-sha>..<change-head-sha>` plus `git diff <change-base-sha>..<change-head-sha>` (using the SHAs captured in step 2). If any non-obvious gotcha emerged (timing-sensitive bootstrap, env-var ordering, schema migration foot-gun, file-handling edge case), append a 2-3 line entry to the relevant section of `CLAUDE.md`'s Pitfalls.

If no new pitfall surfaced from either eval-log or dev log, skip this step. Don't fabricate pitfalls.
```

- [ ] **Step 3: Verify the change**

Read `commands/opsx/archive.md` step 5. Confirm:
- eval-log.md read comes first, before dev log
- "attempt > 1" groups are described as pitfall candidates
- "Don't fabricate pitfalls" guardrail is still present
- The rest of the step (dev log + diff read) is unchanged

- [ ] **Step 4: Commit**

```bash
git add commands/opsx/archive.md
git commit -m "feat: archive.md surfaces eval-log retry groups as CLAUDE.md pitfall candidates"
```

---

### Task 5: Trace-through verification

No test framework exists for command files. Verification is a manual trace through the new flow to catch logical gaps before a real apply run.

- [ ] **Step 1: Trace propose → tasks.md output**

Read the updated `commands/opsx/propose.md` and `schemas/superpowers-driven/templates/tasks.md` together. Mentally run through a propose for a 2-group backend change (no UI). Confirm:

- Group 1 `### Contract` gets filled with SHALL statements, runtime command, code notes
- Group 1 has `1.0 CONTRACT`, `1.1 RED`, `1.2 GREEN`, `1.E EVAL` tasks
- Group 2 same structure
- `contracts/` directory pre-created
- `eval-log.md` pre-created with header
- No `N.Z requesting-code-review` appears anywhere in tasks.md

- [ ] **Step 2: Trace apply PASS path**

Read `commands/opsx/apply.md`. Trace the happy path for group 1:
- `1.0 CONTRACT` → write contracts/group-1.md ✓
- `1.1 RED` → failing test ✓
- `1.2 GREEN` → implementation ✓
- `1.E EVAL` → evaluator spawned, scores 85 total, returns PASS → checkbox marked ✓
- Moves to group 2

Confirm no step tries to invoke `superpowers:requesting-code-review` directly.

- [ ] **Step 3: Trace apply RETRY → PASS path**

Trace attempt 1 fails (score 68), attempt 2 passes (score 82):
- `1.E EVAL` attempt 1 → RETRY, evaluator appends `1.F1 FIX — <fix>` to tasks.md
- Apply agent executes `1.F1 FIX`
- `1.E EVAL` attempt 2 → PASS (82 ≥ 80) → checkbox marked

Confirm attempt counter increments correctly, score_history = [68, 82], no escalation triggered (attempt 2 < 3, delta 14 ≥ 5).

- [ ] **Step 4: Trace apply BLOCK path**

Trace evaluator finds CRITICAL code issue:
- `1.E EVAL` → evaluator invokes requesting-code-review → CRITICAL found → returns STATUS: BLOCK
- Apply pauses, reports CRITICAL finding, waits for human
- No retry loop entry (BLOCK exits immediately)

Confirm: apply agent does NOT continue to group 2 on BLOCK.

- [ ] **Step 5: Trace apply escalation path**

Trace 3 attempts all score below 80, each delta < 5:
- attempt 1 → 65, attempt 2 → 68, attempt 3 → 70
- After attempt 3: attempt > 3 is false (attempt == 3), but check: after attempt 3 completes, loop checks `attempt > 3` → 3 is not > 3.

Wait — the pseudocode says `attempt += 1` THEN checks escalation. So:
- Start: attempt=1, spawn evaluator → RETRY → attempt becomes 2, score_history=[65]
- Loop: attempt=2, spawn evaluator → RETRY → attempt becomes 3, score_history=[65,68], delta=3 < 5 → ESCALATE

Escalate fires after attempt 2 (when delta < 5 is detected). This is correct — no need to wait for a third failure if the improvement plateau is already clear.

If delta was ≥ 5 on attempt 2: attempt becomes 3, score_history=[65,72], no plateau → loop → attempt becomes 4, check `attempt > 3` → TRUE → ESCALATE.

Confirm this logic is correctly represented in the Retry Loop section of apply.md. If not, adjust the pseudocode.

- [ ] **Step 6: Trace archive eval-log read**

Read `commands/opsx/archive.md` step 5. Trace: eval-log.md has group 1 attempt=1 (PASS) and group 2 attempt=2 (PASS). Group 2 attempt 1 finding was "runtime: token not persisted across requests." This IS a pitfall candidate. Archive agent surfaces it for CLAUDE.md consideration.

Confirm the updated step 5 would catch this case.

- [ ] **Step 7: Commit verification notes**

No file changes needed if trace passes cleanly. If any logical gap was found and fixed in steps 1–6, commit those fixes:

```bash
git add commands/opsx/apply.md commands/opsx/propose.md commands/opsx/archive.md schemas/superpowers-driven/templates/tasks.md
git commit -m "fix: correct retry loop logic from trace-through verification"
```

If no fixes needed, skip this commit.

---

## Self-Review

### Spec coverage

Checking `docs/superpowers/specs/2026-05-25-harness-validation-design.md` against plan tasks:

| Spec section | Covered by |
|---|---|
| Contract block in tasks.md | Task 1 |
| Contract filling at propose time | Task 2 |
| contracts/ dir + eval-log.md pre-creation | Task 2 |
| N.0 CONTRACT dispatch in apply | Task 3 Step 2 |
| N.E EVAL dispatch + evaluator prompt | Task 3 Steps 2–3 |
| N.X FIX dispatch | Task 3 Step 2 |
| Retry loop (3 attempts, plateau < 5pt) | Task 3 Step 4 |
| BLOCK path (CRITICAL/HIGH immediate stop) | Task 3 Step 3 (evaluator prompt) |
| Score dimensions (Spec 40%, Runtime 40%, Code 20%) | Task 3 Step 3 (evaluator prompt) |
| Threshold 70 for UI groups | Task 1 (template comment) + Task 2 (filling logic) |
| requesting-code-review invoked inside evaluator | Task 3 Step 3 (evaluator prompt) |
| eval-log.md format | Task 3 Step 3 (evaluator prompt) |
| eval-log.md → CLAUDE.md pitfall surfacing in archive | Task 4 |
| Remove N.Z requesting-code-review from apply guardrails | Task 3 Step 5 |

**No gaps found.**

### Placeholder scan

- No TBD, TODO, or "similar to Task N" in any step
- All code blocks show exact content to write, not descriptions
- File paths are exact throughout

### Type consistency

- `contracts/group-N.md` path used consistently across Tasks 1, 2, 3
- `eval-log.md` path used consistently across Tasks 2, 3, 4
- `N.E EVAL` prefix used consistently in template (Task 1) and dispatch (Task 3)
- `N.0 CONTRACT` used consistently in template (Task 1) and dispatch (Task 3)
- `N.X FIX` dispatch in Task 3 matches `N.F1 FIX` format in evaluator prompt — note: dispatch matches on `FIX` keyword, ordinal is variable. This is explicit in the dispatch case description. Consistent.
