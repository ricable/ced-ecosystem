# Aegis Harness — Ecosystem Architecture PRD

Source: consolidated from ruv-ecosystem-PRD.md. This is the canonical, top-level architecture document for the ced-ecosystem PRD set — see /Users/cedric/work/dev/ced-ecosystem/PRD/README.md for the full document map. Other PRDs (RAN platform, Herdr integration, Mission loop, Infrastructure) refine or extend this document without duplicating its content.

**Title:** Aegis Harness — Herdr-Native Ruflo/Ruvnet Autonomous Multi-Agent Loop Substrate
**File:** `prd.md`
**Version:** 1.0-definitive
**Status:** Production-grade target specification
**Owner:** Ced / Aegis Harness
**Date:** 2026-06-09
**Primary domain:** Local-first autonomous engineering, Ericsson RAN optimization, durable multi-agent orchestration, token-efficient agent loops

---

## 0. Validation Policy

LLMs are nondeterministic subroutines. Prompts are generated artifacts, not the product.

This PRD treats the following as first-class infrastructure:

1. **Herdr** as persistent terminal workspace and process substrate.
2. **Ruflo / Ruvnet ecosystem** as native swarm, loop, workflow, memory, intelligence, federation, verification, and autopilot control plane.
3. **RuVector + RVF** as durable, portable cognitive memory.
4. **RuvLLM** and local model servers as first-choice inference.

---

## 1. Executive Architecture Summary

Aegis Harness is a local-first autonomous multi-agent operating substrate for engineering and Ericsson RAN optimization. It replaces interactive prompt engineering with deterministic infrastructure that manages agents, tools, memory, verification, routing, compression, and safety.

- A prompt is a single move.
- A loop is a strategy.
- The harness is a deterministic runtime that decides, verifies, remembers, adapts, compresses, and stops.

Aegis Harness must not be a prompt collection or custom chat wrapper. Execution is delegated: Ruflo executes, AgentDB remembers, RuVector and RVF retrieve, RTK compresses terminal output. The harness owns safety, policy, verification, specialization, blast radius, and the deterministic controller loop.

```text
┌──────────────────────────────────────────────────────────────────────────────┐
│ Aegis Controller                                                               │
│   state read → score → dispatch → verify → stop                                │
│   owns task graph, stop conditions, confidence, policy, RAN safety             │
├──────────────────────────────────────────────────────────────────────────────┤
│ Ruflo / Ruvnet Control Plane                                                   │
│   swarm, workflow, autopilot, memory tools, intelligence, federation, verify   │
├──────────────────────────────────────────────────────────────────────────────┤
│ Cognitive Memory                                                               │
│   AgentDB / RuVector / RVF — search, trajectory, reflections, evidence,        │
│   witnesses, graph/hyperbolic lookup                                           │
├──────────────────────────────────────────────────────────────────────────────┤
│ Compression Bus                                                                │
│   RTK (terminal) + Headroom (context) — never send uncompressed context        │
├──────────────────────────────────────────────────────────────────────────────┤
│ Hybrid Model Router                                                            │
│   local MLX/Ollama/RuvLLM/LocalAI first; Codex/GPT/Gemini escalation only      │
├──────────────────────────────────────────────────────────────────────────────┤
│ Domain Specialist Layers                                                       │
│   Ericsson RAN advisor, simulation, canary planner, rollback, optional RuView  │
└──────────────────────────────────────────────────────────────────────────────┘
```

Roles at a glance:

- Ruflo executes swarms.
- AgentDB remembers.
- RuVector retrieves.
- RVF packages cognition.
- RTK compresses terminals.

---

## 2. Product Goals

1. Allow a user to define a mission once and have the harness pursue it autonomously.
2. Treat all LLM calls as nondeterministic subroutines.
3. Use Herdr as the persistent runtime substrate.
4. Use Ruflo/Ruvnet as the native loop, swarm, and cognitive control plane.
5. Use RuVector, AgentDB, and RVF as durable memory.
6. Compress all terminal output and context before sending to models.

### 2.1 Non-Goals

The harness must not:

1. Rebuild a competing agent framework instead of using Ruflo.
2. Bypass Ruflo swarm execution with ad hoc direct prompts.
3. Store unverified transcripts as strategic memory.
4. Send uncompressed context to frontier models.
5. Perform destructive shell actions outside a sandbox.
6. Change RAN parameters outside canary-approved workflows.
7. Treat vendor performance claims as production truth without local validation.
8. Send uncompressed logs, RAG chunks, or terminal output to expensive frontier models by default.

---

## 3. Core Architectural Decisions

### 3.1 Ruflo/Ruvnet Is the Native Loop Control Plane

Aegis Harness must rely on the Ruvnet ecosystem as the native loop, memory, swarm, and cognitive orchestration substrate.

Invalid architecture:

```text
custom loop → direct agent prompts → ad hoc unverified memory
```

Required architecture:

```text
Aegis controller → Ruflo workflow/swarm → Herdr pane
  → RTK/Headroom → verification → AgentDB/RuVector/RVF memory → state
```

Aegis must wrap, constrain, extend, and specialize Ruflo. It must not replace Ruflo.

### 3.2 Herdr Owns Persistent Runtime

Herdr is the persistent terminal workspace OS.

- sessions with process persistence and detach/resume
- agent state visibility
- operator observability of terminal I/O

