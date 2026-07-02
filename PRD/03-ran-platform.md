# RAN Optimization Platform PRD

Source: consolidated from ran-prd.md. Refines the ecosystem architecture — see /Users/cedric/work/dev/ced-ecosystem/PRD/01-ecosystem-architecture.md and /Users/cedric/work/dev/ced-ecosystem/PRD/README.md for the full document map.

---

## Document Metadata

- **Title:** RANO — Radio Access Network Optimization Platform (RAN Domain Contract)
- **Version:** 1.0-draft
- **Status:** Production-grade target specification
- **Owner:** Ced / RANO
- **Date:** 2026-06-11
- **Primary domain:** Ericsson LTE/NR RAN optimization, intent-based networking, self-learning inference, carrier-grade safety
- **Relationship:** Domain contract for the RAN platform. The ecosystem substrate — Herdr runtime, Ruflo swarm control, AgentDB/RuVector/RVF memory, RTK/Headroom compression — is specified in the ecosystem architecture (the "Aegis Harness"). This document is the authoritative source of truth for **RAN domain requirements** and refines, rather than duplicates, the harness PRD.

---

## 0. Validation and Source Policy

The shared validation and source policy (targets-to-validate vs. measured results, adapter-boundary discipline) applies to this PRD in full — see [01-ecosystem-architecture.md](./01-ecosystem-architecture.md) for the complete policy text.

This PRD is derived from the 26 Architecture Decision Records under [`../rano-k8s-agents/docs/adr/`](../rano-k8s-agents/docs/adr/), the ruvLLM inference design (`RuVector/examples/ruvLLM/docs/sparc/` and `.../docs/SONA/`), and the Aegis Harness PRD. It captures the RAN platform requirements that the harness PRD scopes out (the ML lifecycle, ruvLLM speculative decoding, the SONA three-loop learner, carrier-grade five-nines safety).

### 0.1 Mandatory Validation Rule (RAN-specific application)

```text
Performance, acceptance-rate, latency, and learning-capability figures sourced from ADRs or
the ruvLLM/SONA architecture specs are TARGETS-TO-VALIDATE, not measured production results.
Implement an adapter boundary, smoke test, and compatibility shim before any unverified figure
is hard-coded into domain logic. Distinguish component micro-benchmarks (NFR Tier A) from
system-level RAN SLAs (NFR Tier B).
```

### 0.2 Reliability Policy

No RAN parameter change reaches a live cell without passing deterministic verification gates: confidence threshold, conflict check, approval gate, and a reversible canary with a rollback plan. ML model outputs are advisory until shadow-validated and canary-promoted.

---

## 1. Executive Summary & Scope

RANO is an intent-based, multi-agent platform that optimizes Ericsson LTE (4G) and NR (5G) Radio Access Networks through closed-loop automation. Operators express high-level intents in natural language ("improve intra-frequency handover success in the downtown cluster"); domain agents translate them into parameter adjustments, predict and resolve conflicts, apply changes through safe canaries, monitor KPIs, and learn from outcomes.

### 1.1 In Scope (v1)

| Theme | Coverage |
|---|---|
| **Inference** | ruvLLM speculative decoding (self-speculative ruvltra family), FastGRNN routing, model pool/cascade |
| **Self-learning** | SONA three-loop (MicroLoRA / BaseLoRA / EWC++) + ReasoningBank + Memory Dreams |
| **Runtime** | Herdr as persistent agent substrate AND operator console |
| **ML & Intelligence** | Forecasting stack, feature engineering, self-supervised learning, cell-archetype taxonomy, MLOps lifecycle, RL, conflict resolution, Python toolchain |
| **Architecture & Safety** | `@rano/*` DDD packages, GEPA 8-stage pipeline, carrier-grade five-nines safety, per-context SLAs |

### 1.2 Out of Scope (v1) — see §11 Deferred

ZeroClaw edge runtime, O-RAN E2/A1/O1/R1 interfaces and RIC integration, and the Kubernetes/IDP vCluster platform are **deferred to future phases** and recorded in §11 so the omission is intentional and traceable.

### 1.3 Relationship to the Aegis Harness PRD

