# OpenSpec × Signadot — Example Integration

Applying the OpenSpec + Superpowers workflow to a real Signadot project, end-to-end.

## Plan

1. Sign up for Signadot trial account → https://app.signadot.com/sign-up
2. Explore test project: https://github.com/signadot/hotrod
3. Get hotrod running with trial cluster
4. Apply `/opsx:explore → propose → apply → archive` to a new feature — as a reference example of OpenSpec + Signadot integration

## Target project

[hotrod](https://github.com/signadot/hotrod) — Jaeger demo microservices app (Go). Services: frontend, driver, route, customer.

Feature candidate: a behavior that spans at least two services, so Signadot's real-cluster cross-service validation adds genuine value over unit tests.

## Key pre-requisites

~~Before `/opsx:apply`, need answers from Joe~~ — mostly self-answered 2026-07-18:
official skills exist (github.com/signadot/agent-skills), plan schema is
runtime-discoverable (`signadot plan schema`, action DAG), verdict via
`plan x get-output`. See "Validation results" in
[`../docs/signadot-integration-spec.md`](../docs/signadot-integration-spec.md).
Still for Joe: image-allowlist bug, sleep primitive, dry-run mode, Windows CLI.

## Contacts

- Ani (CTO) — concept aligned
- Joe (Engineering) — implementation lead on Signadot side

## Status

- [x] Trial account registered
- [x] hotrod running locally
- [x] hotrod connected to Signadot cluster (austin-staging-1, kind)
- [x] Feature candidate chosen (pickup-confirmation, 2026-07-18)
- [x] OpenSpec change opened for the feature (hotrod-opsx, branch opsx-setup)
- [x] `signadot-plan` skill invoked at propose (official skill from signadot/agent-skills)
- [x] `signadot-validate` runs at apply N.V (plan `lkyjcsgkqkjxn` exec `dp4c62nm2suys`, 5/5 pass)
- [x] e2e validated — example complete (archived as 2026-07-18-pickup-confirmation;
      plan registered in openspec/specs/dispatch-tracking/plans/)
