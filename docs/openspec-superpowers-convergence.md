# The Upstream Caught Up (and Broke My Fork)

> 状态：草稿（英文主稿，系列第 6 篇）
> 创建：2026-08-31

---

```
✖ Error: Invalid schema at '...superpowers-driven\schema.yaml':
  artifacts.0.generates: generates field must be a relative path
  inside its allowed directory
```

That's what `openspec new change` printed after I upgraded OpenSpec from 1.2.0 to 1.11.0. My custom schema — the thing this whole series is built on — was suddenly invalid. Every project using it would break on upgrade.

This post covers what changed upstream in OpenSpec and Superpowers over the summer, how much of it converged with the enhancements I'd already built, what I changed in response, and what a month of running the full stack on a real open-source project looks like.

<!--truncate-->

---

## What OpenSpec Shipped (1.2 → 1.11)

Nine minor versions in three months. The ones that matter:

| Version | Change | Why it matters |
|---|---|---|
| 1.6 | **`/opsx:update`** — revise planning artifacts mid-change | My four-phase flow had no revision step. Mid-change edits were hand-edits with no coherence check |
| 1.9–1.10 | **`validate` catches unwritten `## Purpose`**, `validate --archived` | A `TBD` Purpose was the canonical dead-end my archive phase existed to fix — now the CLI polices it |
| 1.11 | **`generates:` paths must stay inside the change directory** | The breaking change above. My schema wrote requirements to `docs/` — authored before the change dir exists |
| 1.11 | `--diff` for changes, `--all` status | Less hand-rolled git plumbing in the evaluator |
| 1.5–1.7 | Stores (beta), auto-update check, more agent targets | Watch, don't adopt yet |

Also new: zero-delta changes are now rejected by `validate` unless `.openspec.yaml` sets `skip_specs: true`. Specs describe behavior; if behavior doesn't change, no spec should change — but you have to say so explicitly now.

## What Superpowers Shipped (6.0 → 6.3)

Superpowers 6.0 was the big rewrite: the two per-task reviewers (spec compliance + code quality) merged into **one Task Reviewer producing two verdicts** from a single read of the diff. Half the review cost, same gates. Handoffs moved into files — task briefs and review packages under `.superpowers/sdd/` — so the main session stops carrying every diff.

6.2 and 6.3 kept pulling the same thread:

- **Ceremony scaling** (6.3): requests get classified as *spike*, *bounded*, or *architectural*. Small tasks skip the two-document ritual entirely.
- **Plans carry a `Spec:` pointer** (6.3): implementation plans link back to the design they satisfy.
- **Review-fix loop hardening** (6.2): the implementer is resumed with a scoped re-review prompt, with a five-round circuit breaker.
- Plan-scoped workspaces, Windows Git Bash dispatch, skill compression.

## The Convergence Scorecard

Here's the part I find genuinely satisfying. When I built the opsx-superpowers harness in May, several design decisions were bets. The upstream releases are a chance to grade them:

| My enhancement (May) | Upstream now (Aug) | Verdict |
|---|---|---|
| One evaluator subagent scoring Spec/Runtime/Code per group | 6.0: one Task Reviewer, two verdicts | **Converged** — same insight, independently |
| Contract block binds tasks to SHALL statements | 6.3: plans carry a `Spec:` pointer | **Converged** |
| Retry loop: max 3 attempts, plateau detection, escalate to human | 6.2: five-round circuit breaker on review-fix | **Converged** |
| Archive fills `## Purpose` (the TBD dead-end fix) | 1.10: `validate` flags unwritten Purpose | **Converged** — and upstream's version is better placed |
| Binary "use OpenSpec or don't" table | 6.3: three-tier ceremony scaling | **They were ahead** — absorbed it |
| Signadot real-cluster verdict as Runtime evidence | — | **Still unique** |
| Durable plan library per capability (`specs/<cap>/plans/`) | — | **Still unique** |
| Pitfall sinking from eval retry signals | — | **Still unique** |

Four convergences, one absorption, three things still ahead of upstream. When two teams independently land on "one skeptical reviewer, double verdict, circuit breaker," that's evidence the shape is right — the same way convergent evolution tells you wings are a good idea.

## What I Changed

