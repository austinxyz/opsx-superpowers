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

Before `/opsx:apply`, need answers from Joe on open questions in [`../docs/signadot-integration-spec.md`](../docs/signadot-integration-spec.md):
- Plan YAML schema
- `signadot-validate` output format
- Dry-run mode availability

## Contacts

- Ani (CTO) — concept aligned
- Joe (Engineering) — implementation lead on Signadot side

## Status

- [ ] Trial account registered
- [ ] hotrod running locally
- [ ] hotrod connected to Signadot cluster
- [ ] Feature candidate chosen
- [ ] OpenSpec change opened for the feature
- [ ] `signadot-plan` skill invoked at propose
- [ ] `signadot-validate` runs at apply N.V
- [ ] e2e validated — example complete