```text
Aegis Harness (ecosystem architecture)  =  the SUBSTRATE
   Herdr runtime · Ruflo swarm · AgentDB/RuVector/RVF · RTK/Headroom · cost governor · safety rails

RANO (this document)                    =  the RAN DOMAIN CONTRACT
   ruvLLM+SONA inference · @rano DDD · GEPA pipeline · ML/MLOps/RL · carrier-grade RAN safety
```

The harness PRD §15 ("Ericsson RAN Specialist Layer") gives the advisory-loop framing; this PRD supplies the full platform requirements behind it.

---

## 2. Domain Context & Glossary

**RAN** — Radio Access Network: base stations (eNodeB/gNodeB), antennas, radio protocols. **RANO** — RAN Optimization. **Intent** — operator goal expressed in natural language, mapped to one of 52 catalogued intents across 13 categories. **KPI / counter / parameter** — ~5,300 KPIs, ~5,200 counters, ~8,000 parameters across 14 RAN domains. **Cell archetype** — behavioural class of a cell used to condition optimization. **GEPA** — the 8-stage runtime optimization pipeline (Goal→Evidence→Plan→Consensus→Action→Evaluate→Adapt, plus Stage 0 enrichment). **SONA** — Self-Organizing Network Architecture, the three-loop self-learning subsystem. **Conflict graph** — 601 documented parameter↔KPI dependencies used to detect competing changes.

### 2.1 `@rano` Bounded Contexts (ADR-013)

| Context | Role |
|---|---|
| Troubleshooting (core) | Cell diagnosis, root-cause analysis |
| Learning | Pattern learning, model training, adaptation |
| Agent Orchestration | Multi-agent lifecycle, spawning, routing |
| Workflow Execution | GEPA pipeline, scheduling, approval gates |
| API Gateway | ENM interface translation, parameter ACLs |
| Alarm Management | Correlation, event-driven remediation |
| Dashboard Monitoring | Observability, alerting, operator visibility |

Shared-kernel types: `CellId`, `Confidence`, `KPIName`, `TimeWindow`, `AgentCapability`.

---

## 3. Personas & Stakeholders

| Persona | Goal | Primary surfaces |
|---|---|---|
| **RAN operator** | Express intents, approve canaries, monitor outcomes | Herdr operator console, approval gates |
| **ML / learning engineer** | Own model training, retraining cadence, drift response | Learning-Agent Hub, MLflow/Evidently |
| **Platform engineer** | Run the agent substrate, manage inference and memory | Herdr substrate, ruvLLM/SONA config |
| **Reviewer / safety officer** | Audit parameter changes, enforce carrier-grade safety | WITNESS audit, rollback manager |

---

## 4. Architecture Overview

### 4.1 Package Layering (ADR-001)

`@rano/*` TypeScript monorepo, no circular dependencies:

```text
Layer 1  @rano/core            value objects, CellId, alarms
Layer 2  @rano/coordination    router, conflict graph, agents
Layer 3  @rano/execution       parameter execution, ENM interface
Layer 4  @rano/assurance       KPI monitoring, rollback, alerting
Cross    @rano/inference       ruvLLM routing, backends, health
Cross    @rano/memory          3-tier memory, SQLite, HNSW
Cross    @rano/neural          MicroLoRA, SONA, GNN
Cross    @rano/observability   metrics, tracing, logging
Orch     @rano/pipeline        GEPA 8-stage pipeline
```

### 4.2 GEPA Pipeline (ADR-003, ADR-019)

```text
Stage 0  Enrichment   archetype lookup + Thompson sampling, <50ms, confidence gate
Stage 1  Goal         intent decomposition
Stage 2  Evidence     retrieval + baseline KPI snapshot
Stage 3  Plan         LLM-driven parameter proposals
Stage 4  Consensus    conflict-graph check + A2A negotiation   ── human approval gate ──
Stage 5  Action       ENM parameter dispatch
Stage 6  Evaluate     KPI monitoring (async)
Stage 7  Adapt        pattern storage, SONA learning (async)
```

### 4.3 Runtime Substrate — Herdr

Herdr hosts persistent RAN agent sessions and is the operator-facing console (§FR-HERDR). ruvLLM+SONA (§FR-INF, §FR-SONA) provide inference; `@rano/memory` + RVF provide durable cognition.

---

## 5. Functional Requirements

Each requirement: `FR-<AREA>-NN`. Priority P0 (must) / P1 (should) / P2 (could). Source ADR/doc in §10 traceability matrix.

