# Your CI Pipeline Is the Wrong Tool for AI Coding Agents

> 状态：草稿（待与 Signadot 确认后发布）
> 创建：2026-07-21

---

CI pipelines were designed for humans. A human developer submits a PR, waits 20 minutes for CI, reads the results, iterates. The feedback loop is slow — but human context doesn't expire in 20 minutes, so the timing works.

Then I read Signadot's piece on [CI for coding agents](https://thenewstack.io/ci-for-coding-agents/). It names something I'd been working around for months without clearly naming it myself. I reached out to Ani, their CTO. We talked through how their sandbox model maps onto the achieve gate in OpenSpec. Three weeks later I had a working integration.

A coding agent doesn't have that luxury. It submits a change in seconds. If it waits 20 minutes for CI feedback, it doesn't wait — it moves on. The feedback arrives in a cold context. The correction has to fight its way back in.

CI isn't wrong. It was right for what it was designed for. It's just the wrong tool for this job.

<!--truncate-->

---

## What Agents Actually Need

A coding agent needs integration feedback at exactly one point in its loop — after implementation, before it moves on.

For self-contained changes — one service, one component — this is solvable. Unit tests run in milliseconds. A fresh-context evaluator subagent scores the implementation against a spec contract in seconds. That's Layer 1.

Layer 2 is harder. Multi-component changes need integration tests. Integration tests need a deployed environment. Until recently, that meant CI.

Here's the problem. If an agent is writing a pickup confirmation endpoint that talks to a Redis store and pushes notifications through a frontend polling path, the right question isn't "do the unit tests pass?" It's "does the actual service interaction work?" You can't answer that without real services. And you can't run real services without a deployment.

So either the agent waits for CI — wrong timing — or you bring the integration environment to the agent.

---

## Sandbox as the Architectural Answer

Signadot builds exactly this. A sandbox gives you a per-branch fork of your cluster at dev time. You specify which services run your local image; the rest of the cluster stays as the baseline. A routing key directs traffic from a test client into the forked services rather than the live ones.

Two skills handle the AI agent surface. `signadot-plan` defines what the sandbox tests and what assertions constitute passing. `signadot-validate` runs the plan against a live sandbox and returns structured pass/fail.

Both map cleanly onto spec-driven development. `signadot-plan` is a machine-readable test specification. `signadot-validate` is a runtime check against a contract. The agent reads the plan, knows what passing means, and gets binary feedback inside its own loop — not 20 minutes later in CI.

---

## The Integration