Five edits, all on the [signadot branch](https://github.com/austinxyz/opsx-superpowers/tree/signadot):

1. **Fixed the breaking change.** Requirements and mocks artifacts now generate *inside* the change directory; `/opsx:propose` copies the canonical files in from `docs/` right after `openspec new change`. Side effect: `openspec status` is finally accurate for these artifacts — the old `{{date}}` placeholder gotcha is gone.
2. **Added `/opsx:update`.** Adapted the upstream revision workflow, extended with the invariants upstream can't know about: stale Contract Spec fields (they're verbatim copies of SHALL statements — silent rot), already-written contract files, signadot plan assertions, and the rule that revising an EVAL-passed group must uncheck its gate.
3. **Ceremony scaling as explore Phase 0.** Spike → exit the workflow, just fix it. Bounded → short requirements, review still mandatory. Architectural → full flow. The classification is said out loud; the user can override.
4. **`## Purpose` moved to propose.** The spec template authors it; archive verifies instead of writes, backed by `validate --archived`.
5. **Validation gates.** `openspec validate` in archive pre-flight; `skip_specs: true` documented for zero-delta changes.

## A Month of Practice: zijing-cup

Theory is cheap. [zijing-cup](https://github.com/austinxyz/zijing-cup) is the practice: a fully open-source tennis team and lineup management tool for a Chinese university alumni tournament — Next.js 16 + FastAPI + Supabase, deployed on Vercel and Render. Built end-to-end with Claude Design + OpenSpec + Superpowers.

The numbers, one week into serious feature work (and still climbing — the first draft of this post said 6 changes; by publish week it was 18):

- **18 changes archived** — rules engine, roster import/display, lineup engine, player management, saved filters, saved lineups with 4-state revalidation, single-seat pinning, win/loss tracking, roster seat editing, scoped per-competition admin auth — with a 19th in flight
- **13 capability specs** live in `openspec/specs/`
- **46k LOC** (Python backend + TypeScript frontend), **382 test files**, 239 commits

Two moments that justify the harness:

**The evaluator caught a real bug after GREEN.** The lineup engine's search group passed unit tests, but the evaluator's first pass found that invalid lineup locks bypassed the per-line constraint checks entirely. BLOCK → fix tasks appended → attempt 2 passed at 99/100 with a validation layer (`check_locks` before search). That's the Generator/Evaluator split doing exactly what it exists for: the implementer had convinced itself; the fresh-context reviewer hadn't.

**The scores are a real distribution, not a rubber stamp.** Across 18 changes the eval gate has produced everything from a perfect 100 (`saved-lineup-order`) to an 87 with a CRITICAL spec-compliance finding (`lineup-page-defaults` — a toggle required by the design simply wasn't implemented) and a 94 carrying a HIGH finding on a missed rules-inspection path (`team-roster-editing`). A gate that only ever says 98 is decoration; one that hands you an 87 with a named missing feature is doing its job.

**Claude Design closes the fidelity gap.** Every UI change starts as mocks (7 HTML mock files so far) drawn during explore. Tasks then sandwich implementation: MOCK (record tokens and exact copy) → RED → GREEN → VISUAL DIFF (dev server up, screenshot against the mock, fix drift). One example of what VISUAL DIFF catches that tests can't: a sidebar token collision put light text on `bg-background` at 1.05:1 contrast — invisible in unit tests, obvious next to the mock. Fixed to 16.07:1 with sidebar-specific tokens.

And the archive phase keeps compounding: CLAUDE.md now carries pitfalls no spec would have predicted — Windows Application Control blocking `uv run uvicorn`, Google Sheets CSVs with headers on row 5, `h-screen overflow-hidden` shells swallowing long lists. Each one was paid for once and recorded; none has been paid for twice.

**The upgrade shows up in the repo itself.** The same project now contains changes from both toolchain generations, and the diff is visible in the artifact trees. Changes archived before the upgrade (`2026-08-29-lineup-engine`) have no `requirements.md` or `mocks.html` inside the change directory — and `openspec status` reported those artifacts as missing for their entire life. Changes from after (`2026-09-01-lineup-results-redesign`) carry both copies in-dir, and status showed all-green for the first time. The absence of `## Purpose` in the new change's spec deltas is the template working as designed, too — Purpose is authored only for *new* capabilities, and these changes modified existing ones.

## Do the SDD Critiques Land Here?

A widely-shared Chinese post ("From OpenSpec to AIDLC") recently made the case for abandoning OpenSpec in team settings, replacing it with Amazon's AIDLC workflow. Its six criticisms of vanilla OpenSpec are a useful audit checklist — worth scoring this harness against them honestly.

| Critique of vanilla OpenSpec | Status here |
|---|---|
| Commands too complex; nobody remembers explore-vs-propose order | **Mostly addressed.** Hard gates enforce the order mechanically (`propose` refuses without a REVIEWED requirements doc), and explore's Phase 0 decides *for* you whether the workflow applies at all. But there are still commands to learn — AIDLC's "resident butler" model, where one always-on workflow dispatches sub-flows, is genuinely lower-friction |
| Original intent never recorded; reviewers can't tell why a spec says what it says | **Addressed.** `requirements.md` *is* the intent artifact — Goals, Non-Goals, User Stories, Open Questions — reviewed to REVIEWED status and (since 1.11) archived inside every change. What we don't keep is the turn-by-turn conversation; AIDLC's `audit.md` records the dialog, we record the conclusions |
| Requirement granularity impossible to judge; wish-style prompts produce demos | **Addressed.** Phase 0 tiers + explore's scope check (too big → decompose into sub-changes). Evidence: 18 zijing-cup changes averaging one capability slice each, 12 of them in a single week |
| Docs drift from code; editing tasks.md never updates design.md | **Addressed by design.** `/opsx:update` exists precisely for this: any-direction reconciliation, stale Contract detection, re-opening passed gates when their group is revised. The per-group EVAL also checks code against spec continuously. (Honest caveat: zijing-cup hasn't needed `/opsx:update` yet — the mechanism is newer than the practice) |
| Nobody knows when to start the workflow (bug mid-apply? new idea mid-fix?) | **Addressed.** Phase 0 answers "just fix it or run the flow" out loud; bugs during apply have a built-in FIX-task loop; mid-change ideas hit update's refine-vs-new-change heuristic |
| No team collaboration: no approval gates, no accountability for who approved what | **Addressed in the team layer — differently.** [The team workflow](/blog/openspec-harness-team-workflow) routes human approval through git-native machinery instead of custom audit files: **PR1** is a spec-only review (requirements + proposal + design + tasks, no code) whose merge *locks* the spec before apply starts; the four-layer achieve gate's Layer 3 is a human PR gate (≥1 approval + CI green + no open comments); and every approval is signed by the reviewer's GitHub identity for free. AIDLC rebuilds this inside the workflow (`audit.md` with user/email, approve-vs-continue split) — useful if you can't lean on a git host, redundant if you can. For solo development neither is needed |

Net: five of six addressed, one mostly. The critique is aimed at bare OpenSpec, and it stops landing once a harness is wrapped around the spec layer — the team-approval point in particular was answered by the team workflow months earlier, with git PRs playing the role AIDLC assigns to its audit files.

## Takeaways

1. **Fork maintenance is real but bounded.** Nine upstream versions cost one breaking fix and an afternoon — because the maintenance boundary (schema + four command files) was drawn deliberately in May.
2. **Convergence is the best code review.** Upstream independently arriving at your design tells you more than any benchmark.
3. **Keep the parts upstream doesn't have.** Real-cluster Runtime verdicts, the plan library, and pitfall sinking are still the fork's reason to exist.
4. **Audit the critiques, don't chase them.** The popular case against SDD dissolves almost entirely against a harnessed setup — and the team-approval point it leads with was already answered by git-native PR gates in the team workflow. Score criticism against what you've built before absorbing someone else's fix.
5. **The practice repo is the proof.** 18 archived changes with eval logs anyone can read beats any workflow diagram.

---

## Series

1. [Stacking OpenSpec and Superpowers](/blog/openspec-superpowers-combined)
2. [Three Weeks Later](/blog/openspec-superpowers-three-weeks-later)
3. [Then I Added a Harness](/blog/openspec-superpowers-harness)
4. [Then We Added Engineers](/blog/openspec-harness-team-workflow)
5. [Your CI Pipeline Is the Wrong Tool for AI Coding Agents](/blog/ci-wrong-tool-for-ai-agents)
6. **This post** — upstream convergence + a month of practice

## References

1. [OpenSpec releases](https://github.com/Fission-AI/OpenSpec/releases) — v1.4.0 through v1.11.0
2. [Superpowers releases](https://github.com/obra/superpowers/releases) — v6.0.0 through v6.3.0
3. [opsx-superpowers (signadot branch)](https://github.com/austinxyz/opsx-superpowers/tree/signadot) — the fork, with sync log in docs/workflow.md
4. [zijing-cup](https://github.com/austinxyz/zijing-cup) — the open-source practice project
5. 骞先生, "从OpenSpec到AIDLC，我是如何提升团队AI代码质量的" — the SDD critique scored in this post; AIDLC source: [aidlc-skills](https://github.com/qtalen/aidlc-skills), upstream: AWS aidlc-workflows