### 5.1 FR-INF — ruvLLM Inference & Speculative Decoding (ADR-002, ADR-014; ruvLLM SPARC/SONA)

| ID | Pri | Requirement |
|---|---|---|
| FR-INF-01 | P0 | The platform MUST provide self-speculative decoding within the ruvltra family: draft model `ruvltra-small-0.5b` proposes tokens; verifier `ruvltra-medium-1.1b` accepts/rejects via acceptance sampling. Both run on-device. |
| FR-INF-02 | P0 | Speculative decoding MUST expose a configurable acceptance threshold and report per-request acceptance rate and realized speedup (target 1.3–1.5× per ruvLLM spec). |
| FR-INF-03 | P0 | A FastGRNN router MUST select model configuration per request from observed features, with EMA performance tracking; target router accuracy >95%. |
| FR-INF-04 | P0 | Inference MUST be served through a pluggable backend contract (Anthropic, Ollama, LM Studio, ruvLLM compat, ruvLLM native) with health checking (30s success TTL, 5s failure TTL) and cascade fallback. |
| FR-INF-05 | P1 | A 4-tier cascade (ruvltra → LM Studio → per-port llama-server → Ollama) MUST degrade gracefully on backend unavailability. |
| FR-INF-06 | P1 | All routing and acceptance decisions MUST be appended to a WitnessLog ring buffer (≥10K entries) for SONA learning and audit. |
| FR-INF-07 | P2 | A recursive/multi-pass reasoner (TRM-style) SHOULD refine hard queries above a confidence threshold. |

### 5.2 FR-SONA — Three-Loop Self-Learning (ADR-004; ruvLLM SONA docs)

| ID | Pri | Requirement |
|---|---|---|
| FR-SONA-01 | P0 | **MicroLoRA (Loop A, per-request)** MUST adjust low-rank (rank 1–2) routing/draft-acceptance weights on each interaction via `micro_lora_update()`, gated by a conditional trigger. |
| FR-SONA-02 | P0 | **BaseLoRA (Loop B, background)** MUST extract patterns from trajectory windows on a periodic background cycle and update medium-rank adapters. |
| FR-SONA-03 | P0 | **EWC++ (Loop C, deep)** MUST protect high-confidence weights from catastrophic forgetting using Fisher-information regularization (`L_total = L_task + λ·Σ Fᵢ(θᵢ−θ*ᵢ)²`), with no measurable forgetting over 10K update cycles. |
| FR-SONA-04 | P0 | SONA MUST adapt BOTH draft-model acceptance behaviour (improving FR-INF speculative speedup) AND RAN routing weights. |
| FR-SONA-05 | P1 | A ReasoningBank MUST record trajectories and distil reusable patterns (RETRIEVE→JUDGE→DISTILL→CONSOLIDATE) into `@rano/memory`. |
| FR-SONA-06 | P1 | Memory Dreams (offline consolidation) SHOULD synthesize patterns during idle/background cycles. |
| FR-SONA-07 | P0 | SONA MUST support shadow mode: learning runs in parallel with production without affecting live weights until promoted. |
| FR-SONA-08 | P1 | SONA persistence MUST align to the 3-tier memory model (working / episodic / knowledge) backed by SQLite + 384-dim HNSW vector search. |

### 5.3 FR-HERDR — Runtime Substrate & Operator Console

| ID | Pri | Requirement |
|---|---|---|
| FR-HERDR-01 | P0 | Herdr MUST host persistent RAN agent sessions with per-agent terminal panes and session persistence across restarts. |
| FR-HERDR-02 | P0 | Herdr MUST support git-worktree fan-out so multiple domain agents operate on isolated workspaces concurrently. |
| FR-HERDR-03 | P0 | Herdr MUST expose a scriptable agent API (start / send / wait / read) for deterministic orchestration of RAN loops. |
| FR-HERDR-04 | P0 | Herdr MUST provide an operator-facing console giving real-time visibility into each agent (pane-per-agent transparency), with no hidden parallelism. |
| FR-HERDR-05 | P0 | The console MUST surface canary status and approval gates and allow operators to approve, reject, or roll back proposed RAN changes. |
| FR-HERDR-06 | P1 | The console MUST stream observability events (GEPA stage telemetry, KPI deltas, drift alerts) for live RAN loops. |

