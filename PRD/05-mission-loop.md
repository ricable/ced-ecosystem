# Mission + Self-Learning RAN Agent Loop PRD

Source: consolidated from mission-prd.md. Refines the ecosystem architecture's self-learning loop — see /Users/cedric/work/dev/ced-ecosystem/PRD/01-ecosystem-architecture.md and /Users/cedric/work/dev/ced-ecosystem/PRD/README.md for the full document map.

> Governing docs: this document extends `docs/ruv-ecosystem-PRD.md`.
> `docs/ran-prd.md` wins on conflict with this document for all RAN-domain specifics.
> Binding implementation contracts live in the nearest `AGENTS.md` of each affected subtree.

---

## 1. Overview & Motivation

### 1.1 What the Mission Loop Is

The **mission loop** is a persistent, iterative OODA (Observe–Orient–Decide–Act) cycle that
drives a `MissionState` through repeated invocations of `ControllerLoop.runOneIteration()` until
a stop condition fires or all tasks are complete. Unlike one-shot prompting, it:

- Accumulates SONA routing weights across iterations (MicroLoRA + BaseLoRA + EWC++ protection)
- Enforces a `DeterministicCostGovernor` budget cascade per iteration
- Emits structured OTEL spans and `.aegis/` trajectory artifacts for every herdr execution
- Commits episodes and reflections to `InMemoryAegisMemoryService` for cross-task recall

### 1.2 Why It Suits RAN Autoresearch

RAN KPI optimization (handover rates, RACH success, scheduler throughput) is not solved by
a single LLM call. It requires iterative model training, SafetyGate validation, and
canary staging — a natural fit for the OODA loop where each iteration:

1. **Dispatches** a `loop://rano_queen_swarm` workflow via the Ruflo adapter
2. **Runs** RANO Queen (`demo/rano-queen/queen.py`) across SkyPilot sandbox workers
3. **Verifies** val_RMSE < 0.08 and SafetyGate PASS via the 7-gate `VerificationService`
4. **Learns** routing weights from the trajectory via `SonaLearningConsumer`

### 1.3 How It Differs from One-Shot Prompting

| Dimension | One-shot | Mission Loop |
|---|---|---|
| State | Ephemeral | `MissionState` persists across iterations |
| Budget | Uncontrolled | `DeterministicCostGovernor` cascades per iteration |
| Learning | None | SONA MicroLoRA weights accumulate across iterations |
| Safety | None | SafetyGate + 5 stop conditions |
| Rollback | Manual | `blockedTaskIds` + `commitReflection` |

### 1.4 Success Metrics

| Metric | Target |
|---|---|
| Mission completion rate | ≥ 90% of missions reach `mission_complete` within 10 iterations |
| SONA weight convergence | `LOCAL` backend weight ≥ 0.6 after ≥ 5 outcomes |
| val_RMSE (RANO Queen) | < 0.08 across LSTM, Transformer, TCN |
| SafetyGate violations | 0 per production mission |
| Budget utilization | < 80% of `budgetUsd` at `mission_complete` |

---

## 2. Architecture & Data Flow

