# Multi-Engineer OpenSpec Workflow — Speaker Notes

## 01_cover

This deck introduces the OpenSpec workflow for teams of 2–6 engineers. The core premise is simple: in an AI-accelerated environment, spec quality determines delivery quality — not coding speed. The four phases (Explore → Propose → Apply → Archive) form a hard sequential pipeline with no skipping allowed.

## 02_the_shift

The engineer's job has fundamentally changed. Before: the senior engineer was the fastest coder. After: the senior engineer is the sharpest spec author. Claude handles implementation; you handle judgment. The bottleneck has shifted from hands-on-keyboard to clarity-of-thought.

## 03_four_phases

Each phase has hard entry gates and produces a specific output artifact that the next phase consumes. You cannot start Apply without reviewed requirements and a locked spec. The ⛔ GATE indicators are not suggestions — they are enforced by the harness and tooling.

## 04_two_track

Small features (≤2 days, single module) go straight to a single PR. Large features (≥3 days, architecture, cross-module) require a spec PR first — this locks tasks.md and design.md before any code is written. The two-track model prevents scope creep and keeps spec review fast.

## 05_achieve_gate

Completing apply is not the same as achieving the feature. Four gates must clear in order: local eval passes, CI is green, a human verifies spec fidelity, and archive captures the learning. PR merge to main is the moment of Achieve — not when code compiles.

## 06_parallel_dev

The key to safe parallel development is catching interface conflicts at propose time — before anyone writes code. Read active branches and proposals first. If two features share an interface, declare depends_on and sequence them. If not, parallelize immediately. Merge conflicts are a design smell, not a git problem.

## 07_review_model

Humans do spec review (15–30 min, high bar, intent and decomposition). CI and the evaluator subagent do code review. When you open a PR, ask one question: "Did Claude build the right thing?" — not "Is this code well-written?" The harness already answered the code question.

## 08_shared_files

At 4–6 engineers, shared files become write bottlenecks. CLAUDE.md and README.md turn into merge conflict magnets when multiple engineers archive in the same sprint. Solution: each capability owns its own pitfalls.md. CLAUDE.md becomes a 2-line index entry per capability. Global pitfalls batch into a dedicated maintenance PR each sprint.

## 09_getting_started

Three actions, start today. Run /opsx:explore before writing any code — even on a feature you think you understand. For large features, open the spec PR before touching implementation. During apply, let the harness drive and review spec fidelity, not line counts. Achieve is archive + CI green + PR merged.