Ruflo runs inside Herdr.

### 3.3 RTK and Headroom Are Mandatory Compression

RTK and Headroom are mandatory optimizations. Never send uncompressed information to a model.

### 3.4 RAN Changes Are Advisory by Default

Ericsson RAN optimization is high-stakes. The RAN agent may recommend, simulate, replay, and produce canary plans. It may not perform wide live changes without explicit, verified canary policy and rollback hooks. The harness owns policy.

The `ruflo-control` swarm owns dispatch. Workers run bounded tasks.

---

## 4. System Context and Runtime Topology

Deterministic control sequence:

```text
read state → retrieve memory → compress context → score task graph → dispatch swarm
  → capture execution → verify → reflect → adapt harness → commit memory → stop/continue
```

---

## 5. Deterministic Controller Loop

### 5.1 Loop Phases

```text
read state → retrieve memory → compress context → score task graph → dispatch swarm
  → capture execution → verify → reflect → adapt harness → commit memory → stop/continue
```

### 5.2 Composite Control Loop

The controller composes several strategies:

- **ReAct**: inner worker execution can interleave tool use with local reasoning.
- **Reflexion**: failed or partial iterations generate verbal feedback and strategic lessons.
- **Life-Harness**: repeated failure changes the harness interface, tool wrappers, environment contracts, procedural skills, retry policies, or verification contracts — not merely retrying.
- **ReasoningBank-style memory**: strategic lessons are retrieved, judged, distilled, and consolidated.
- **SPARC**: Specification → Pseudocode → Architecture → Refinement → Completion, with Review and Consolidation.

### 5.3 Loop State Machine

```text
INIT → READ_STATE → RETRIEVE_MEMORY → COMPRESS → SCORE → DISPATCH
  → CAPTURE → VERIFY → REFLECT → ADAPT → COMMIT → STOP_CHECK → (READ_STATE | DONE)
```

### 5.4 Core Data Structures

```python
@dataclass
class MissionState:
    mission_id: str
    phase: LoopPhase
    open_tasks: list[Task]
    completed_task_ids: set[str]
    blocked_task_ids: set[str]
    confidence_history: list[float]
    spent_usd: float
    budget_usd: float
    stop_reason: str


@dataclass(frozen=True)
class VerificationResult:
    build_passed: bool
    test_pass_ratio: float
    semantic_review_score: float
    criteria_coverage: float
    memory_precedent_match: float
    security_clear: bool
    budget_health: float
    findings: list[Mapping[str, Any]]


@dataclass(frozen=True)
class ExecutionResult:
    task_id: str
    output: str
    rtk_summary: Mapping[str, Any]
    cost_usd: float
    input_tokens: int
    output_tokens: int
```

### 5.5 Adapter Ports

```python
class RufloAdapter(Protocol):
    def create_swarm(self, ...) -> Any:
        ...

    def run_workflow(self, ...) -> ExecutionResult:
        ...


class CompressionBus(Protocol):
    def compress(self, raw_ref: str) -> Mapping[str, Any]:
        ...
```

### 5.6 Confidence and Stop Logic

```python
def confidence(v: VerificationResult) -> float:
    return (
        0.30 * v.test_pass_ratio
        + 0.20 * v.semantic_review_score
        + 0.15 * v.criteria_coverage
        + 0.10 * v.memory_precedent_match
        + 0.10 * (1.0 if v.security_clear else 0.0)
        + 0.10 * v.security_clear
        + 0.05 * v.budget_health
    )
    return confidence


def should_stop(state: MissionState) -> bool:
    if state.stop_reason:
        return True
    if state.spent_usd >= state.budget_usd:
        state.stop_reason = "budget_breach"
        return True
    if len(state.confidence_history) >= 5:
        recent = state.confidence_history[-5:]
        if max(recent) - min(recent) < 0.02 and sum(recent) / len(recent) < 0.70:
            state.stop_reason = "stagnant_confidence"
            return True
    if not state.open_tasks:
        state.stop_reason = "mission_complete"
        return True
    return False
```

### 5.7 Controller Loop Skeleton

```python
class ControllerLoop:
    def __init__(
        self,
        ruflo: RufloAdapter,
        compression: CompressionBus,
        verification: VerificationService,
        memory: MemoryService,
        cost: CostGovernor,
    ):
        self.ruflo = ruflo
        self.compression = compression
        self.verification = verification
        self.memory = memory
        self.cost = cost

    def step(self, task: Task, state: MissionState) -> MissionState:
        if not self.cost.authorize(task, state):
            state.stop_reason = "cost_governor_denied"
            return state

        # dispatch → capture → verify
        result = self.ruflo.run_workflow(task)
        verification = self.verification.verify_task(result)
        conf = confidence(verification)

        if conf >= 0.85:
            state.completed_task_ids.add(task.id)
            self.memory.commit_episode(result, verification)
        elif conf >= 0.30:
            self.memory.commit_reflection({
                "task_id": task.id,
                "type": "partial_failure",
                "confidence": conf,
                "findings": verification.findings,
            })
            state.blocked_task_ids.add(task.id)
        else:
            state.stop_reason = "low_confidence"
            self.memory.commit_reflection({
                "task_id": task.id,
                "type": "hard_failure",
                "confidence": conf,
                "findings": verification.findings,
            })

        should_stop(state)
        return state
```

If confidence is below the blocked threshold, stop.