```
runMission(mission, deps)                      ← src/orchestration/mission_runner.ts
  │
  └─ while openTasks && !stopReason:
       ControllerLoop.runOneIteration(state)   ← src/orchestration/controller_loop.ts
         │
         ├─ evaluateStopConditions(state)       ← src/orchestration/stop_conditions.ts
         ├─ selectNextTask(state)               ← src/orchestration/task_graph.ts
         ├─ cost.authorize(task, state)         ← DeterministicCostGovernor
         ├─ memory.retrieveSimilar(task, 8)     ← InMemoryAegisMemoryService
         ├─ compression.compressContext(bundle) ← InMemoryCompressionBus
         ├─ ruflo.runWorkflow("loop://rano_queen_swarm", task)
         │     └─ RanoQueenLauncher.run()       ← src/orchestration/rano_queen_launcher.ts
         │           ├─ herdr.ensureSession / ensureWorkspace
         │           ├─ herdr.startSubagentPane → queen.py in pane
         │           │     └─ SkyPilot: LSTM / Transformer / TCN workers
         │           │     └─ SafetyGate(val_RMSE threshold)
         │           └─ herdr.waitForAgentStatus("done")
         ├─ executeInHerdr(task, herdr, swarmId, missionId, deps)
         │     ├─ emitSpan(OTEL start)
         │     ├─ herdr.waitForAgentStatus("done", TASK_TIMEOUT_MS)
         │     ├─ herdr.readPaneOutput(paneId, 50)
         │     ├─ captureTrajectory (working → done hook)
         │     │     └─ SonaLearningConsumer.recordSonaFeedback
         │     │           ├─ SONALearningLoop.recordOutcome  (MicroLoRA + BaseLoRA)
         │     │           └─ InMemoryVectorStore.upsert      (384-dim embedding)
         │     └─ emitSpan(OTEL close)
         ├─ verification.verifyTask(task, result)  ← FixtureVerificationService (7 gates)
         ├─ computeConfidence(verification)        ← src/orchestration/confidence.ts
         │     weights: test=0.30 semantic=0.20 coverage=0.20
         │             precedent=0.15 security=0.10 budget=0.05
         ├─ confidence ≥ 0.85 → completedTaskIds  /  < 0.5 → low_confidence HALT
         └─ memory.commitEpisode / commitReflection
```

### 2.1 Stop Conditions (5)

| Condition | Trigger |
|---|---|
| `budget_breach` | `spentUsd >= budgetUsd` |
| `security_high` | finding with `severity: "high"` |
| `mission_complete` | `openTasks.length === 0` after completion |
| `low_confidence` | `confidence < 0.5` after verification |
| `stagnant_confidence` | last-5 confidences: `max−min < 0.02 AND avg < 0.7` |

---

## 3. Implementation Requirements

### FR-MISSION-01: Workflow Registration ✅ Done
- `loop://rano_queen_swarm` registered in `src/swarm/workflow_registry.ts`
- Gates: `semantic_review_gate`, `security_gate`, `memory_gate`, `deployment_canary_gate`
- Entry agent: `rano-queen-agent` (added to `AgentRole` union and `defaultAgentProfiles()`)

### FR-MISSION-02: Mission Runner ✅ Done
- `src/orchestration/mission_runner.ts` — `runMission(mission, deps, config?)`
- `maxIterations` guard (default 20)
- `onIteration` progress callback
- Wires `buildHerdrLearningLoop()` automatically when `deps.herdr` is present

### FR-MISSION-03: RANO Queen Launcher ✅ Done
- `src/orchestration/rano_queen_launcher.ts` — `RanoQueenLauncher`
- `start()` returns `false` on error (never throws to callers)
- `wait()` returns `false` on timeout or error
- `read()` returns empty string on error

### FR-MISSION-04: SONA Default Wiring ✅ Done
- `runMission()` calls `buildHerdrLearningLoop()` when `herdr` is present
- `capture` from `buildHerdrLearningLoop` injected as `HerdrExecutionDeps.trajectory`
- SONA activates after 5 trajectory outcomes (microLoRA shadow mode → routing advice)

### FR-MISSION-05: RAN Mission Demo ✅ Done (standalone demo removed under ADR-027)
- `src/demo/demo-ran-mission.ts` — fully offline, 4-outcome SONA warm-up + 2 mission iterations.
  Removed on `feat/rano-demo-all` (ADR-027, zero importers); the capability remains covered by
  `tests/orchestration/mission_runner.test.ts` + `tests/orchestration/rano_queen_launcher.test.ts`.

### FR-MISSION-06: LoopPhase Instrumentation (Future)
- 12 of 17 `LoopPhase` values are defined but never set by `ControllerLoop`
- Add: `HYDRATE_MEMORY`, `COMPRESS_CONTEXT`, `DISPATCH_RUFLO`, `EXECUTE_HERDR`,
  `VERIFY`, `COMPUTE_CONFIDENCE`, `COMMIT_MEMORY` at the appropriate transitions
- No behavior change — observability only

---

## 4. Test Requirements

