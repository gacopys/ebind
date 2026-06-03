# Roadmap: ebind Pause/Resume DAG Execution

**Milestone:** v1 — Pause/Resume DAG Execution
**Granularity:** Coarse (3-5 phases)
**Created:** 2026-06-03
**Total v1 Requirements:** 23

---

## Phases

- [x] **Phase 1: State Machine & Pure Logic** — Add pausing/paused DAG status constants and pure state transition functions (completed 2026-06-03)
- [x] **Phase 2: Scheduler Pause Awareness** — Gate event processing, dispatch, and sweep for pausing/paused DAGs (completed 2026-06-03)
- [x] **Phase 3: Pause API + Resume API** — Go API methods for pausing and resuming DAG execution (completed 2026-06-03)
- [x] **Phase 4: CLI Commands + Integration Tests** — CLI commands and full end-to-end integration tests (completed 2026-06-03)

---

## Phase Details

### Phase 1: State Machine & Pure Logic
**Goal**: Foundation types and pure state logic for pausing/paused DAG statuses are correct, tested, and consistent with existing patterns.
**Depends on**: Nothing
**Requirements**: ST-01, ST-02, ST-03, ST-04, ST-05, ST-06, TST-01
**Success Criteria** (what must be TRUE):
  1. `DAGStatusPausing` ("pausing") and `DAGStatusPaused` ("paused") constants exist and follow existing lowercase naming convention
  2. `HasInFlightSteps(state)` returns `true` iff at least one step has `StatusRunning`
  3. `CanPause(meta)` returns `true` only when `meta.Status == DAGStatusRunning`; `CanResume(meta)` returns `true` only when `meta.Status == DAGStatusPaused`
  4. `maybeFinalize` does not transition pausing or paused DAGs to any final state
  5. `Cancel()` transitions pausing and paused DAGs to canceled via CAS
  6. All pure unit tests pass with `-race` (TST-01 scope)
**Plans**: 1 plan
Plans:
- [x] 01-01-PLAN.md — Add state constants, pure pause/resume functions, update Cancel/maybeFinalize/Terminal, add PausedAt field, write unit tests

### Phase 2: Scheduler Pause Awareness
**Goal**: Scheduler correctly gates event processing, step dispatch, and sweep recovery for pausing/paused DAGs without consuming CPU for idle paused DAGs.
**Depends on**: Phase 1
**Requirements**: SG-01, SG-02, SG-03, SG-04, TST-02
**Success Criteria** (what must be TRUE):
  1. Completion events for steps in pausing/paused DAGs are Acked without triggering step transitions or downstream dispatch (SG-01)
  2. `enqueueReady` skips steps belonging to pausing or paused DAGs — no new dispatches while dormant (SG-02)
  3. On leader acquisition, sweep detects pausing DAGs and transitions them to paused; sweep skips paused DAGs entirely (SG-03)
  4. pausing→paused transition fires automatically when the last in-flight step completes, driven by the event loop without a ticker (SG-04)
  5. Scheduler unit tests with MemStore pass (TST-02 scope)
**Plans**: 2 plans
Plans:
- [x] 02-01-PLAN.md — Add scheduler pause gates and auto-transition (SG-01, SG-02, SG-03, SG-04)
- [x] 02-02-PLAN.md — Write scheduler pause unit tests (TST-02)

### Phase 3: Pause API + Resume API
**Goal**: Callers can pause and resume DAG execution via the Go API with correct error handling, CAS semantics, and consistency with the existing Cancel pattern.
**Depends on**: Phase 1, Phase 2
**Requirements**: API-01, API-02, API-03, API-04
**Success Criteria** (what must be TRUE):
   1. `Pause(ctx, wf, dagID)` CAS-transitions running→pausing (or running→paused if zero in-flight steps) and returns nil (API-01)
   2. `Resume(ctx, wf, dagID)` CAS-transitions paused→running, publishes EventResumed for scheduler to re-enqueue ready steps, and returns nil (API-02)
   3. Both functions return descriptive errors for invalid transitions — e.g., pause on canceled DAG, resume on running DAG (API-03)
   4. Both functions use KV CAS with retry (5 attempts, `ErrStaleRevision` on exhaustion), consistent with existing `Cancel()` pattern (API-04)
**Plans**: 2 plans
Plans:
- [x] 03-01-PLAN.md — Implement Pause/Resume API functions, sentinel errors, EventResumed, and scheduler handler
- [x] 03-02-PLAN.md — Write comprehensive unit tests for Pause/Resume API

### Phase 4: CLI Commands + Integration Tests
**Goal**: Operators can pause and resume DAGs from the CLI with clear output; full end-to-end system passes integration and race-condition tests.
**Depends on**: Phase 3
**Requirements**: CLI-01, CLI-02, CLI-03, CLI-04, TST-03, TST-04, TST-05
**Success Criteria** (what must be TRUE):
  1. `ebctl dag pause <id>` calls DagPause and outputs confirmation with DAG ID (CLI-01)
  2. `ebctl dag resume <id>` calls DagResume and outputs confirmation with DAG ID (CLI-02)
  3. `ebctl dag ls` displays PAUSING and PAUSED in the status column alongside RUNNING, DONE, FAILED, CANCELED (CLI-03)
  4. CLI error messages for invalid pause/resume attempts (e.g., pause on canceled DAG) are clear and actionable (CLI-04)
  5. End-to-end integration tests with embedded NATS pass (TST-03 scope)
  6. Race condition tests for concurrent pause+resume, pause+cancel, and pause+step-completion pass (TST-04 scope)
  7. CLI command tests pass (TST-05 scope)
 **Plans**: 3 plans
Plans:
- [x] 04-01-PLAN.md — CLI pause/resume commands + dag ls status filter update
- [x] 04-02-PLAN.md — E2E integration test + race condition tests
- [x] 04-03-PLAN.md — CLI command tests

---

## Progress

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. State Machine & Pure Logic | 1/1 | Complete    | 2026-06-03 |
| 2. Scheduler Pause Awareness | 2/2 | Complete    | 2026-06-03 |
| 3. Pause API + Resume API | 2/2 | Complete    | 2026-06-03 |
| 4. CLI Commands + Integration Tests | 3/3 | Complete    | 2026-06-03 |

---

## Dependency Graph

```
Phase 1: State Machine (nothing)
    ↓
Phase 2: Scheduler (needs Phase 1)
    ↓
Phase 3: API (needs Phase 1 + Phase 2)
    ↓
Phase 4: CLI + Integration (needs Phase 3)
```

---

*Last updated: 2026-06-03*