---

## 6. Confidence Score

### 6.1 Confidence Formula

```text
confidence =
    0.30 * test_pass_ratio
  + 0.20 * semantic_review_score
  + 0.15 * criteria_coverage
  + 0.10 * memory_precedent_match
  + 0.10 * security_clear
  + 0.05 * budget_health
```

### 6.2 Thresholds

- `>= 0.85` — promote task, commit episode.
- `0.30–0.85` — partial failure; commit reflection, block task.
- `< 0.30` — hard failure; commit reflection, stop.

---

## 7. Ruflo/Ruvnet Ecosystem Integration

### 7.1 Ruflo Surfaces

Expose Ruflo through CLI, MCP, daemon, SDK, and adapter surfaces.

```text
ruflo swarm
ruflo workflow
ruflo autopilot
ruflo loop-workers
ruflo memory
ruflo intelligence
ruflo federation
ruflo verify
ruflo cost-tracker
ruflo security-audit
ruflo testgen
ruflo docs
ruflo observability
```

### 7.2 Ruflo Plugin Categories

| Category | Plugins |
|---|---|
| Core | daemon, health checks, discovery, MCP registration |
| Orchestration | swarm coordination, workflows, autopilot, goals, loop workers |
| Memory | AgentDB, RVF, RuVector, RAG memory, knowledge graph |
| Intelligence | trajectory learning, RETRIEVE/JUDGE/DISTILL/CONSOLIDATE, graph intelligence |
| Quality | test generation, browser tests, diff risk scoring, docs maintenance |
| Security | CVE scan, prompt injection defense, PII stripping, migrations policy |
| DevOps | observability, cost tracking, metrics export |
| Federation | mTLS, Ed25519 identity, trust scoring, peer authorization |

### 7.3 Ruflo Agent Profiles

All agents must be Ruflo-compatible profiles (controller, coder, reviewer, memory-agent, RAN-agent, etc.).

### 7.4 Workflow Registry

Reusable loops must be modeled as Ruflo-compatible workflows.

Workflow schema:

```json
{
  "workflow_id": "loop://implement_feature",
  "version": "1.0.0",
  "goal": "Implement a bounded feature with verification",
  "entry_agent": "architect",
  "required_gates": [
    "build_gate",
    "behavioral_test_gate",
    "semantic_review_gate",
    "security_gate",
    "memory_gate"
  ],
  "stop_conditions": [
    "budget_breach",
    "stagnant_confidence",
    "repeated_failure",
    "security_high",
    "blast_radius_unclear"
  ],
  "memory_namespaces": {
    "read": ["project-memory", "reflection", "code-patterns"],
    "write": ["trajectory", "reflection"]
  }
}
```

---

## 8. RTK + Headroom Compression Bus

### 8.1 Responsibility

- RTK compresses terminal/tool output for Codex, Gemini CLI, and Herdr panes.
- Headroom compresses files, logs, RAG chunks, memory, and MCP payloads.

RTK is lossy. Do not compress vendor-managed files.

### 8.2 Recommended Path

```text
.codex/plugins/ruflo-rtk/rtk-pre-bash.sh
.codex/settings.local.json
```

Central settings must not be modified by generated scripts unless explicitly approved.

### 8.3 Hook Wrapper Template

```bash
#!/usr/bin/env bash
set -euo pipefail

INPUT_JSON="$(cat)"
COMMAND="$(echo "${INPUT_JSON}" | jq -r '.tool_input.command // empty')"

if [ -z "${COMMAND}" ]; then
  echo "${INPUT_JSON}"
  exit 0
fi

if command -v rtk >/dev/null 2>&1; then
  REWRITTEN_COMMAND="$(rtk rewrite "${COMMAND}" 2>/dev/null || echo "${COMMAND}")"
else
  REWRITTEN_COMMAND="${COMMAND}"
fi

jq -n \
  --arg cmd "${REWRITTEN_COMMAND}" \
  --argjson orig "${INPUT_JSON}" \
  '$orig
   | .tool_input.command = $cmd
   | {
       hookSpecificOutput: {
         hookEventName: "PreToolUse",
         permissionDecision: "allow",
         permissionDecisionReason: "RTK compression active",
         updatedInput: .tool_input
       }
     }'
```

Implementation agents must test this against the actual GPT Codex/Ruflo hook contract before production use.

---

## 9. Layered Cognitive Memory

### 9.1 Memory Must Be Typed

Memory tiers:

```text
mission-state/
trajectory/
reflection/
project-memory/
code-patterns/
autopilot-patterns/
swarm-state/
security-events/
benchmark-results/
token-economics/
compression/
ran-cases/
federation/
skills/
loops/
```

### 9.2 Memory Pipeline

```text
RETRIEVE → JUDGE → DISTILL → CONSOLIDATE
```

### 9.3 Memory Record Envelope

```json
{
  "confidence": 0.87,
  "created_by": "memory-agent",
  "verified_by": "verification-gate",
  "created_at": "ISO8601"
}
```

### 9.4 RVF Container Layout

```text
.aegis/memory/project.rvf
.aegis/memory/global.rvf
.aegis/memory/ran-cases.rvf
.aegis/memory/benchmarks.rvf
.aegis/memory/compression.rvf
```

Each RVF update must be evidence-bearing, linked to:

- mission ID
- task ID
- episode ID
- source artifact
- verification result
- confidence score
- compression metrics
- parent RVF hash or version
- authoring agent
- timestamp

### 9.5 Cognitive Memory Types

| Type | Description |
|---|---|
| Working memory | Active working memory |
| Episodic memory | Episodic replay |
| Reflection memory | Reflection memory |
| Skill library | Reusable skills |
| Causal memory | Causal heuristics |
| Hierarchical context | Hierarchical context trees |
| Semantic memory | Semantic clusters |
| RAN cases | KPI/counter/config cases |
| Compression memory | Compression memory |

### 9.6 AgentDB Bandit Re-Ranking Requirement

AgentDB/RuVector retrieval must support feedback re-ranking. At minimum:

```text
final_score = 0.70 * semantic_similarity + 0.30 * historical_success_probability
```

---

## 10. RuVector / AgentDB / RVF Ports

### 10.1 AgentDB / RVF Port

```python
class RVFService(Protocol):
    def verify_rvf(self, path: str) -> Mapping[str, Any]:
        ...
```

### 10.2 RuVector Port

```python
class RuVectorService(Protocol):
    def upsert_embedding(self, namespace: str, item_id: str, vector: list[float], metadata: Mapping[str, Any]) -> None:
        ...

    def search(self, namespace: str, query_vector: list[float], limit: int) -> Sequence[Mapping[str, Any]]:
        ...

    def search_trajectory(self, ...) -> Sequence[Mapping[str, Any]]:
        ...

    def graph_neighbors(self, node_id: str, depth: int) -> Sequence[Mapping[str, Any]]:
        ...
```

---

## 11. Hybrid Model Routing and Cost Governor

### 11.1 Routing Principle

```text
Local first.
Compressed always.
Frontier only on escalation.
```

### 11.2 Routing Tiers

| Tier | Class | Use |
|---|---|---|
| Tier 0 | Local MLX/Ollama/RuvLLM/LocalAI | memory extraction, ranking, summarization, token triage |
| Tier 1 | Cheap cloud / Flash-class | bulk RAG/doc analysis |
| Tier 2 | Sonnet / GPT-mini class | standard coding and review |
| Tier 3 | Opus / GPT-5-class / Gemini Pro | architecture pivots, hard debugging |

### 11.3 Cost Governor Policy

```python
ratio = spent / budget
if ratio >= 1.00:
    return "HALT"
if ratio >= 0.85:
    return "DEMOTE_TO_LOCAL"
return "ALLOW"
```

---

## 12. Local Model Adapters

### 12.1 Local-First Requirement

The routing layer requires local inference first. Aegis must also support MLX, Ollama, LocalAI, llama.cpp, or vLLM-compatible adapters.

### 12.2 Required Adapter Boundary

```python
class LocalModelServer(Protocol):
    def generate(self, ...) -> Mapping[str, Any]:
        ...
```

### 12.3 Adaptive Weight Claims Policy

Any MicroLoRA, SONA, EWC, TurboQuant, FastGRNN router, or KV-cache compaction feature must be implemented as experimental and disabled by default until:

- unit tests pass
- benchmark proves latency improvement
- output quality regression is below threshold
- rollback path exists
- memory pollution safeguards are active

---

## 13. Optional RuView Spatial Sensing Extension

### 13.1 Status

RuView / ESP32 CSI sensing is an optional experimental extension. It must not be required for core Aegis Harness. It is useful for physical-aware edge environments, room occupancy, lab safety monitoring, or smart-home integration.

### 13.2 Safety Constraints

RuView processes sensitive physical presence and vital-sign data.

- disabled by default
- explicit operator opt-in required
- local-only processing preferred
- no cloud upload by default
- signed sensor frames required
- retention limits required
- PII/sensitive health interpretation must be labeled as non-medical and non-diagnostic
- never used for employee surveillance or safety-critical medical decisions without review

### 13.3 RuView Data Schema

```json
{
  "frame_id": "csi-uuid",
  "sensor_id": "esp32-001",
  "signature": "ed25519-sig",
  "event_type": "presence|motion|vital",
  "confidence": 0.0
}
```

---

## 14. GEPA Prompt and Policy Optimization

### 14.1 Role

GEPA optimization is not the live controller. It is an offline or scheduled optimizer for:

- controller prompt templates
- worker prompts
- review rubrics
- memory extraction instructions
- RAN case analysis prompts
- workflow recipes

### 14.2 Optimization Pipeline

```text
gold dataset
  → split D_pareto / D_feedback
  → initialize candidate prompts
  → evaluate candidates
  → reflective mutation + crossover
  → minibatch gate
  → Pareto validation gate
  → promote prompt package
  → store in skills/prompts and RVF
```

### 14.3 Promotion Rule

```json
{
  "prompt_package_id": "pkg-uuid",
  "version": "1.2.0",
  "parents": ["pkg-a", "pkg-b"],
  "scores": {
    "task_success": 0.83,
    "test_pass": 0.91,
    "token_efficiency": 0.77,
    "safety": 1.0
  },
  "pareto_status": "non_dominated",
  "created_by": "gepa-optimizer",
  "rollback_to": "pkg-previous"
}
```

---

## 15. Ericsson RAN Specialist

### 15.1 Scope

- KPI/counter/config analysis
- cell/sector/cluster case memory
- replay simulation
- canary planning
- rollback generation

