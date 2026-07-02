# PRD Document Map

Product requirements for the ced-ecosystem (Aegis Harness / RANO Swarm). Reorganized from six standalone root-level PRDs into one indexed set. Each file below is a cleaned, reorganized (not summarized) version of its source — full technical content, code, schemas, and tables are preserved.

## Reading order

| # | Doc | Source | Scope |
|---|-----|--------|-------|
| 1 | [01-ecosystem-architecture.md](01-ecosystem-architecture.md) | `ruv-ecosystem-PRD.md` | **Canonical top-level doc.** Validation policy, product goals, core architectural decisions (Ruflo control plane, Herdr runtime, RTK/Headroom compression, LLMs-as-subroutines, RAN advisory/canary), system context, deterministic controller loop, DDD bounded contexts, ADR governance, roadmap, security model. All other PRDs refine this without duplicating it. |
| 2 | [02-infrastructure-stack.md](02-infrastructure-stack.md) | `infra-PRD.md` | Edge cluster hardware inventory, network topology, Kairos/AuroraBoot/netboot platform layer, k3s HA cluster, NetBird networking, inference stack (LocalAI/LiteLLM/MLX), phase plan. |
| 3 | [03-ran-platform.md](03-ran-platform.md) | `ran-prd.md` | RAN Optimization Platform: domain glossary, personas, architecture overview, functional requirements (FR-INF, FR-SONA, FR-HERDR, FR-ML, FR-MLOPS, FR-RL, FR-PY, FR-SAFE), non-functional requirements. |
| 4 | [04-herdr-integration.md](04-herdr-integration.md) | `herdr-prd.md` | Herdr as AI-native agent multiplexer: workspace layout, full `HerdrAdapter` TypeScript interface, verified CLI surface, controller loop integration, self-learning loop (Herdr × AgentDB × RuVector × RuvLLM), trajectory/log-pane schemas. |
| 5 | [05-mission-loop.md](05-mission-loop.md) | `mission-prd.md` | Mission + self-learning RAN agent loop: architecture, stop conditions, implementation/test requirements with done/future status. |
| 6 | [06-elex-ran-wasm-multihost.md](06-elex-ran-wasm-multihost.md) | `ruvllm-wasmelex.md` | ELEX RAN multi-host deployment: MCP server, browser WASM agent, Claude Code/Codex/pi.dev/Hermes host adapters. |

## Relationship to other docs

- **`/adr/`** — Architecture Decision Records (ADR-001..029+). PRDs reference ADRs inline; ADRs are the source of truth for *why* a decision was made, PRDs for *what* is being built.
- **`/ddd/`** — Domain-Driven Design bounded-context definitions (DDR-001..015+) and phase roadmap.
- **`CLAUDE.md`** (repo root) — harness configuration, agent coordination patterns, documentation ownership rules.

## Notes

- Source files (`infra-PRD.md`, `ruv-ecosystem-PRD.md`, `ruvllm-wasmelex.md`, `ran-prd.md`, `mission-prd.md`, `herdr-prd.md`) remain at repo root; this folder is the reorganized, cross-linked replacement for day-to-day reading. Consider removing the root copies once this set is reviewed, to avoid drift between two live versions of the same content.
- Inline ADR numbers inside `04-herdr-integration.md` (e.g. ADR-016/017/018) reflect the source document's own numbering at time of writing and were not remapped to the current canonical sequence in `/adr/` — cross-check by title, not number.
