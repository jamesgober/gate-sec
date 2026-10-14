# gate-sec - Project Plan

> The design, strategy, and decision record for `gate-sec`. A living document: every design decision, idea, or open question about gate-sec is written here or in [`ROADMAP.md`](./ROADMAP.md) before it is acted on.
>
> **Read order:** [`../REPS.md`](../REPS.md) &rarr; [`DIRECTIVES.md`](./DIRECTIVES.md) &rarr; **this file** &rarr; [`ROADMAP.md`](./ROADMAP.md).
>
> **Status:** scaffold, `0.1.0`. Design lock is `0.2.0`.

---

## Contents

1. [Purpose](#1-purpose)
2. [The gate family](#2-the-gate-family)
3. [Where it sits](#3-where-it-sits)
4. [Scope](#4-scope)
5. [The model](#5-the-model)
6. [Policy](#6-policy)
7. [Object-level checks](#7-object-level-checks)
8. [Declaration and startup checks](#8-declaration-and-startup-checks)
9. [Audit and explanation](#9-audit-and-explanation)
10. [Threats](#10-threats)
11. [Architecture](#11-architecture)
12. [Performance targets](#12-performance-targets)
13. [Ecosystem and dependencies](#13-ecosystem-and-dependencies)
14. [Testing](#14-testing)
15. [Open decisions](#15-open-decisions)
16. [Ideas parking lot](#16-ideas-parking-lot)
17. [Decision log](#17-decision-log)

---

## 1. Purpose

gate-sec answers the second question: **may this identity do this, to this thing, right now?** It is the security gate on every action. Authentication (who you are) is `gate-net`; gate-sec takes an authenticated subject and decides.

---

## 2. The gate family

| Crate | Question |
|---|---|
| `gate-net` | who are you? |
| **gate-sec** | may you do this? |
| users core (name open) | the user records; consumes and bridges both |

---

## 3. Where it sits

```
gate-net (authn) ─▶ bouncer (identity) ─▶ [gate-sec: route] ─▶ customs (body) ─▶ handler ─▶ [gate-sec: object] ─▶ ...
```

Two kinds of check:

- **route check** before the body is read: may this subject call this action at all?
- **object check** inside the handler, once the target is loaded: may this subject do this to *this* record?

It is protocol-free and storage-free: callers pass in the subject and the resource attributes they already loaded.

---

## 4. Scope

### 4.1 In scope

Subjects, actions, resources, permissions, roles with hierarchy, attribute rules, object-level checks, step-up requirements, scoped keys, policy versions, startup declaration checks, decision audit, explanations, events.

### 4.2 Out of scope

| Not here | Where |
|---|---|
| authentication, sessions | `gate-net` |
| loading users, roles, or resources from storage | the caller and the users core |
| row filtering inside a database engine | the database (it can use gate-sec decisions as input) |

---

## 5. The model

| Concept | Content |
|---|---|
| **Subject** | principal id, roles, attributes (tenant, department, ...), session facts from gate-net (when the second factor was last proven, address prefix, client fingerprint) |
| **Action** | a declared verb, interned to a dense id (`invoice.read`, `invoice.refund`, `admin.users.ban`) |
| **Resource** | a type, an id, an owner, a tenant, and declared attributes |
| **Permission** | a grant of an action on a resource type, optionally with conditions |
| **Decision** | `Allow` or `Deny { reason }`; the reason is for audit and administrators, never for the subject |

---

### 5.1 Where the subject's data comes from (never the client)

gate-sec is storage-free, but it never decides from data the client supplied. The chain:

```
login (gate-net) ─▶ users core loads roles and attributes from the auth store
                 ─▶ snapshot written into the server-side session record (memory, backed by the durable store)
request          ─▶ gate-net verifies the token against that record, checks the account epoch
                 ─▶ gate-sec receives a reference to the snapshot (no copy, no database hit)
```

| Event | Effect | Cost |
|---|---|---|
| role or permission granted or removed | account epoch bumped; every cached session record of that account is stale on its next use and reloaded | one epoch write; one reload per live session of that account |
| role definition changed (a permission added to "editor") | policy version bumped; compiled policy swapped atomically; snapshots hold role ids, so no session reload is needed | one recompile, no request blocked |
| account suspended | explicit deny plus session revocation through gate-net and bouncer-io | immediate |

| Path | Cost (estimate) |
|---|---|
| route check, role-only | under 50 ns (bitset test) |
| object check with attribute rules | under 200 ns p99 |
| subject reload after an epoch bump | the auth store's read latency, once per session |

The client's cookie carries a random token only; roles are never read from it. See `gate-net/dev/SESSIONS.md`.

James has an idea for how authorization data is fetched and held (2026-10-13); it is recorded here when designed.

## 6. Policy

- **Deny by default.** No grant, no access. An action nobody declared is unreachable.
- **Roles:** sets of permissions; roles can inherit from other roles (cycles are rejected at load).
- **Attribute rules** for what roles cannot express:
  - ownership (`resource.owner == subject`),
  - tenancy (`resource.tenant == subject.tenant`, always enforced in multi-tenant mode),
  - time windows,
  - network (address prefix in an allowed set, for admin actions),
  - freshness (second factor proven within N minutes: step-up for sensitive actions).
- **Explicit denies** override grants (for suspensions and emergency restrictions).
- **Scoped keys:** an API key or delegated token carries a subset of its owner's permissions and can only narrow them; every use is re-checked against the owner's current rights, so revoking a right from the owner removes it from every key.
- **Versions:** each loaded policy has a version; decisions record the version they were made under, and a policy change takes effect atomically.

---

## 7. Object-level checks

The most common real-world access-control flaw is a user changing an id in a request to reach someone else's record (insecure direct object reference). Route-level checks cannot catch it.

gate-sec makes the object check a first-class call:

```
gate.check(&subject, Action::INVOICE_READ, &invoice_attrs)?;
```

and the web layer's handler API is designed so that loading a resource and checking it are one step, which makes forgetting the check hard.

---

## 8. Declaration and startup checks

- Every route and action declares its required permission.
- At startup, gate-sec receives the full list of routes from the web layer and **refuses to start** if any route has no declaration, or if a declaration names an action or role that does not exist.
- A coverage report lists every action, who can reach it, and through which role or rule, for security review.

---

## 9. Audit and explanation

- Decisions on actions marked sensitive (admin, money, data export, permission changes) are written to `audit-trail` with subject, action, resource, decision, and policy version.
- Denials are reported to `sentinel-sec`: a burst of denials across many object ids is how enumeration attacks look.
- `explain(subject, action, resource)` returns the chain of grants, rules, and denies that produced a decision, for administrators and tests. It is never exposed to the subject.

---

## 10. Threats

| Threat | Defence |
|---|---|
| insecure direct object reference | object-level checks as a first-class API |
| missing checks on new routes | deny by default plus the startup declaration check |
| privilege escalation through role edits | permission changes are sensitive (audited, step-up required) and rotate the subject's session |
| cross-tenant access | tenancy rule always enforced in multi-tenant mode |
| over-powered API keys | scoped keys that can only narrow, re-checked against the owner |
| admin takeover from a stolen session | step-up freshness and network rules on admin actions |
| enumeration by probing ids | denials reported to sentinel |

---

## 11. Architecture

```
src/
  lib.rs        crate docs, Tier-1 entry points
  subject.rs    Subject
  action.rs     action registry and dense ids
  resource.rs   resource attributes
  policy.rs     roles, grants, denies, rules; loading and validation
  compile.rs    policy compiled to bitsets and rule tables
  decide.rs     check(), Decision
  scope.rs      scoped keys
  declare.rs    route declarations and startup check
  explain.rs    explanations
  events.rs     event types
  error.rs      GateSecError (#[non_exhaustive])
```

Roles compile to permission bitsets, so a pure role check is a few word operations; attribute rules are evaluated only for permissions that carry them.

---

## 12. Performance targets

| Path | Target |
|---|---|
| role-only decision | under 50 ns |
| decision with attribute rules | under 200 ns p99 |
| policy reload | atomic swap, no request blocked |

---

## 13. Ecosystem and dependencies

| Crate | Relationship |
|---|---|
| `gate-net` | supplies subject facts (session, factor freshness); no code dependency needed beyond shared id types |
| users core | supplies roles and attributes |
| web layer | supplies route declarations; calls route and object checks |
| `audit-trail`, `sentinel-sec` | receive events |

No runtime dependencies are needed for the core.

---

## 14. Testing

- Policy cases as data: grants, denies, inheritance, every rule type, each with allow and deny expectations.
- Property tests: deny by default holds for any random policy and request; scoped keys never exceed their owner; explicit denies always win.
- Startup check: an undeclared route always fails to start.
- Benchmarks for section 12.

---

## 15. Open decisions

| Id | Question | Recommendation |
|---|---|---|
| O-1 | Policies declared only in Rust, or also loadable from a file (NOML) so administrators can change roles without a deploy? | both: Rust declares actions; roles and grants loadable from file and from the database |
| O-2 | Publish `0.1.0` to crates.io to claim the name? | yes |

---

## 16. Ideas parking lot

- Time-boxed elevated access ("break glass"): an administrator requests a permission for N minutes, with a reason, fully audited.
- Policy diff tool: what changes in who-can-do-what between two policy versions.
- Static analysis of the web layer's handlers to find resource loads without an object check.

---

## 17. Decision log

| Id | Date | Decision | Reason |
|---|---|---|---|
| D-001 | 2026-10-13 | `gate-sec` handles authorization; `gate-net` handles authentication | James's naming and split |
| D-002 | 2026-10-13 | Deny by default with a startup declaration check | nothing becomes reachable by omission |
| D-003 | 2026-10-13 | Object-level checks are a first-class API | the most common real-world access-control flaw |
| D-004 | 2026-10-13 | Storage-free: callers pass loaded attributes | no database dependency; testable in isolation |
| D-005 | 2026-10-13 | Roles compiled to bitsets | role decisions in a few word operations |
| D-006 | 2026-10-13 | Subject data comes only from the server-side session record, invalidated by account epoch; never from client-supplied data | authorization must not trust the client; epochs make role changes immediate without a database hit per request |
