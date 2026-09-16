# Effort A — Candidate Log

Effort A improves the architecture and code quality of the NRS Automation
Platform without changing behaviour. An architecture review produced nine
candidates, plus a bench of smaller clean-ups.

Domain terms: **Project** (one test target), **Journey** (one user path in one
environment stage), **Requirement** (one testable statement), **Generated test**
(machine-written test code), **Spec** (the `.spec.ts` file), **Materialize**
(turn generated code into a spec plus a database row), **Feature** (a test's
group), **Review state** (draft → active), **Confidence** (a separate quality
axis), **Event** (`tests_updated`).

## Done

### 1 — One materialization module (server)

**Problem.** Several flows wrote generated tests to disk and to the database,
each with slightly different rules for tags, the Feature, and validation.

**Change.** One module owns materialization: it writes the spec, inserts the
row, canonicalises the tags, resolves the Feature, validates the spec, and
emits the `tests_updated` Event.

**Effect.** One place to change. The spec file, the test library, and the run
filters now agree because they read the same resolved Feature.

### 2 — One journey-test gate and title-tag grammar (server)

**Problem.** The rule that decides which generated tests are runnable, and the
grammar that reads tags out of a test title, were spread across call sites.

**Change.** One gate and one tag grammar, shared by every reader and writer.

**Effect.** A title is parsed the same way everywhere, so a test cannot be
"runnable" in one place and "draft" in another.

### 3 — One LLM call envelope; a load-bearing provider seam (server)

**Problem.** Each LLM-backed feature built its own request, retry, and response
handling, and code branched on the provider id.

**Change.** One call envelope for every LLM call, and a provider registry that
owns provider differences. Behaviour no longer branches on `provider.id`.

**Effect.** A new provider is added in one place. Every feature gets the same
retry and error behaviour for free.

### 8 — Validate `project.json` (server)

**Problem.** A write-back to `project.json` could erase runtime fields that
were written by another part of the system.

**Change.** A validated schema for `project.json`, and write-backs that merge
with the current on-disk state instead of replacing it.

**Effect.** Saves no longer destroy fields they do not own.

### 4 — One client request adapter (client)

**Problem.** About nine hooks each repeated the same work: send a request,
check the status, parse the body, and build an error. There were four error
classes, three parse helpers, and the 204/409 special cases were re-derived at
each call site.

**Change.** `lib/http.ts` provides one adapter: `ApiError` (code, status,
message), `req<T>` for JSON, `reqBlob` for downloads, and `apiRequest` for the
streaming agent call. The four error classes collapsed into one. All call sites
use the adapter; the one remaining raw `fetch` in the client is inside it.

**Result.** Net **−310 lines**, plus 9 tests for the adapter (JSON, 204, a
status allow-list, error code/status/message, a non-JSON error body, and blob
downloads).

### 5 — Typed cache tree and operation-owned invalidation (client)

**Problem.** Query keys were hand-written as string arrays at every read and
every refresh. About 50 refresh calls used four different mechanisms. A typo in
a key silently refreshed nothing.

**Change.** `lib/query-keys.ts` gives one builder per query, so a wrong key is a
compile error. `lib/cache.ts` gives one function per event
(`projectsChanged`, `journeysChanged`, `testsChanged`, `runsChanged`,
`journeyChanged`, `agentChanged`, `gitStatusChanged`), each declaring the
resources it touches. The refresh is coarse on purpose: a forgotten key cannot
leave a screen stale.

**Bugs found and fixed.** A refresh used the key `agent-info` while the query
used `agent-info`'s real key `['agent','info']`, so a settings banner never
refreshed; a dead key `drafts` was removed; the project overview was never
refreshed at all.

**Result.** No raw cache refresh calls remain in the client. Two reactive
cache-subscription hooks were removed. 12 tests, including 1000 seeded
randomized runs that check every refresh matches a real query, that one
project's refresh cannot reach another project, and that a slug is preserved.

## Done

### 6 — Fracture the journey page (client)

**Problem.** One file, `JourneyDetail.tsx`, holds the whole Journey page: the
layout, five stage tabs (Documents, Requirements, Walkthrough, Tests,
Readiness), and about 18 sub-components — more than 3,200 lines. The store also
holds three near-identical job-status loops (requirement extraction, the
Walkthrough, and Generated test creation).

**Change.** Split the one file into a `journey/` folder:

- `JourneyLayout.tsx` — the layout and the five tab bodies.
- `DocumentsStage`, `RequirementsStage`, `TestsStage`, `PlanStage`,
  `ReadinessStage` — one module per stage.
- `ui.tsx` — the two shared parts (`NextStep`, `EmptyTab`).
- `confidence.ts` — the confidence thresholds and banding, shared by
  Requirements and Tests.

`JourneyDetail.tsx` is now an 8-line re-export; the route and the
user-visible behaviour are unchanged.

The store's three job-status loops became one `runJob(spec)`. A job supplies a
small descriptor (name, progress request, success text, refresh); the loop owns
the cadence, the "was I replaced?" guard, and the running / done / failed /
error outcomes. The public `start…` and `resume…` functions are unchanged.

**Result.** Six store tests with fake timers cover the running, done, failed,
thrown-status, replaced-job, and duplicate-start cases. The per-stage handle
hooks (one read/poll/mutate hook per stage) remain as a follow-up, held back
until there are component tests to protect them.

## Cross-cutting — complexity and comments

After candidate 6, every function in the codebase was brought to a cyclomatic
complexity of **10 or less** (the client started with 32 functions over 10; the
server with 43; four more in the MCP server and platform code). The worst
offenders before the pass:

| Function | Before | After |
|----------|--------|-------|
| Run configuration modal | 62 | 4 |
| Run summary panel | 40 | 1 |
| Requirements section | 33 | ≤10 |
| Assistant panel | 32 | 5 |
| Generated-tests section | 30 | ≤10 |
| Run start (server) | 29 | 6 |
| Settings patch builder | 24 | 4 |
| Test discovery (server) | 26 | 3 |

The method was always the same: extract small sub-components, move pure
booleans and ternaries into helpers or lookup tables, and replace long
`if/else` chains with guard clauses. No request shape, database write, error
code, document output, or user-visible text changed. Comments were trimmed to
short, necessary notes.

Every step was verified with type checks, the test suites (client 27, server 22),
a production build, and a lint rule for complexity.

## Planned

### 7 — Repository read models and one run-events module (server)

The runs repository returns raw rows and the routes re-map them; a summary
re-reads failed test titles once per row; the projects repository re-reads
`project.json` per row; and JSONL is parsed in two places. One read-model layer
and one run-events module.

### 9 — One shared wire-contract module (both)

The client's `types/index.ts` is manually mirrored by comments in server files.
One shared contract module replaces the copy.

## Bench — smaller clean-ups

- A shared helper for the two Playwright configs.
- A DSL reference module (script plus a checked-in artefact plus two topic maps).
- Unify the two job trackers (journey jobs and document extraction jobs).
- Split pure path layout from disk-aware resolution.
- A `requireProject` / `requireJourney` guard for the route handlers.
- A base TypeScript config and a root type check that covers every workspace.
- The Docker build lists a workspace but omits it from the build.