### 5.4 FR-ML — ML Modeling Stack (ADR-015, 016, 017, 018, 019)

| ID | Pri | Requirement |
|---|---|---|
| FR-ML-01 | P1 | A three-tier forecasting stack MUST combine statistical (StatsForecast), GBDT (XGBoost+LightGBM with tsfresh features), and foundation models (Chronos-2/Moirai/TimesFM/TTM), meta-stacked by AutoGluon, with conformal 90% intervals. |
| FR-ML-02 | P1 | Cold-start forecasting MUST follow Day 0–7 (foundation zero-shot) → Day 7–28 (geo-cluster transfer) → Day 28+ (full stack). |
| FR-ML-03 | P1 | Feature engineering MUST extract tsfresh (794) + catch22 (22) features and apply two-stage selection: Benjamini-Yekutieli FDR filter then SHAP pruning to 30–50 features, retaining spatial features always. |
| FR-ML-04 | P2 | Label-scarce domains (<1% labelled) MUST use TF-C self-supervised pretraining (dual time/frequency branch, NT-Xent loss) with NannyML CBPE performance estimation without labels. |
| FR-ML-05 | P1 | A two-level cell-archetype taxonomy MUST classify cells into 6 global archetypes + ML-discovered (K-means) sub-archetypes, with confidence scoring; cells below 0.6 confidence are excluded from automation. |
| FR-ML-06 | P0 | GEPA Stage 0 MUST inject archetype context and archetype-specific thresholds, and run a PolicyKernel Thompson-sampling bandit for parameter selection, within a <50ms budget. |

### 5.5 FR-MLOPS — ML Lifecycle (ADR-020, 022, 023, 024)

| ID | Pri | Requirement |
|---|---|---|
| FR-MLOPS-01 | P0 | A centralized Learning-Agent Hub MUST own all ML training, `.rvf` packaging, archetype catalog, COST_CURVE management, drift detection, and retraining orchestration; domain agents are consumers that load `.rvf` and report outcomes via NATS. |
| FR-MLOPS-02 | P1 | Retraining cadence MUST be domain-specific: daily (mobility, interference, BLER, load-balancing), weekly (throughput, accessibility, retainability, power), monthly (capacity, energy, coverage, antenna), plus drift-triggered. |
| FR-MLOPS-03 | P0 | Drift monitoring MUST run every 15 min on 1-hour rolling windows (Evidently AI: PSI, KS, Wasserstein) with tiers Warning (PSI>0.2) → Critical (PSI>0.5, retrain) → Severe (PSI>1.0, pause serving). |
| FR-MLOPS-04 | P1 | MLflow MUST track every training run (data hash, features, hyperparameters, metrics) and a model registry MUST manage name/version/stage/artifacts. |
| FR-MLOPS-05 | P0 | Model deployment MUST follow Shadow (1–4 wk, zero-risk) → Canary (5–10% stratified cells, auto-rollback on >2% KPI delta) → Full (10→25→50→100%), with risk-tiered durations and champion-challenger tracking. |

### 5.6 FR-RL — Reinforcement Learning & Conflict Resolution (ADR-021, 025)

| ID | Pri | Requirement |
|---|---|---|
| FR-RL-01 | P2 | Inter-frequency load balancing MAY use PPO (Stable-Baselines3): state = PRB util/HO rates/LB offsets/time encodings; continuous action [−6,+6] dB; reward balances imbalance reduction, HO success, and action magnitude. Targets ≥20% PRB imbalance reduction, ≥95% HO success. |
| FR-RL-02 | P0 | Cross-intent conflict MUST be predicted by an XGBoost model over parameter-change→KPI-delta outcomes, with a rule-based fallback from the enriched conflict catalog when model confidence <70%. |
| FR-RL-03 | P0 | Conflict arbitration MUST use Byzantine consensus among 2/3+ affected domain agents, producing one of: execute original, execute modified, defer, reject. |
| FR-RL-04 | P0 | RL/conflict outcomes MUST integrate with WITNESS audit and the Rollback Manager (§FR-SAFE). |

### 5.7 FR-PY — Python ML Toolchain & Language Boundary (ADR-026)

