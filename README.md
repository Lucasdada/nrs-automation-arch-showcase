# NRS Automation Platform — Architecture Work (Effort A)

A public case study of an architecture-focused refactor of the **NRS Automation
Platform**: a Playwright + TypeScript test-automation platform with a React
dashboard. The platform turns a source document into **Requirements**, then into
**Generated tests**, runs them against a **Journey**'s environment, and reports
readiness.

The implementation code is private. This repository documents the work: the
candidates, the decisions, and the measured outcomes.

## What Effort A set out to do

Improve the architecture and code quality of the platform without changing
behaviour. An architecture review produced nine candidates. Each candidate was
taken through the same loop: **facts → a short design interview → implement →
type-check + tests → commit**.

See [`docs/METHOD.md`](docs/METHOD.md) for that loop and
[`docs/EFFORT_A.md`](docs/EFFORT_A.md) for the candidate details.

## Candidates and status

| # | Candidate | Area | Status |
|---|-----------|------|--------|
| 1 | Unify generated-test materialization into one module | Server | Done |
| 2 | Unify the journey-test gate and title-tag grammar | Server | Done |
| 3 | One LLM call envelope; make the provider seam load-bearing | Server | Done |
| 4 | One client request adapter | Client | Done |
| 5 | Typed cache tree + operation-owned invalidation | Client | Done |
| 6 | Fracture the journey page into stage modules; collapse the poll loops | Client | Done |
| 7 | Repository read models + one run-events module | Server | Planned |
| 8 | Stop `project.json` write-backs erasing runtime fields | Server | Done |
| 9 | Shared wire-contract module | Both | Planned |

A set of smaller clean-ups sits behind these: shared Playwright-config helper,
a DSL reference module, unified job trackers, a path-resolution split, request
guards for handlers, and a root type-check that covers every workspace.

## Results so far

| Candidate | Outcome |
|-----------|---------|
| 4 — client request adapter | Every request now uses one adapter. The client has **one** raw `fetch` left — inside that adapter. Four error classes collapsed to one. Net **−310 lines**, plus 9 tests for the adapter. |
| 5 — cache tree + invalidation | ~50 scattered invalidation calls replaced by one module. Two reactive cache-subscription hooks removed. Three latent bugs fixed, including a key mismatch that left a settings banner stale. 12 tests, including seeded randomized scenarios. |
| 6 — journey page fracture | A 3,257-line page split into a layout module, five stage modules, and shared UI. The route and behaviour are unchanged. Three near-identical job-status loops became one `runJob(spec)`. 6 fake-timer tests for the loop. |
| Complexity pass | Every function in all four workspaces is now cyclomatic complexity **≤ 10** (client had 32 over, server 43). Same method each time: sub-components, pure helpers, lookup tables, guard clauses. No behaviour change. |

Each candidate kept or increased the test count and passed the server and client
type checks before its commit.

## Engineering principles used

- **Small interface, deep module.** Put a lot of behaviour behind a small
  surface, so one implementation pays back across many call sites.
- **One owner per concern.** A request, a cache key, and a refresh each have
  exactly one place that defines them.
- **The interface is the test surface.** If a rule can be tested without a
  browser or a server, it is written so it can be.
- **Refactor in behaviour-preserving steps**, each verified before the next.

## Notes

- This repository is a write-up only. It contains no product source code, no
  credentials, and no operational runbooks.
- The platform's domain language (Project, Journey, Requirement, Generated
  test, Spec, Materialize, Feature, Review state, Confidence, Event) is used
  throughout.

## Licence

All rights reserved. Shared for portfolio and discussion. No licence is granted
to reuse, copy, or redistribute the contents. See [`NOTICE`](NOTICE).
