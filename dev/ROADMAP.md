# gate-sec - Roadmap

> Path from scaffold to a stable 1.0. Hard parts are front-loaded; each phase has hard exit criteria.
> Design and decisions: [`PLAN.md`](./PLAN.md).
>
> **Anti-deferral rule:** no listed hard task moves to a later phase unless this file records the move and the reason.

## v0.1.0 - Scaffold (DONE, 2026-10-13)
- [x] Manifest, README, CHANGELOG, REPS, dual license, CI, deny, clippy, rustfmt, DIRECTIVES, ROADMAP, PLAN.

## v0.2.0 - Design lock
Exit: the model (subject, action, resource, permission, decision), policy format, and check API written in `docs/API.md` and approved.
- [ ] Policy loading and validation rules; versioning.
- [ ] Route declaration interface for the web layer.
- [ ] Open decisions O-1 and O-2 answered.

## v0.3.0 - Roles and route checks
Exit: deny by default holds for any random policy; an undeclared route refuses to start; role checks meet the performance target.
- [ ] Action registry, roles with hierarchy (cycle rejection), grants, explicit denies.
- [ ] Compilation to bitsets; `check` for route-level decisions.
- [ ] Startup declaration check and coverage report.

## v0.4.0 - Attributes and objects
Exit: object-level checks catch every cross-owner and cross-tenant case in the test suite.
- [ ] Attribute rules: owner, tenant, time, network, freshness.
- [ ] Object-level check API.
- [ ] Scoped keys that can only narrow.

## v0.5.0 - Audit and operations
Exit: sensitive decisions are audited, denials reach sentinel, and an administrator can explain any decision.
- [ ] Audit events; denial events.
- [ ] `explain`.
- [ ] Atomic policy reload from file and database.

## v0.9.0 - Hardening
- [ ] Property tests for every invariant; adversarial review with findings fixed test-first; benchmarks.

## v1.0.0 - Stable
- [ ] Frozen after the web stack has used it end to end.