### 15.2 Default Operating Mode

```text
default = advisory
canary = limited, approved, reversible
wide live change = disabled by default
```

### 15.3 RAN Loop

```text
OBSERVE
  KPI/counter alarms, topology, config

HYPOTHESIZE
  classify coverage/interference/load/mobility/scheduler/config

PROPOSE
  candidate knobs: P0, alpha, scheduler, mobility, tilt, neighbors

SIMULATE
  historical replay, no-regression constraints, blast-radius estimate

CANARY
  limited cells, rollback plan, monitoring window

LEARN
  store accepted/rejected case with evidence
```

### 15.4 RAN Case Schema

```json
{
  "domain": "ran-cases",
  "technology": "LTE|NR",
  "symptom": "UL interference rose by 15%",
  "candidate_knobs": ["p0", "alpha", "scheduler", "mobility", "tilt"],
  "evidence": {
    "counters": "UL interference rose by 15%",
    "kpis_before": {
      "avg_ul_throughput_mbps": 12.0,
      "ul_sinr_db": 8.5,
      "bler": 0.08
    },
    "kpis_after": {
      "avg_ul_throughput_mbps": 14.1,
      "ul_sinr_db": 9.2,
      "bler": 0.05
    }
  },
  "blast_radius": "cell|sector|cluster|market",
  "verdict": "accepted|rejected|needs_more_data",
  "rollback_plan_ref": "rollback-001",
  "confidence": 0.0
}
```

---

## 16. Verification and Gates

### 16.1 Gate Model

Verification is deterministic. Gates block promotion of failed tasks.

### 16.2 Gate Result Schema

```json
{
  "task_id": "task-001",
  "status": "passed|failed|blocked",
  "score": 0.91,
  "findings": [
    {
      "severity": "medium",
      "file": "src/controller.py",
      "line": 122,
      "issue": "Missing route oscillation cooldown",
      "fix_hint": "Add lockout window after model escalation"
    }
  ],
  "created_at": "ISO8601"
}
```

### 16.3 Required Test Gates

- build gate
- behavioral/test gate
- semantic review gate
- security gate
- controller memory gate

---

## 17. Domain-Driven Design

### 17.1 Bounded Contexts

#### Orchestration Context

Owns:

- workflow lifecycle
- Ruflo adapter
- Herdr pane lifecycle

Entities:

```text
Mission
Task
TaskGraph
WorkflowRun
AgentAssignment
LoopIteration
StopCondition
ConfidenceScore
```

#### Swarm Context

Owns:

- agent
- worker
- delegation

Entities:

```text
AgentProfile
RoleContract
```

#### Memory Context

Entities:

```text
Episode
Trajectory
Reflection
MemoryRecord
RVFContainer
```

#### Routing Context

Entities:

```text
Provider
BudgetPolicy
RouteDecision
CostRecord
RouteCooldown
```

#### Verification Context

Owns:

- gates
- findings
- evidence
- confidence calculation

Entities:

```text
VerificationRun
GateResult
Finding
EvidenceRef
ConfidenceScore
```

#### RAN Domain Context

Owns:

- Ericsson RAN cases
- KPI/counter ingestion
- P0/alpha policy
- replay simulation
- canary safety

Entities:

```text
RANCase
KPIWindow
CounterSnapshot
ParameterCandidate
CanaryPlan
RollbackPlan
BlastRadius
```

### 17.2 Ubiquitous Language

| Term | Meaning |
|---|---|
| Mission | User-level goal |
| Task | Atomic executable unit |
| Workflow | Reusable loop |
| Agent | Bounded worker |
| Gate | Deterministic verification checkpoint |
| Reflection | Distilled strategic lesson |
| RVF | Portable cognitive memory container |
| Blast radius | Scope of impact for a change |
| Compression bus | RTK + Headroom pruning pipeline |

---

## 18. ADR Governance

### 18.1 Required ADRs

- ADR-001: Aegis wraps Ruflo, does not replace it
- ADR-002: Ruflo is the native control plane
- ADR-003: Herdr owns persistent runtime
- ADR-004: AgentDB + RuVector + RVF are mandatory memory
- ADR-005: RTK + Headroom are mandatory compression
- ADR-006: Controller owns stop authority
- ADR-015: Adapter boundaries protect against unstable ecosystem commands

### 18.2 ADR Template

```markdown
# ADR-XXX: Title

## Status
Proposed | Accepted | Superseded

## Context
What problem or constraint led to this decision?

## Decision
What decision is made?

## Consequences
Positive and negative consequences.

## Alternatives
Alternative A
Alternative B

## Verification
How the decision will be tested.

## Rollback
How to reverse this decision if wrong.
```

---

## 19. London School TDD Implementation Methodology

### 19.1 Rule

Every adapter, controller service, memory service, compression service, router, workflow executor, and RAN policy must be developed outside-in with London School TDD.

### 19.2 Pattern

- ControllerLoop must call RufloAdapter.run_workflow with the workflow.
- It must call VerificationService before MemoryService.commit.
- It must not promote task status unless confidence passes the threshold.

### 19.3 Example Test Prompt for an AI Coding Agent

```text
Mock:
  RufloAdapter
  MemoryService
  VerificationService
  CostGovernor
  CompressionBus

Given:
  One open task exists.
  Ruflo returns an implementation result.
  Verification passes.
  Confidence is above 0.85.

Assert:
  run_workflow is called once.
  verify_task is called once.
  commit_episode is called after verification.
  task status is promoted.
  no stop condition is raised.

Do not implement production code.
Only write the failing test.
```