| ID | Pri | Requirement |
|---|---|---|
| FR-PY-01 | P1 | Python dependency and GPU-job management MUST use UV + SkyPilot, with per-domain dependency groups and a cross-platform `uv.lock`. |
| FR-PY-02 | P0 | There MUST be NO Python at inference time: training exports ONNX (sklearn/XGBoost/LightGBM) or joblib (AutoGluon) + JSON feature config into RVF `MODEL_SEG`; inference is TypeScript/Rust via ruv-FANN or ONNX runtime. |
| FR-PY-03 | P1 | The TS→Python trigger MUST be an HTTP (FastAPI) endpoint exchanging Parquet/Arrow data files; the `/ml` tree MUST be isolated from `/packages`. |

### 5.8 FR-SAFE — Carrier-Grade Safety (ADR-012)

| ID | Pri | Requirement |
|---|---|---|
| FR-SAFE-01 | P0 | A Rollback Manager MUST monitor KPIs with tiered confidence (Tier 1–4) and automatically reverse changes on regression. |
| FR-SAFE-02 | P0 | Approval Gates MUST guard GEPA stage transitions, with fast-track vs standard paths by risk tier. |
| FR-SAFE-03 | P0 | All parameter changes MUST be recorded in tamper-evident WITNESS audit trails (with ENM change history). |
| FR-SAFE-04 | P0 | The system MUST degrade gracefully: confidence-tier fallback, single-agent mode, and offline degraded operation. |
| FR-SAFE-05 | P1 | The safety posture MUST target five-nines (99.999%, ≤5.26 min/yr downtime) through redundant safety pathways. |

---

## 6. Non-Functional Requirements

Two clearly separated tiers.

### 6.1 NFR Tier A — Component Micro-Benchmarks (TARGETS-TO-VALIDATE; ruvLLM/SONA specs)

| ID | Metric | Target |
|---|---|---|
| NFR-A-01 | MicroLoRA forward | <50μs (stretch <20μs) |
| NFR-A-02 | MicroLoRA update | <100μs |
| NFR-A-03 | BaseLoRA forward | <200μs |
| NFR-A-04 | Pattern search (HNSW, ef=50) | <1ms, recall ≥0.98 @1K |
| NFR-A-05 | Self-learning throughput cost | ~15% reduction for full SONA |
| NFR-A-06 | SONA memory footprint | <100–200MB |
| NFR-A-07 | Speculative-decode acceptance/speedup | report acceptance rate; 1.3–1.5× speedup |
| NFR-A-08 | Max regression threshold | ≤10% before rollback |

### 6.2 NFR Tier B — System-Level RAN SLAs (ADR-009)

| ID | Metric | Target |
|---|---|---|
| NFR-B-01 | Per-bounded-context latency | Per-context SLAs (Learning, Troubleshooting, Orchestration, API Gateway, Dashboard, Workflow, Alarm) |
| NFR-B-02 | Control-loop timing | Non-RT (>1s) and legacy (>1s) loops; bottleneck is network-bound ENM/LLM, not compute |
| NFR-B-03 | GEPA Stage 0 budget | <50ms (typical <42ms) |
| NFR-B-04 | Availability | five-nines target (see FR-SAFE-05) |
| NFR-B-05 | Baseline honesty | NFRs MUST reflect corrected ADR-009 baselines, not aspirational PRD claims |

---

## 7. Acceptance Criteria

| ID | Criterion |
|---|---|
| AC-01 | Speculative decoding runs ruvltra-small draft + ruvltra-medium verifier and reports acceptance rate + speedup (FR-INF-01/02). |
| AC-02 | All three SONA loops are active and shadow-mode toggleable; EWC++ shows no forgetting over a 10K-cycle test (FR-SONA-01..04, 07). |
| AC-03 | Herdr hosts persistent multi-agent RAN sessions with pane-per-agent visibility and operator approve/reject/rollback (FR-HERDR-01/04/05). |
| AC-04 | A new model traverses Shadow→Canary→Full with automatic rollback on >2% KPI delta (FR-MLOPS-05). |
| AC-05 | Cell archetype classification gates automation at <0.6 confidence; Stage 0 stays within 50ms (FR-ML-05/06). |
| AC-06 | Cross-intent conflicts are blocked/escalated via XGBoost + Byzantine consensus, with rule-based fallback (FR-RL-02/03). |
| AC-07 | No Python executes at inference time; models load from RVF `MODEL_SEG` as ONNX/joblib (FR-PY-02). |
| AC-08 | Every live parameter change has a WITNESS audit entry and a rollback plan (FR-SAFE-01/03). |

