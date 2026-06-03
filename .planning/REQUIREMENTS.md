# Requirements: ebind — Pause/Resume DAG Execution

**Defined:** 2026-06-03
**Core Value:** Durable, reliable workflow execution on a single dependency. A paused DAG should stay paused across process restarts, with no CPU consumed, and resume exactly where it left off.

## v1 Requirements

Requirements for initial release. Each maps to roadmap phases.

### State Machine

- [ ] **ST-01**: New DAGStatus constants `DAGStatusPausing` and `DAGStatusPaused` following existing lowercase naming convention
- [ ] **ST-02**: Pure function `HasInFlightSteps(state)` returns true if any step has StatusRunning
- [ ] **ST-03**: Pure function `CanPause(meta)` returns true only if meta.Status is DAGStatusRunning
- [ ] **ST-04**: Pure function `CanResume(meta)` returns true only if meta.Status is DAGStatusPaused
- [ ] **ST-05**: `maybeFinalize` guards against finalizing DAGs in pausing/paused status
- [ ] **ST-06**: `Cancel()` handles pausing/paused states explicitly (CAS transition to canceled)

### Scheduler Gates

- [ ] **SG-01**: Event entry gate — when DAG status is pausing or paused, Ack the completion event without processing (no step transitions, no downstream dispatch)
- [ ] **SG-02**: Dispatch gate — `enqueueReady` checks DAG status is running before enqueuing; skip if pausing/paused
- [ ] **SG-03**: Sweep recovery — on leader acquisition, sweep detects pausing DAGs and transitions them to paused; sweep skips paused DAGs entirely
- [ ] **SG-04**: `pausing→paused` transition occurs automatically when last in-flight step completes (event-driven, no ticker)

### API Functions

- [ ] **API-01**: `Workflow.DagPause(ctx, dagID) error` — CAS transition from running → pausing; if no in-flight steps, skip directly to paused
- [ ] **API-02**: `Workflow.DagResume(ctx, dagID) error` — CAS transition from paused → running; trigger DAG re-evaluation (enqueue ready steps)
- [ ] **API-03**: Both functions return descriptive errors for invalid state transitions (e.g., pause on canceled DAG, resume on running DAG)
- [ ] **API-04**: Both functions use KV CAS with retry, consistent with existing Cancel() pattern

### CLI Commands

- [ ] **CLI-01**: `ebctl dag pause <id>` — calls DagPause, displays confirmation
- [ ] **CLI-02**: `ebctl dag resume <id>` — calls DagResume, displays confirmation
- [ ] **CLI-03**: `ebctl dag ls` — status column displays PAUSING and PAUSED states alongside existing statuses
- [ ] **CLI-04**: Error messages for invalid pause/resume attempts are clear and actionable

### Testing

- [ ] **TST-01**: Pure unit tests for new state machine functions (ST-01 through ST-06)
- [ ] **TST-02**: Scheduler unit tests with MemStore + toggleElector for pause gate behavior (SG-01 through SG-04)
- [ ] **TST-03**: Integration tests with embedded NATS for end-to-end pause/resume
- [ ] **TST-04**: Race condition tests for concurrent pause + resume, pause + cancel, pause + step completion
- [ ] **TST-05**: CLI command tests

## v2 Requirements

Deferred to future release. Tracked but not in current scope.

### Enhancements

- **ST-07**: Pause reason/annotation metadata on DAGMeta (operator identity, ticket reference)
- **API-05**: Async pause — fire-and-forget with event notification when fully paused
- **CLI-05**: `--reason` flag on pause/resume commands
- **TST-06**: CAS conflict stress tests with sustained contention

## Out of Scope

Explicitly excluded. Documented to prevent scope creep.

| Feature | Reason |
|---------|--------|
| Auto-resume timer | Can be built externally by caller watching DAG status |
| Pause timeout / auto-cancel | Same — caller can Cancel on timeout |
| Handler-initiated pause | Would require new event type and state coordination |
| Cancel in-flight tasks on pause | Violates "clean pause" design |
| Per-step pause | Entirely different feature scope |
| Bulk pause/resume | Not needed for initial feature |

## Traceability

Which phases cover which requirements. Updated during roadmap creation.

| Requirement | Phase | Status |
|-------------|-------|--------|
| ST-01 | Phase 1 | Pending |
| ST-02 | Phase 1 | Pending |
| ST-03 | Phase 1 | Pending |
| ST-04 | Phase 1 | Pending |
| ST-05 | Phase 1 | Pending |
| ST-06 | Phase 1 | Pending |
| SG-01 | Phase 2 | Pending |
| SG-02 | Phase 2 | Pending |
| SG-03 | Phase 2 | Pending |
| SG-04 | Phase 2 | Pending |
| API-01 | Phase 3 | Pending |
| API-02 | Phase 3 | Pending |
| API-03 | Phase 3 | Pending |
| API-04 | Phase 3 | Pending |
| CLI-01 | Phase 4 | Pending |
| CLI-02 | Phase 4 | Pending |
| CLI-03 | Phase 4 | Pending |
| CLI-04 | Phase 4 | Pending |
| TST-01 | Phase 1 | Pending |
| TST-02 | Phase 2 | Pending |
| TST-03 | Phase 4 | Pending |
| TST-04 | Phase 4 | Pending |
| TST-05 | Phase 4 | Pending |

**Coverage:**
- v1 requirements: 23 total
- Mapped to phases: 23
- Unmapped: 0 ✓

---
*Requirements defined: 2026-06-03*
*Last updated: 2026-06-03 after roadmap creation — phase mappings confirmed, 23/23 v1 requirements covered*