### 19.4 Implementation Cycle

```text
1. Read PRD and ADR
2. Write failing test
3. Mock collaborators
4. Define adapter port
5. Implement
6. Run RTK-wrapped tests
7. Run verification gate
8. Run semantic review
9. Commit trajectory memory
10. Extract failure reflection
11. Update task graph
```

---

## 20. Agent Prompt Guidance

### 20.1 Controller Prompt

```text
You are the Aegis Controller running aegis-loop.

Mission:
Complete the active mission without further human prompting unless a stop condition is hit.

Contract:
- Read mission state, task graph, verification ledger, cost ledger, and memory summary.
- Retrieve similar cases before selecting a task.
- Select exactly one highest-value task unless parallelism is explicitly safe.
- Dispatch bounded work through Ruflo workflows.
- Require verification before state promotion.
- Require evidence before memory promotion.
- Stop on destructive actions, security high, cost breach, route oscillation, missing credentials,
  memory corruption, or unclear RAN blast radius.

Return a typed decision.
```

### 20.2 Coder Prompt

```text
You are a bounded implementation worker.

- Do not broaden scope.
- Do not modify architecture unless the task explicitly requires it.
- Do not commit directly to main.
- Run tests through RTK-wrapped commands.

Return "ok|blocked|failed" with evidence.
```

### 20.3 Reviewer Prompt

```text
You are the semantic review agent.

Score the change against acceptance criteria.
Return findings with severity, file, line, issue, and fix hint.
```

### 20.4 Memory Agent Prompt

```text
You are the Memory Agent.

Follow:
1. RETRIEVE similar memories.
2. JUDGE whether this episode changes existing knowledge.
3. DISTILL concise reusable lessons.
4. CONSOLIDATE only evidence-bearing deltas.

Do not promote raw transcript as strategic memory.
Return a typed memory delta.
```

### 20.5 RAN Agent Prompt

```text
You are the Ericsson RAN Specialist Agent.

Default mode is advisory only.
Analyze KPI/counter/config evidence.
Recommend only bounded, reversible candidate actions.
Never propose live-wide changes without canary, rollback, and blast-radius classification.

Return:
{
  "risk": "low|medium|high",
  "confidence": 0.0
}
```

---

## 21. Skills and Loop Registry

### 21.1 Skills Directory

```text
skills/
  loops/
    implement_feature.yaml
    fix_bug.yaml
    write_prd.yaml
    generate_adr.yaml
    optimize_ran_case.yaml
    simulate_ran_canary.yaml
    incident_recovery.yaml
  prompts/
    controller.md
    coder.md
    reviewer.md
    memory.md
    ran-agent.md
  tools/
    rtk_shell.yaml
    headroom_context.yaml
    ruvector_search.yaml
    agentdb_commit.yaml
```

### 21.2 Skill Schema

```json
{
  "skill_id": "skill://run_tests",
  "version": "1.0.0",
  "description": "Run RTK-wrapped test suite",
  "inputs": {},
  "outputs": {}
}
```

---

## 22. Repository Structure

```text
aegis/
  controller/
    loop.py
    confidence.py
    stop_conditions.py
  adapters/
    ruflo_adapter.py
  compression/
    rtk_wrapper.py
    headroom_proxy.py
    token_economics.py
  routing/
    model_router.py
    cost_governor.py
  providers/
    local_mlx.py
    ollama.py
    ruvllm.py
    codex.py
    openai.py
    gemini.py
  verification/
    gates.py
    semantic_review.py
    security_policy.py
    evidence.py
  ran/
    kpi_window.py
    parameter_candidate.py
    canary_plan.py
    rollback.py
    fppc_policy.py
  sensing/
    ruview_adapter.py
    csi_schema.py
skills/
  loops/
  prompts/
  tools/
.aegis/
  state/
    mission.json
    task_graph.json
    confidence_history.json
  memory/
    project.rvf
    global.rvf
    ran-cases.rvf
    benchmarks.rvf
  logs/
  compression/
```

---

## 23. Phased Roadmap

### Phase 0 — Repository and Governance Foundation

| Task | Atomic Work | Validation |
|---|---|---|
| Create repo skeleton | Add folders from Section 22 | Tree matches PRD |
| Add ADR framework | Create ADR-001..ADR-015 stubs | ADR lint passes |
| Add DDD docs | Create bounded context docs | Context names match PRD |
| Add TDD guide | Add London School examples | Tests reference mocks |

### Phase 1 — Herdr + Ruflo Runtime Ignition

| Task | Atomic Work | Validation |
|---|---|---|
| Herdr workspace bootstrap | Create `aegis-prod` session and `aegis-loop` workspace | panes visible |
| Ruflo control pane | Start Ruflo in `ops/ruflo-control` | health command passes |
| RufloAdapter port | Define interface and mock | unit test passes |
| CLI/MCP adapter | Implement installed-version compatibility layer | smoke test passes |
| Version pinning | Record installed versions | manifest committed |

### Phase 2 — Deterministic Controller Skeleton

