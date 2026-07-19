# OpenSpec + Superpowers × Signadot — Status & Feedback for Joe

**From:** Austin
**Date:** 2026-07-18
**TL;DR:** The integration works end-to-end. I ran a real feature through the full
spec-driven workflow on hotrod, with a Signadot plan authored at spec time, executed
against a real cluster before the PR existed, and its verdict gating the merge.
Found a few bugs/gaps worth your attention (below).

---

## 1. What I built

Recap of the concept (per the earlier discussion with Ani): my workflow
(**OpenSpec + Superpowers**) is a four-phase spec-driven development loop —
`explore → propose → apply → archive` — where every task group is gated by an
independent evaluator that scores Spec / Runtime / Code. The integration makes
Signadot the **Runtime** dimension: instead of an agent guessing from local runs,
a Signadot plan runs the behavior against a real cluster and the verdict feeds the gate.

What was implemented:

- **Extended the workflow skills** (in the [hotrod fork](https://github.com/austinxyz/hotrod),
  branch `opsx-setup`) with:
  - `propose` phase: authors a Signadot plan per integration-critical task group,
    with **unbound params** (Ani's model — no stubs), and binds the group's contract
    to it (`Runtime: validated by signadot plan <behavior-id>`)
  - `apply` phase: new **`N.V VALIDATE`** step between implementation (TDD GREEN)
    and evaluation — binds params, runs the plan via CLI, appends the structured
    verdict to the eval log
  - evaluator: reads the plan verdict as Runtime evidence (pass=100, fail=0,
    fail = immediate block); groups without a plan behave exactly as before
  - `archive` phase: registers validated plans into a durable per-capability
    plan library (`openspec/specs/<capability>/plans/`) — the versioned
    "what correct means" catalog
- **Used your official agent skills** (github.com/signadot/agent-skills) rather
  than writing custom wrappers — the `signadot-plan` / `signadot-validate` wrappers
  we originally planned for you to build turned out to be unnecessary.

## 2. Test case: `pickup-confirmation` on hotrod

New cross-service feature, developed entirely through the workflow:

- Driver service persists a dispatch record in Redis after computing best ETA
- New endpoint `POST /dispatches/{requestID}/arrived` on the driver service:
  404 unknown id, idempotent repeat, status transition + "Driver X arrived at pickup"
  notification
- Frontend shows the arrival through its existing notification polling — zero
  frontend changes

Validation chain the plan covers: `HTTP POST → driver → Redis → frontend polling` —
three components, not coverable by unit tests. Exactly the case from your
"CI wasn't built for coding agents" article.

## 3. Status — what passed

Environment: local kind cluster + Signadot Operator, cluster `austin-staging-1`,
org `austinxyza`, free tier.

| Checkpoint | Result |
|---|---|
| TDD groups (dispatchstore CRUD, arrival endpoint) | ✅ 12/12 unit tests, evaluator 98 & 99/100, first attempt |
| Plan authored against live `signadot plan schema` | ✅ `pickup-confirmation-arrival` (plan `lkyjcsgkqkjxn`) |
| Sandbox run: forked driver (`hotrod-pickup:dev`) against baseline frontend/redis/kafka | ✅ sandbox `pickup-confirmation-dev`, execution `dp4c62nm2suys` |
| Plan assertions | ✅ 5/5: dispatch-accepted, arrival-200, idempotent-repeat, exactly-one-notification-visible, unknown-404 |
| Verdict consumed as the gate's Runtime evidence | ✅ (evaluator read the execution output, not a local guess) |
| Plan registered into durable library at archive | ✅ `openspec/specs/dispatch-tracking/plans/` |

Everything on the happy path worked. The routing-key header injection via
`routingContext` + `request-http` worked exactly as documented.

## 4. Issues found (the useful part)

1. **Image allowlist appears broken.** With run-container/k6 images allowlisted in
   Platform → Managed Runners → Configure Actions (tried both shorthand and
   fully-qualified `docker.io/...` forms; Apply saved), `plan create` still rejects
   every image — including k6's own system image `grafana/k6:1.7.1` — with
   "not in this org's allowed-images list". Only actionbox-backed actions
   (request-http / eval / check) were usable. Org: `austinxyza`. This forced the
   workaround in #2.

2. **No sleep/wait primitive in plans.** Kafka processing is async, so the arrival
   POST needs polling. `request-http` exits 1 on transport timeout, so a
   blackhole-URL delay fails the step. I resorted to chaining
   `GET httpbin.org/delay/4` steps as ~4s spacers with a 6× retry chain — ugly but
   works. A `wait` action, or a retry/backoff policy on `request-http`, would
   remove the hack entirely. (This is my top feature request.)

3. **No Windows CLI build.** All releases are darwin/linux; source build is blocked
   by the private `signadot/libconnect` dependency. The docker image works as a
   wrapper, but `signadot auth login` fails in-container (dbus keyring); the
   `config.yaml` API-key fallback works. A windows binary — or a documented
   docker-wrapper path — would help Windows users a lot.

4. **Docs gap.** "Enabling Plan Runner Groups" is referenced in release notes but
   the page 404s. I found the dashboard toggle (Platform → Managed Runners) by
   trial and error. Also worth documenting: a `jobrunnergroup` does NOT satisfy
   "no plan runner group on cluster" — those are separate concepts and the error
   message doesn't say so.

**Open questions:**

- Is there a dry-run mode for plans (validate structure without spinning up a run)?
- One plan can cover a behavior spanning multiple task groups; my next feature
  will exercise that N:M mapping — any advice on how you think about plan↔scope
  granularity?

## 5. How someone else would use this

Prerequisites (one-time, ~30 min):

1. Kubernetes cluster (kind works fine) + Signadot Operator installed, cluster
   connected in app.signadot.com
2. **Managed Plan Runners enabled** for the cluster (Platform → Managed Runners)
3. Signadot CLI authenticated (`signadot auth login`, or config.yaml API key)
4. Claude Code with the Superpowers plugin, plus your official agent skills:
   `npx skills add signadot/agent-skills`
5. OpenSpec CLI: `npm i -g @fission-ai/openspec`

Setup per project:

1. Install the opsx scaffolding (schema + 4 slash commands) — installer script in
   [austinxyz/opsx-superpowers](https://github.com/austinxyz/opsx-superpowers)
   (branch `signadot`; sync from the validated hotrod-opsx copies is pending)
2. In `openspec/config.yaml`, set:
   ```yaml
   integrations:
     signadot:
       enabled: true
       cluster: <your-cluster-name>
   ```

Then per feature:

| Command | What happens (Signadot side) |
|---|---|
| `/opsx:explore <topic>` | discussion flags integration-critical behaviors |
| `/opsx:propose <topic>` | plan authored per critical group, unbound params, contract bound to plan id |
| `/opsx:apply <topic>` | TDD, then `N.V VALIDATE`: sandbox with the forked service, plan runs, verdict gates the evaluator |
| `/opsx:archive <topic>` | plan registered into the capability's plan library |

Working reference: the hotrod fork above — the archived change
(`openspec/changes/archive/2026-07-18-pickup-confirmation/`) shows every artifact,
including the eval log with the real verdict, and
`openspec/specs/dispatch-tracking/plans/pickup-confirmation-arrival.yaml` is a
complete plan example (params, eval-composed URLs, routingContext, checks).

## 6. Next on my side

- Second feature (`driver-status`) — exercises plan↔group N:M mapping and plan
  library reuse
- Sync the skill changes back into the opsx-superpowers repo as the installable
  source of truth

Happy to walk through any of this on a call.

— Austin