---

## 8. KPIs & Telemetry

RAN + ML observability metrics (100+ Prometheus ML metrics per ADR-023): `ran_canary_success_rate`, `ran_rollback_count`, `drift_retrain_count`, `archetype_confidence`, `forecast_mape`, `silhouette`, `model_f1`, `spec_decode_acceptance_rate`, `spec_decode_speedup`, `router_accuracy`, `sona_weekly_gain`, `training_duration`, `canary_pass_rate`. Telemetry events carry `event_id`, `cell_id`, `intent_id`, `stage`, `status`, `confidence`, `kpi_deltas`.

---

## 9. Phased Roadmap

| Phase | Delivers | FR areas |
|---|---|---|
| R1 | `@rano` package skeleton + GEPA pipeline + Herdr substrate | FR-HERDR, §4 |
| R2 | ruvLLM inference + speculative decoding + FastGRNN routing | FR-INF |
| R3 | SONA three-loop self-learning + ReasoningBank | FR-SONA |
| R4 | ML modeling stack + cell archetypes + Stage 0 | FR-ML |
| R5 | MLOps lifecycle (Learning-Agent Hub, drift, shadow-canary-full) | FR-MLOPS |
| R6 | RL + conflict resolution + Python toolchain | FR-RL, FR-PY |
| R7 | Carrier-grade safety hardening (five-nines) | FR-SAFE |
| R8+ | Deferred: ZeroClaw, O-RAN interfaces, K8s/IDP (see §11) | — |

---

## 10. ADR Traceability Matrix

| Requirement area | Source ADR(s) / doc | PRD coverage status |
|---|---|---|
| §4 Architecture / packages / GEPA | ADR-001, 003, 013 | New — extends harness §17 DDD |
| FR-INF | ADR-002, 014; ruvLLM SPARC 01/03 | New — extends harness §11 routing |
| FR-SONA | ADR-004; ruvLLM SONA 00/02/03 | New — extends harness §9 memory |
| FR-HERDR | harness §3.2, §15; `runbooks/herdr-workflow.md` | Extends existing harness sections |
| FR-ML | ADR-015, 016, 017, 018, 019 | New |
| FR-MLOPS | ADR-020, 022, 023, 024 | New |
| FR-RL | ADR-021, 025 | New |
| FR-PY | ADR-026 | New |
| FR-SAFE | ADR-012 | New — extends harness §24.6 |
| NFR Tier A | ruvLLM/SONA 08 benchmarks | New |
| NFR Tier B | ADR-009 | New |
| Memory/RVF/HNSW support | ADR-006, 008 | Partial — RVF packaging only (edge runtime deferred) |

---

## 11. Deferred / Out of Scope (v1)

Explicitly deferred to keep v1 focused. Each is a future roadmap phase, not an omission.

| Deferred capability | Source ADR | Reason / future phase |
|---|---|---|
| **ZeroClaw edge runtime** (Rust, on-device agents) | ADR-006, 007 | Inference refocused on ruvLLM+SONA+Herdr for v1; revisit for edge deployment. RVF model/memory packaging is retained. |
| **O-RAN interfaces** (E2/A1/O1/R1, Near-RT/Non-RT RIC, NETCONF/YANG) | ADR-011 | v1 uses ENM REST (PM/FM/CM) only; O-RAN migration is a later phase. |
| **Kubernetes / IDP platform** (3-vCluster, SkyPilot, KEDA, Crossplane, ArgoCD, GPU scheduling) | ADR-005 | Infrastructure layer; v1 targets the agent/ML domain contract. SkyPilot is referenced only for Python ML jobs (FR-PY-01). |

---

## 12. Cross-References

- [01-ecosystem-architecture.md](./01-ecosystem-architecture.md) — ecosystem substrate / Aegis Harness (Herdr, Ruflo, memory, compression, RAN advisory loop §15) and the full Validation and Source Policy.
- [`../rano-k8s-agents/docs/adr/`](../rano-k8s-agents/docs/adr/) — the 26 source ADRs.
- [herdr-prd.md](../herdr-prd.md) — Herdr runtime PRD.
- [`runbooks/herdr-workflow.md`](../runbooks/herdr-workflow.md) — Herdr operational workflow.
- [README.md](./README.md) — PRD document map.