| Task | Atomic Work | Validation |
|---|---|---|
| TaskGraph | Implement entity and repository | unit tests |
| Stop conditions | Implement budget/stagnation/security stops | unit tests |
| Confidence | Implement score formula | unit tests |
| ControllerLoop | Outside-in TDD with mocked services | integration dry run |
| State files | Write/read `.aegis/state/*.json` | schema validation |

### Phase 3 — RTK + Headroom Compression Bus

| Task | Atomic Work | Validation |
|---|---|---|
| RTK install | Verify binary | `rtk --version` |
| RTK wrapper | Implement command compression adapter | cargo/test/git fixtures |
| Headroom proxy | Implement context compression adapter | reversible retrieval test |
| Token ledger | Record raw/compressed/sent tokens | metrics JSON |
| Compression gate | Prevent uncompressed large context | failing test |

### Phase 4 — AgentDB + RuVector + RVF Memory

| Task | Atomic Work | Validation |
|---|---|---|
| Memory schemas | Implement typed schemas | JSON schema tests |
| AgentDB service | Commit/retrieve records | integration test |
| RuVector service | Semantic/trajectory search | fixture retrieval test |
| RVF container | Export/import/verify | hash/provenance test |
| Memory promotion | RETRIEVE/JUDGE/DISTILL/CONSOLIDATE | no raw transcript promotion |

### Phase 5 — Multi-Agent Ruflo Workflows

| Task | Atomic Work | Validation |
|---|---|---|
| Agent profiles | Define controller/coder/reviewer/etc. | schema lint |
| Workflow registry | Implement core `loop://*` workflows | workflow tests |
| Swarm dispatch | Controller dispatches via RufloAdapter | mocked test |
| Review gate | Reviewer output schema | fixture test |
| Memory agent | Reflection extraction | evidence test |

### Phase 6 — Verification and Security Hardening

| Task | Atomic Work | Validation |
|---|---|---|
| Build gate | Run compile/build | fixture pass/fail |
| Test gate | Run unit/integration tests | pass ratio |
| Semantic review | Review agent + deterministic parser | confidence output |
| Security gate | secret/risky command scan | high severity stops |
| Sandbox | WASM/microVM adapter | dangerous command contained |

### Phase 7 — Local-First Routing and Cost Governor

| Task | Atomic Work | Validation |
|---|---|---|
| Local adapters | MLX/Ollama/RuvLLM interface | health check |
| Cloud adapters | Codex/OpenAI/Gemini interface | mock first |
| Cost ledger | track cost by call | budget tests |
| Route policy | local/cheap/frontier selection | deterministic tests |
| Oscillation lock | cooldown route logic | unit test |

### Phase 8 — DSPy GEPA Offline Optimization

| Task | Atomic Work | Validation |
|---|---|---|
| Gold dataset | 100+ examples | no overlap split |
| Signatures | Define prompt modules | schema tests |
| Evaluator | Multi-objective metrics + ASI | fixture test |
| GEPA run | Pareto archive | candidate report |
| Promotion | Write prompt package | rollback test |

### Phase 9 — Ericsson RAN Specialist

| Task | Atomic Work | Validation |
|---|---|---|
| KPI schema | Define KPIWindow/CounterSnapshot | schema tests |
| FPPC policy | P0/alpha candidate generation | rule tests |
| Replay sim | historical window replay | fixture replay |
| Canary plan | generate rollback | no plan → stop |
| RAN memory | commit accepted/rejected cases | RVF test |

### Phase 10 — Federation and Multi-Host

| Task | Atomic Work | Validation |
|---|---|---|
| Federation config | trust-gated peers | disabled by default |
| mTLS/Ed25519 | verify identities | handshake test |
| PII stripper | outbound memory filter | redaction test |
| Trust scoring | peer privilege levels | downgrade test |
| Multi-host workflow | NUC/Pi/Mac execution | smoke test |

### Phase 11 — Optional RuView Sensing

| Task | Atomic Work | Validation |
|---|---|---|
| Privacy review | document opt-in policy | accepted ADR |
| CSI schema | define frame/event types | schema test |
| Adapter | ingest local sensor events | fixture input |
| Signature check | verify sensor signatures | invalid rejected |
| Home Assistant bridge | optional output | disabled by default |

### Phase 12 — Production Readiness

| Task | Atomic Work | Validation |
|---|---|---|
| Runbooks | add incidents docs | reviewed |
| Dashboards | token/cost/confidence/gate metrics | screenshots |
| Benchmarks | SWE-style, internal toy repos, RAN replay | baseline |
| Upgrade procedure | version pin + smoke tests | documented |
| Release bundle | `prd.md`, `AGENTS.md`, bootstrap scripts | reproducible |

---

## 24. Incident Runbooks

### 24.1 Loop Stuck in Blocked

1. Freeze dispatch.
2. Snapshot Herdr panes.
3. Export `.aegis/state`.
4. Read last verification failure.
5. Trigger memory retrieval for similar failures.
6. Apply Life-Harness adaptation or mark manual intervention.
7. Do not retry blindly.

### 24.2 Repeated Task Failure

1. Mark task class as failure pattern.
2. Create reflection.
3. Switch model route or workflow.
4. Add tool wrapper or verification change.
5. Retry once after adaptation.
6. Stop if no improvement.

### 24.3 Memory Corruption

1. Stop memory writes.
2. Switch AgentDB/RVF to read-only.
3. Verify last good RVF hash.
4. Rebuild RuVector indexes.
5. Replay from last verified episode.
6. Commit corruption incident.