I integrated Signadot into [opsx-superpowers](https://github.com/austinxyz/opsx-superpowers/blob/signadot/docs/workflow-signadot.md), my OpenSpec + Superpowers stack. The integration lives in the `achieve` phase, where evaluator subagents score completed implementation against group contracts.

For groups that span multiple components, the evaluator now runs `signadot-plan` to define sandbox assertions, then calls `signadot-validate` against a live environment. For single-component groups, nothing changes. The sandbox is conditional — it only runs when multi-component behavior is what's actually being verified.

---

## The Proof: Pickup Confirmation

I tested the integration using [hotrod](https://github.com/austinxyz/hotrod), a ride-sharing microservices demo. The feature: the driver service (`:8082`) gets `POST /dispatches/{requestID}/arrived` when the driver arrives at pickup. Unknown IDs return 404. Repeat calls return 200 without changing state. A successful arrival pushes "Driver X arrived at pickup" through the existing frontend notification path — no frontend changes needed.

The feature split into two groups. Group 1 was the Redis storage layer — dispatch record schema, CRUD operations, TTL behavior. Group 2 was the arrival handler and notification flow — HTTP handler, state transition, idempotency, frontend notification.

**Group 1** ran without a sandbox. Single component. The evaluator spawned with fresh context and scored against the group contract: spec (40%), runtime (40%), code (20%). Unit tests ran with miniredis. All 4 dispatch store tests passed. Score: 98/100. No sandbox needed.

**Group 2** needed one. The arrival endpoint talks through to the frontend notification path. Unit mocks can't give you a meaningful signal here — the integration is the point.

`signadot-plan` defined 5 assertions for Group 2:

- `dispatch-accepted` — POST to `/dispatch` returns a valid requestID
- `arrival-returns-200-after-dispatch` — POST to `/dispatches/{id}/arrived` on a known dispatch returns 200
- `repeat-arrival-idempotent-200` — second POST returns 200
- `exactly-one-arrival-notification-visible` — exactly one notification in frontend `/notifications`
- `unknown-dispatch-404` — POST to `/dispatches/unknown/arrived` returns 404

`signadot-validate` ran against sandbox `pickup-confirmation-dev` on cluster `austin-staging-1`. All 5 passed.

Evidence from the eval log:

> Fork driver (image `hotrod-pickup:dev`) in sandbox `pickup-confirmation-dev`; dispatch via baseline frontend produced `dispatch:424242`; arrival POST transitioned it (200 from first post-delay attempt); repeat POST 200 with exactly one `req-424242-arrived` notification ("Driver T740352C arrived at pickup") in frontend `/notifications`; unknown id 404.

Group 2 score: 99/100.

Full eval log: [hotrod/openspec/changes/archive/2026-07-18-pickup-confirmation/eval-log.md](https://github.com/austinxyz/hotrod/blob/opsx-setup/openspec/changes/archive/2026-07-18-pickup-confirmation/eval-log.md).

---

## What the Score Means Now

Before: runtime (40% of total) was dominated by unit tests. A 100% runtime score on a multi-component change meant "unit tests pass."

After: for a Group 2 change, 100% runtime means Signadot validated every assertion against live services.

Same number. Different claim. That's the change that matters.

---

## What This Changes for SDD

Spec-driven development gets most of its leverage from a machine-readable definition of "done." OpenSpec handles the contract layer — what the implementation should do, organized by group, with verifiable assertions. The evaluator harness handles scoring.

Before this integration, "runtime" for multi-component changes was an approximation. After, it's a real check against live services.

The current toolchain:

```
spec → implementation (Haiku) → evaluation (fresh-context subagent + signadot-validate) → achieve or rework
```

No human in the loop until achieve approval.

Planning quality matters more now, not less. A misspecified `signadot-plan` wastes validation time and generates false negatives. I ran the planning pass under Claude Fable — the output was tight enough that Haiku implementation agents never needed to return to planning. Zero rework on this feature.

---

## The Gap CI Wasn't Filling

CI was filling a real gap — integration verification before merge. Right gate for human-driven development, where feedback timing matches the developer's workflow.

AI coding agents need that same check at a different point. During implementation. Before the task completes. Not earlier in the CI pipeline — earlier in the agent's own loop.

Sandboxes at dev time move integration testing from "after merge, before main" to "during implementation, before achieve." That's where a coding agent can actually use the feedback — while context is warm, while the implementation is still the current task.

5 integration assertions. All passing. Scored into the evaluator output before achieve. No CI wait. No cold context.

That's the gap CI wasn't designed to fill. Now something else does.

---

## Series

This post is part of the OpenSpec + Superpowers series:

1. [Stacking OpenSpec and Superpowers](/blog/openspec-superpowers-combined) — combined SDD workflow, shipped a real refactor in 3 hours with 86 new tests
2. [Three Weeks Later](/blog/openspec-superpowers-three-weeks-later) — five friction points from running the stack across multiple projects
3. [Then I Added a Harness](/blog/openspec-superpowers-harness) — independent evaluator subagent per group, fresh context, structured scoring
4. [Then We Added Engineers](/blog/openspec-harness-team-workflow) — branch model, parallel conflict detection, four-layer achieve gate
5. **This post** — sandbox integration at dev time

---

## References

1. Signadot — [CI for Coding Agents](https://thenewstack.io/ci-for-coding-agents/) — the piece that started this
2. [opsx-superpowers (signadot branch)](https://github.com/austinxyz/opsx-superpowers/blob/signadot/docs/workflow-signadot.md) — workflow documentation for the Signadot integration
3. [hotrod](https://github.com/austinxyz/hotrod) — the test vehicle (Jaeger demo app, ride-sharing simulation)
4. [Pickup confirmation eval log](https://github.com/austinxyz/hotrod/blob/opsx-setup/openspec/changes/archive/2026-07-18-pickup-confirmation/eval-log.md) — full evaluation output for the feature built in this post