### TR-MISSION-01: Workflow Registration ✅ Done
- `tests/swarm/swarm.test.ts` — "workflow registry covers PRD-required loop workflows"
  extended to include `loop://rano_queen_swarm`

### TR-MISSION-02: Mission Runner ✅ Done
- `tests/orchestration/mission_runner.test.ts` — 6 test cases:
  - Single-task mission completes in one iteration
  - Multi-iteration convergence
  - `maxIterations` guard
  - Budget breach detection
  - Empty open-tasks guard
  - `onIteration` callback fires

### TR-MISSION-03: RANO Queen Launcher ✅ Done
- `tests/orchestration/rano_queen_launcher.test.ts` — 12 test cases:
  - `start()` happy path + error path
  - `wait()` success, timeout, throw, no-start
  - `read()` success, no-start, throw
  - `run()` null on start failure, timedOut, full success

### TR-MISSION-04: cherryPickFeatures (Future)
- Extend `tests/orchestration/controller_loop.test.ts`
- Cover: valid selection, empty selection, invalid selection

### TR-MISSION-05: LoopPhase Instrumentation (Future)
- Assert all mandatory phases appear in order for a happy-path iteration

---

## 5. Validation Plan

| Stage | Gate | Command |
|---|---|---|
| V1 — Offline mission | Runs without errors, mission_complete | `node --test tests/orchestration/mission_runner.test.ts` (demo-ran-mission.ts removed, ADR-027) |
| V2 — SONA convergence | `LOCAL` weight ≥ 0.6 after 5 outcomes | Was demonstrated by `demo-ran-mission.ts` (removed, ADR-027); no standalone gate remains — exercised via the herdr self-learning loop |
| V3 — Queen standalone | Produces `val_RMSE=...` in stdout | `python3 demo/rano-queen/queen.py --generate 12 --workers 2` |
| V4 — Herdr integration | Trajectory in `.aegis/trajectories/` | Live herdr session with `RanoQueenLauncher.run()` |
| V5 — Full mission | 3 real iterations, SONA persists | `runMission()` against live queen.py in herdr pane |

---

## 6. Optimization Targets

| Parameter | Current | Target |
|---|---|---|
| `completionThreshold` | 0.85 hard-coded in `controller_loop.ts` | Expose via `MissionConfig` |
| SONA shadow mode activation | 5 outcomes | Document; expose via `SONALearningConfig` |
| EWC++ protection | Default λ | Tune after oslo-north pattern stabilizes |
| `SpeculativeAcceptanceLearner` | Standalone (demo only) | Wire to herdr speculative-decoding stats |
| Confidence formula weights | test=0.30 semantic=0.20 coverage=0.20 | Make SONA-tunable after V5 validation |

---

## 7. Affected Files

| File | Status |
|---|---|
| `src/swarm/schemas.ts` | `rano-queen-agent` added to `AgentRole` |
| `src/swarm/agent_profiles.ts` | `rano-queen-agent` profile added |
| `src/swarm/workflow_registry.ts` | `loop://rano_queen_swarm` registered |
| `src/orchestration/rano_queen_launcher.ts` | Created |
| `src/orchestration/mission_runner.ts` | Created |
| `src/demo/demo-ran-mission.ts` | Created, then removed under ADR-027 (zero importers) |
| `tests/orchestration/mission_runner.test.ts` | Created (6 tests) |
| `tests/orchestration/rano_queen_launcher.test.ts` | Created (12 tests) |
| `tests/swarm/swarm.test.ts` | Extended (rano_queen_swarm in registry test) |

---

## 8. Verification Commands

```bash
# Regression suite
node --test tests/**/*.test.ts

# Targeted suites
node --test tests/orchestration/mission_runner.test.ts
node --test tests/orchestration/rano_queen_launcher.test.ts
node --test tests/swarm/swarm.test.ts

# Offline mission (V1 / V2 validation) — demo-ran-mission.ts removed under ADR-027
node --test tests/orchestration/mission_runner.test.ts

# Queen standalone (V3 validation, no herdr needed)
python3 demo/rano-queen/queen.py --generate 12 --workers 2
```