### 24.4 Provider Outage

1. Demote noncritical tasks to local.
2. Queue frontier-only tasks.
3. Disable route oscillation.
4. Log provider status.
5. Resume after health check.

### 24.5 Security High

1. Stop all writes/deployments.
2. Snapshot worktree and panes.
3. Isolate offending worker.
4. Run secret scan and dependency scan.
5. Require manual review.
6. Commit security event memory.

### 24.6 RAN Canary Regression

1. Execute rollback immediately.
2. Stop RAN live actions.
3. Preserve KPI/counter before/after evidence.
4. Commit negative RAN case.
5. Require manual review before future canary.

---

## 25. Benchmark and Telemetry Plan

### 25.1 Benchmark Families

| Family | Purpose |
|---|---|
| Controller dry-run | deterministic loop correctness |
| Adapter smoke tests | Herdr/Ruflo/AgentDB/RuVector/RTK/Headroom |
| Toy coding missions | end-to-end feature/fix loop |
| SWE-style tasks | coding quality |
| AgentBench/WebArena-like tasks | long-horizon behavior |
| Token economics | compression and cost savings |
| RAN replay bench | domain decision quality |
| Security scenarios | stop condition correctness |

### 25.2 Required Metrics

```text
mission_completion_rate
mean_iterations_per_task
mean_confidence
confidence_trend
stop_reason_count
raw_tokens
compressed_tokens
sent_tokens
compression_ratio
cost_usd
avoided_cost_usd
memory_hit_rate
memory_reward_score
route_oscillation_count
verification_pass_rate
security_findings
ran_canary_success_rate
ran_rollback_count
```

### 25.3 Observability Event Schema

```json
{
  "event_id": "evt-uuid",
  "time": "ISO8601",
  "mission_id": "mission-001",
  "task_id": "task-001",
  "event_type": "dispatch|verify|memory_commit|stop|route|compression",
  "agent": "controller",
  "status": "ok|failed|blocked",
  "confidence": 0.91,
  "tokens": {
    "raw": 10000,
    "sent": 1200
  },
  "cost_usd": 0.12,
  "message": "semantic review passed"
}
```

---

## 26. Security Model

### 26.1 Secrets

- secrets never enter prompts by default
- secrets injected per task only
- secret values redacted from logs
- memory writes containing secrets rejected
- `.env` and credential paths guarded

### 26.2 Federation

- disabled by default
- mTLS + Ed25519 identities required
- trust scores required
- PII stripping before outbound exchange
- untrusted peers cannot access private memory
- all federation events audited

### 26.3 Tool Safety

- command classification before execution
- sandbox for risky commands
- approval gate for destructive actions
- no production deployment from worker agents

---

## 27. AGENTS.md Guidance

The repository must include `AGENTS.md` with:

```markdown
# Aegis Harness Agent Rules

1. Read `prd.md` before implementing.
2. Read relevant ADR.
3. Follow London School TDD.
4. Write failing tests before production code.
5. Use adapter interfaces, not hard-coded CLI commands.
6. Do not bypass Ruflo for swarm execution.
7. Do not bypass RTK/Headroom for large context.
8. Do not commit long-term memory without evidence.
9. Do not perform destructive shell actions outside sandbox.
10. RAN changes are advisory unless canary-approved.
```

---

## 28. Acceptance Criteria

The platform is accepted when:

1. A mission can be defined once in `.aegis/state/mission.json`.
2. The controller can select and dispatch a task through Ruflo.
3. Herdr panes persist and show controller/workers.
4. RTK compresses shell output in tests.
5. Headroom compresses RAG/memory context with retrievable originals.
6. AgentDB/RuVector/RVF store and retrieve typed memory.
7. Verification gates block failed tasks.
8. Confidence scoring controls advancement.
9. Cost governor demotes/halts correctly.
10. Stop conditions are deterministic and tested.
11. RAN agent remains advisory by default.
12. London School TDD tests exist for core ports.
13. ADRs document major architecture decisions.
14. Token economics telemetry is recorded.
15. A full dry-run mission completes without manual prompting.

---

## 29. Final Implementation Priority

The next implementation work must prioritize Ruflo integration before optional custom orchestration features.

Mandatory order:

```text
Phase 1: Herdr workspace + Ruflo control pane
Phase 2: RufloAdapter interface + mocked tests
Phase 3: Deterministic controller loop
Phase 4: AgentDB/RuVector/RVF memory layer
Phase 5: RTK + Headroom compression bus
Phase 6: Verification gates
Phase 7: Skill/loop registry
Phase 8: Cost governor and model router
Phase 9: Ericsson RAN specialist
Phase 10: Federation and multi-host execution
Phase 11: Optional RuView spatial sensing
Phase 12: DSPy GEPA optimization
```

---

## 30. Closing Principle

Aegis Harness is not a new agent framework competing with Ruflo.

Aegis Harness is a production-grade deterministic infrastructure harness that makes Ruflo, AgentDB, RuVector, RVF, RTK, Headroom, Herdr, local/cloud LLMs, and Ericsson RAN expertise operate as one coherent system.

```text
Prompts are implementation details.
Loops are reusable strategies.
Ruflo executes.
Herdr persists.
RTK and Headroom compress.
AgentDB remembers.
RuVector retrieves.
RVF packages.
Aegis governs.
```
