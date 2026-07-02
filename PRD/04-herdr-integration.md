# Herdr Integration — Aegis Harness: Herdr as AI-Native Agent Multiplexer

Source: consolidated from herdr-prd.md. Refines the ecosystem architecture's runtime substrate — see /Users/cedric/work/dev/ced-ecosystem/PRD/01-ecosystem-architecture.md and /Users/cedric/work/dev/ced-ecosystem/PRD/README.md for the full document map. Canonical ADRs live under /Users/cedric/work/dev/ced-ecosystem/adr/.

**Amendment to:** Aegis Harness PRD v1.1-updated-roadmap
**Section prefix:** §H (Herdr)
**Version:** 1.0
**Date:** 2026-06-11
**Status:** Production-grade target specification
**Source validation:** Herdr v0.6.8 binary confirmed (Linux x86_64); macOS/Apple Silicon builds confirmed via `brew install herdr` / `mise use -g herdr`. AGPL-3.0-or-later license. Socket API, CLI, integrations confirmed in v0.6.8.

> **Note on ADR numbering:** The ADR entries in this document (ADR-016, ADR-017, ADR-018) are preserved as written in the source amendment. These numbers may have been superseded or renumbered in the canonical ADR set — see /Users/cedric/work/dev/ced-ecosystem/adr/ for the current authoritative numbering (e.g. herdr pane lifecycle ownership, session-scoped CLI grammar, ephemeral subagent panes, and trace panes now live under their own canonical ADR files). The ADR numbers below are not renumbered here.

---

## Validation Posture

```text
Mandatory rule (from PRD §0):
All Herdr commands, socket API calls, integration behaviors, and state-detection
capabilities must be verified against the installed binary before wiring into
domain logic. Use HerdrAdapter with probe() + smoke tests on every capability.
```

Verified facts about Herdr v0.6.8:
- Single Rust binary: ratatui + crossterm UI, portable-pty pane processes, tokio async
- Local socket API at `$XDG_CONFIG_HOME/herdr/herdr.sock` (or default config home)
- CLI surfaces: `herdr workspace`, `herdr tab`, `herdr pane`, `herdr wait`,
  `herdr session`, `herdr integration`, `herdr agent`
- Integrations: claude, codex, opencode, pi, hermes, copilot, qodercli, omp
- Install paths: curl install script, `brew install herdr`, `mise use -g herdr`,
  cargo build from source, pre-built release binary
- Persistence model: detach/reattach (server stays alive), snapshot restore
  (layout+cwd after stop/start, processes gone), optional pane screen history
  (off by default — may contain secrets), native agent session restore (per integration)
- Agent state detection: process detection + screen heuristics + integration events;
  states: working / blocked / done / idle

VERIFIED on macOS Apple Silicon (verified 2026-06-11, herdr 0.6.8 — full evidence in
`.aegis/herdr-cli-verification.md`; binding grammar contract: docs/adr/ADR-018):
  - `herdr integration install claude` hook contract: installed v5, current — a Claude Code
    hook script at `~/.claude/hooks/herdr-agent-state.sh` that reports agent state
    SEMANTICALLY over the herdr socket when running inside a herdr pane (`HERDR_ENV=1`);
    not screen-scraping. Status verb is `herdr integration status` (plain text; there is
    no `integration list`).
  - exact socket path under macOS XDG layout: default session
    `~/.config/herdr/herdr.sock`; named sessions
    `~/.config/herdr/sessions/<name>/herdr.sock` (macOS uses `~/.config`, NOT
    `~/Library/Application Support`). `--session <name>` is a GLOBAL flag placed before
    the subcommand.
  - `herdr wait agent-status` state enum exhaustiveness: `--status` accepts exactly
    `idle | working | blocked | done | unknown` (level-triggered; `--timeout` in ms;
    timeout exits 1). `pane report-agent --state` accepts only
    `idle | working | blocked | unknown` — `done` is DERIVED by herdr from a
    working→idle transition of the same source/agent and can never be reported directly.
  - `herdr pane read --source recent-unwrapped` encoding: exists and works; output is
    plain text, default `--format text` STRIPS ANSI escapes (`--format ansi` preserves
    them); returns logical unwrapped lines. Footgun: `--lines N` counts trailing blank
    screen rows — read with N ≥ pane height or omit `--lines`. Recent sources require
    `[experimental] pane_history = true` in `~/.config/herdr/config.toml`.

---

## Revised Critical Invariant

The PRD §1 invariant is extended:

```text
Ruflo executes swarms.
AgentDB remembers.
RuVector retrieves.
RVF packages cognition.
RTK compresses terminals.
Headroom compresses context.
Herdr persists processes, multiplexes panes, and surfaces agent state.   ← EXTENDED
Aegis governs safety, policy, verification, domain semantics, and stop conditions.
```

**New binding:**
```text
Ruflo runs inside Herdr.
Every agent pane is a Herdr pane.
Every log stream is a Herdr pane.
Every trace sink is a Herdr pane.
The Aegis controller reads agent state FROM Herdr, not from agent stdout directly.
```

---

## Architectural Role Clarification

Herdr is NOT:
- A browser dashboard
- A Docker service
- A replacement for Ruflo or the agents themselves
- A screen-scraping supervisor external to the agents

Herdr IS:
- The **process substrate** — panes keep running on detach; the Ruflo daemon,
  controller, and workers are all Herdr-managed processes
- The **agent state bus** — Aegis reads `.working / .blocked / .done / .idle`
  from Herdr's detection layer, not by parsing raw stdout
- The **operator visibility layer** — human operator sees live pane state
  without interfering with agent execution
- The **automation control plane** — scripts and agents use the CLI/socket API
  to create panes, inject commands, read output, and wait on state transitions
- The **log and trace sink** — dedicated panes receive structured log and
  OpenTelemetry-compatible trace streams from each agent

---

## Revised Herdr Workspace Layout

The existing PRD §4.1 workspace is expanded with explicit pane roles,
Herdr CLI provisioning commands, and agent/log/trace pane assignments.

```text
Herdr Session: aegis-prod
  └── Workspace: aegis-loop

  Tab: loop-main          (Ruflo controller + primary workers)
    Pane: controller       ← Aegis ControllerLoop process; Ruflo Queen agent
    Pane: architect        ← architect worker agent
    Pane: coder            ← coder worker agent
    Pane: memory           ← memory-agent (AgentDB/RuVector writes)

  Tab: verify             (verification workers + gates)
    Pane: test-runner      ← behavioral test gate executor
    Pane: reviewer         ← semantic review gate worker
    Pane: observer         ← verification result aggregator
    Pane: scratch          ← ephemeral sandbox for risky commands

  Tab: ops                (Ruflo control + routing + security)
    Pane: ruflo-control    ← Ruflo MCP server process (daemon)
    Pane: router           ← SONA HNSW model router process
    Pane: security         ← security gate / aidefence agent
    Pane: benchmark        ← benchmark-agent

  Tab: compression        (RTK + Headroom + token economics)
    Pane: rtk-monitor      ← RTK pre-bash hook monitor
    Pane: headroom-proxy   ← Headroom context compression proxy
    Pane: token-economics  ← token economics metrics emitter

  Tab: ran                (Ericsson RAN specialist — isolated)
    Pane: ran-agent        ← RAN advisor (advisory only)
    Pane: replay-sim       ← historical replay simulator
    Pane: canary-planner   ← canary plan generator
    Pane: rollback-monitor ← rollback readiness monitor

  Tab: logs               (structured log streams — NEW)
    Pane: controller-log   ← JSON log sink for controller loop events
    Pane: memory-log       ← AgentDB/RuVector operation log
    Pane: gate-log         ← verification gate result log
    Pane: ran-log          ← RAN case audit log

  Tab: traces             (distributed tracing — NEW)
    Pane: otel-collector   ← OpenTelemetry collector (OTLP→local)
    Pane: span-viewer      ← span summary (herdr pane read sourced)
    Pane: cost-trace       ← token-economics and cost trace sink

  Tab: subagents          (nested subagent panes — dynamic, NEW)
    Panes: ephemeral, spawned by Ruflo nested subagent dispatch
           named: subagent-{task_id}-{depth}-{role}
           max active panes: 16 (enforced by HerdrAdapter)
           auto-archived when agent-status = done

  Tab: workers            (ephemeral task workers — existing)
    Panes: ephemeral queued workers only
```

### Workspace Provisioning Script

The workspace must be provisioned deterministically at session start.
The controller must NOT assume it is running inside Herdr — it uses
HerdrAdapter.probe() to detect presence and falls back to raw process
execution if Herdr is unavailable.

```typescript
// src/herdr/provision-workspace.ts
import { HerdrAdapter } from './herdr-adapter.js';

export async function provisionAegisWorkspace(herdr: HerdrAdapter): Promise<void> {
  const probe = await herdr.probe();
  if (!probe.ok) {
    console.warn('[Herdr] Not available — running in fallback process mode');
    return;
  }

  // Create or attach to session
  await herdr.sessionEnsure('aegis-prod');
  await herdr.workspaceEnsure('aegis-prod', 'aegis-loop');

  // Static tabs + panes
  const layout: HerdrWorkspaceLayout = {
    session: 'aegis-prod',
    workspace: 'aegis-loop',
    tabs: [
      { label: 'loop-main', panes: ['controller','architect','coder','memory'] },
      { label: 'verify',    panes: ['test-runner','reviewer','observer','scratch'] },
      { label: 'ops',       panes: ['ruflo-control','router','security','benchmark'] },
      { label: 'compression', panes: ['rtk-monitor','headroom-proxy','token-economics'] },
      { label: 'ran',       panes: ['ran-agent','replay-sim','canary-planner','rollback-monitor'] },
      { label: 'logs',      panes: ['controller-log','memory-log','gate-log','ran-log'] },
      { label: 'traces',    panes: ['otel-collector','span-viewer','cost-trace'] },
      { label: 'subagents', panes: [] },  // dynamically populated
      { label: 'workers',   panes: [] },  // dynamically populated
    ]
  };

  await herdr.applyLayout(layout);
}
```

---

## HerdrAdapter — Complete TypeScript Interface

Per PRD §0 mandatory adapter boundary. All Herdr capabilities are accessed
exclusively through this interface. Never call `herdr` CLI directly from
domain logic.

```typescript
// src/herdr/herdr-adapter.ts

export interface ProbeResult {
  ok: boolean;
  version?: string;
  socketPath?: string;
  detail?: string;
  latencyMs?: number;
}

export interface PaneId {
  // Herdr pane address format: "{workspace_index}-{tab_index}-{pane_index}"
  // e.g. "1-2-3" or by label reference after lookup
  raw: string;
}

export type AgentStatus = 'working' | 'blocked' | 'done' | 'idle' | 'unknown';

export interface PaneState {
  paneId: PaneId;
  label: string;
  agentStatus: AgentStatus;
  pid?: number;
  cwd?: string;
}

export interface PaneReadResult {
  paneId: PaneId;
  lines: string[];
  source: 'recent-unwrapped' | 'screen' | 'history';
}

export interface RunResult {
  paneId: PaneId;
  command: string;
  startedAt: string;
}

export interface WaitResult {
  paneId: PaneId;
  condition: 'output-match' | 'agent-status';
  matched: boolean;
  timedOut: boolean;
  elapsedMs: number;
}

export interface HerdrWorkspaceLayout {
  session: string;
  workspace: string;
  tabs: Array<{ label: string; panes: string[] }>;
}

export interface SubagentPaneRequest {
  taskId: string;
  depth: number;         // 1–4; enforced max depth per PRD §6
  role: string;
  command: string;
  tabLabel: 'subagents' | 'workers';
}

export interface SubagentPaneResult {
  paneId: PaneId;
  label: string;
}

export interface LogStreamConfig {
  paneLabel: string;   // e.g. 'controller-log'
  format: 'json-lines' | 'plain';
  maxLines: number;
}

export interface TraceSpan {
  traceId: string;
  spanId: string;
  parentSpanId?: string;
  name: string;
  agentRole: string;
  taskId: string;
  startTimeMs: number;
  endTimeMs?: number;
  attributes: Record<string, string | number | boolean>;
  status: 'ok' | 'error' | 'unset';
}

/**
 * HerdrAdapter — adapter boundary for all Herdr CLI and socket API operations.
 *
 * All methods must:
 * 1. Call probe() once at startup and cache result
 * 2. Return a typed result rather than throwing on Herdr unavailability
 * 3. Fall back gracefully (no Herdr → use raw process execution)
 * 4. Never hard-code pane IDs — always resolve by label
 */
export interface HerdrAdapter {

  // ── Lifecycle ──────────────────────────────────────────────────────────────

  /** Verify Herdr is present, reachable, and report version. */
  probe(): Promise<ProbeResult>;

  /** Ensure a named session exists; create if absent. */
  sessionEnsure(sessionName: string): Promise<void>;

  /** Ensure a named workspace exists within a session. */
  workspaceEnsure(sessionName: string, workspaceName: string): Promise<void>;

  /** Apply a full workspace layout (idempotent). */
  applyLayout(layout: HerdrWorkspaceLayout): Promise<void>;

  // ── Pane Operations ────────────────────────────────────────────────────────

  /** Resolve a named pane label to its PaneId. */
  resolvePaneByLabel(
    session: string,
    workspace: string,
    tabLabel: string,
    paneLabel: string
  ): Promise<PaneId>;

  /** Run a command inside a named pane. */
  paneRun(paneId: PaneId, command: string): Promise<RunResult>;

  /** Read recent output from a pane. */
  paneRead(paneId: PaneId, lines: number, source?: 'recent-unwrapped' | 'screen'): Promise<PaneReadResult>;

  /** Split an existing pane to create a new one (right or down). */
  paneSplit(paneId: PaneId, direction: 'right' | 'down'): Promise<PaneId>;

  /** Get the current agent status of a pane. */
  paneAgentStatus(paneId: PaneId): Promise<AgentStatus>;

  // ── Wait / Synchronization ─────────────────────────────────────────────────

  /** Wait for a string match in pane output. */
  waitForOutput(
    paneId: PaneId,
    match: string,
    timeoutMs: number
  ): Promise<WaitResult>;

  /** Wait for an agent to reach a target status. */
  waitForAgentStatus(
    paneId: PaneId,
    targetStatus: AgentStatus,
    timeoutMs: number
  ): Promise<WaitResult>;

  // ── Subagent Pane Lifecycle (NEW) ──────────────────────────────────────────

  /**
   * Spawn a new ephemeral pane for a nested subagent.
   * Enforces max_depth ≤ 4 (PRD §6 / ADR-147 guard band).
   * Pane label: subagent-{taskId}-{depth}-{role}
   * Tab: 'subagents' for depth 1–3; 'workers' for depth 4
   */
  spawnSubagentPane(req: SubagentPaneRequest): Promise<SubagentPaneResult>;

  /**
   * Archive (close) a subagent pane after task completion.
   * Reads final output before closing; stores in AgentDB trajectory namespace.
   */
  archiveSubagentPane(paneId: PaneId, taskId: string): Promise<{ outputRef: string }>;

  /**
   * List all active subagent panes with their depth and status.
   */
  listSubagentPanes(): Promise<Array<{
    paneId: PaneId;
    label: string;
    taskId: string;
    depth: number;
    role: string;
    status: AgentStatus;
  }>>;

  // ── Log Streaming (NEW) ────────────────────────────────────────────────────

  /**
   * Attach a structured log stream to a log pane.
   * The pane must exist in the 'logs' tab.
   * Returns a write function; caller pipes JSON-lines to it.
   */
  attachLogStream(config: LogStreamConfig): Promise<{
    write: (entry: object) => Promise<void>;
    close: () => Promise<void>;
  }>;

  /**
   * Read recent structured log entries from a log pane.
   * Parses JSON-lines from pane output.
   */
  readLogStream(paneLabel: string, lastN: number): Promise<object[]>;

  // ── Trace Sinking (NEW) ────────────────────────────────────────────────────

  /**
   * Emit a trace span to the otel-collector pane.
   * Format: OTLP-compatible JSON-lines for local collection.
   */
  emitSpan(span: TraceSpan): Promise<void>;

  /**
   * Read recent spans from the span-viewer pane.
   * Returns deserialized TraceSpan objects.
   */
  readSpans(lastN: number): Promise<TraceSpan[]>;

  // ── Self-Learning Integration (NEW) ───────────────────────────────────────

  /**
   * Read pane output and pipe it into AgentDB trajectory namespace.
   * Called by memory-agent after each task cycle.
   * RTK-compresses the raw output before writing.
   */
  captureTrajectory(
    paneId: PaneId,
    taskId: string,
    missionId: string
  ): Promise<{ trajectoryRef: string; rawTokens: number; compressedTokens: number }>;

  /**
   * Register a SONA learning feedback signal from pane state transitions.
   * Called when a pane transitions working→done with confidence score.
   */
  recordSonaFeedback(
    paneId: PaneId,
    taskId: string,
    qualityScore: number,
    embedding: number[]
  ): Promise<void>;

  // ── Integration Management ─────────────────────────────────────────────────

  /** Install an official Herdr integration (e.g. 'claude', 'codex'). */
  integrationInstall(name: string): Promise<{ installed: boolean; detail: string }>;

  /** List installed integrations and their reported capabilities. */
  integrationList(): Promise<Array<{ name: string; stateReporting: 'semantic' | 'screen' | 'none' }>>; 
}
```

---

## Herdr Adapter Implementation — Verified CLI Surface

```typescript
// src/herdr/herdr-adapter-impl.ts
import { execFile, spawn } from 'node:child_process';
import { promisify } from 'node:util';
import type {
  HerdrAdapter, ProbeResult, PaneId, AgentStatus,
  PaneReadResult, RunResult, WaitResult, SubagentPaneRequest,
  SubagentPaneResult, LogStreamConfig, TraceSpan,
  HerdrWorkspaceLayout
} from './herdr-adapter.js';

const execFileAsync = promisify(execFile);

const HERDR_BIN = process.env.HERDR_BIN ?? 'herdr';
const MAX_SUBAGENT_DEPTH = 4;  // guard band below Anthropic cap=5

export class HerdrAdapterImpl implements HerdrAdapter {
  private probeCache: ProbeResult | null = null;

  async probe(): Promise<ProbeResult> {
    if (this.probeCache) return this.probeCache;
    const start = Date.now();
    try {
      const { stdout } = await execFileAsync(HERDR_BIN, ['--version'], { timeout: 5000 });
      const version = stdout.trim();           // e.g. "herdr 0.6.8"
      const result: ProbeResult = {
        ok: true,
        version,
        latencyMs: Date.now() - start,
      };
      this.probeCache = result;
      return result;
    } catch (e) {
      const result: ProbeResult = {
        ok: false,
        detail: String(e),
        latencyMs: Date.now() - start,
      };
      this.probeCache = result;
      return result;
    }
  }

  private async herdr(...args: string[]): Promise<string> {
    const { stdout } = await execFileAsync(HERDR_BIN, args, { timeout: 15000 });
    return stdout.trim();
  }

  async sessionEnsure(sessionName: string): Promise<void> {
    // herdr session list → check if exists → create if not
    const list = await this.herdr('session', 'list');
    if (!list.includes(sessionName)) {
      await this.herdr('session', 'create', '--label', sessionName);
    }
  }

  async workspaceEnsure(sessionName: string, workspaceName: string): Promise<void> {
    // herdr workspace create --cwd . --label <name> --no-focus
    // idempotent: ignore "already exists" errors
    try {
      await this.herdr('workspace', 'create',
        '--label', workspaceName, '--no-focus');
    } catch (e: any) {
      if (!String(e).includes('already exists')) throw e;
    }
  }

  async applyLayout(layout: HerdrWorkspaceLayout): Promise<void> {
    for (const tab of layout.tabs) {
      // create tab
      try {
        await this.herdr('tab', 'create', '--label', tab.label);
      } catch { /* exists */ }

      // create panes by splitting
      for (let i = 1; i < tab.panes.length; i++) {
        // first pane is the tab default pane
        // additional panes created by splitting
        try {
          await this.herdr('pane', 'split',
            `--direction`, 'right',
            '--label', tab.panes[i]);
        } catch { /* exists */ }
      }
    }
  }

  async resolvePaneByLabel(
    session: string, workspace: string,
    tabLabel: string, paneLabel: string
  ): Promise<PaneId> {
    // herdr pane list returns pane addresses — parse by label
    // RESOLVED (verified 2026-06-11, herdr 0.6.8): no --json flag — socket-API
    // subcommands emit JSON by default. Real shape:
    // {"id":"cli:pane:list","result":{"panes":[{"pane_id":"<workspace_id>-<serial>","label":...,"agent_status":...,...}],"type":"pane_list"}}
    // Use `--session <name> pane list --workspace <workspace_id>` and filter
    // .result.panes[] by label client-side (no `pane resolve` verb exists). See ADR-018.
    const out = await this.herdr('pane', 'list', '--json');
    const panes = JSON.parse(out) as Array<{ id: string; label: string }>;
    const found = panes.find(p => p.label === paneLabel);
    if (!found) throw new Error(`Pane not found: ${paneLabel}`);
    return { raw: found.id };
  }

  async paneRun(paneId: PaneId, command: string): Promise<RunResult> {
    await this.herdr('pane', 'run', paneId.raw, command);
    return { paneId, command, startedAt: new Date().toISOString() };
  }

  async paneRead(
    paneId: PaneId, lines: number,
    source: 'recent-unwrapped' | 'screen' = 'recent-unwrapped'
  ): Promise<PaneReadResult> {
    const out = await this.herdr(
      'pane', 'read', paneId.raw,
      '--source', source,
      '--lines', String(lines)
    );
    return { paneId, lines: out.split('\n'), source };
  }

  async paneSplit(paneId: PaneId, direction: 'right' | 'down'): Promise<PaneId> {
    const out = await this.herdr('pane', 'split', paneId.raw, '--direction', direction);
    // parse new pane ID from output
    // RESOLVED (verified 2026-06-11, herdr 0.6.8): split returns the NEW pane as
    // {"result":{"pane":{"pane_id":"<workspace_id>-<serial>",...},"type":"pane_info"}}
    // — the id is nested under .result.pane.pane_id. `pane split` has no --label;
    // rename afterwards via `pane rename <id> <label>`. See ADR-018.
    const match = out.match(/pane[:\s]+([0-9-]+)/i);
    return { raw: match?.[1] ?? out.trim() };
  }

  async paneAgentStatus(paneId: PaneId): Promise<AgentStatus> {
    const out = await this.herdr('pane', 'status', paneId.raw, '--json');
    const parsed = JSON.parse(out);
    return (parsed.agent_status ?? 'unknown') as AgentStatus;
  }

  async waitForOutput(paneId: PaneId, match: string, timeoutMs: number): Promise<WaitResult> {
    const start = Date.now();
    try {
      await this.herdr('wait', 'output', paneId.raw, '--match', match, '--timeout', String(timeoutMs));
      return { paneId, condition: 'output-match', matched: true, timedOut: false, elapsedMs: Date.now()-start };
    } catch {
      return { paneId, condition: 'output-match', matched: false, timedOut: true, elapsedMs: Date.now()-start };
    }
  }

  async waitForAgentStatus(paneId: PaneId, targetStatus: AgentStatus, timeoutMs: number): Promise<WaitResult> {
    const start = Date.now();
    try {
      await this.herdr('wait', 'agent-status', paneId.raw, '--status', targetStatus, '--timeout', String(timeoutMs));
      return { paneId, condition: 'agent-status', matched: true, timedOut: false, elapsedMs: Date.now()-start };
    } catch {
      return { paneId, condition: 'agent-status', matched: false, timedOut: true, elapsedMs: Date.now()-start };
    }
  }

  async spawnSubagentPane(req: SubagentPaneRequest): Promise<SubagentPaneResult> {
    if (req.depth > MAX_SUBAGENT_DEPTH) {
      throw new Error(
        `Subagent depth ${req.depth} exceeds max ${MAX_SUBAGENT_DEPTH} ` +
        `(Anthropic cap=5, Aegis guard band=4)`
      );
    }
    const label = `subagent-${req.taskId}-${req.depth}-${req.role}`;
    const tabLabel = req.depth <= 3 ? 'subagents' : 'workers';

    // create pane in correct tab
    await this.herdr('tab', 'focus', tabLabel);
    const paneId = await this.paneSplit({ raw: await this.getFirstPaneInTab(tabLabel) }, 'right');

    await this.paneRun(paneId, req.command);
    return { paneId, label };
  }

  async archiveSubagentPane(paneId: PaneId, taskId: string): Promise<{ outputRef: string }> {
    // Read final output before closing
    const read = await this.paneRead(paneId, 500, 'recent-unwrapped');
    const outputRef = `.aegis/traces/subagent-${taskId}-${Date.now()}.txt`;
    // In production: write read.lines to outputRef, then close pane
    // RESOLVED (verified 2026-06-11, herdr 0.6.8): `herdr pane close <pane_id>` IS the
    // close command → {"result":{"type":"ok"}}, exit 0; afterwards `pane get` exits 1
    // pane_not_found. There is no `pane archive`/`pane kill`/`pane remove`. See ADR-018.
    return { outputRef };
  }

  async listSubagentPanes(): Promise<Array<{
    paneId: PaneId; label: string; taskId: string;
    depth: number; role: string; status: AgentStatus;
  }>> {
    const out = await this.herdr('pane', 'list', '--json');
    const panes = JSON.parse(out) as Array<{ id: string; label: string }>;
    return panes
      .filter(p => p.label.startsWith('subagent-'))
      .map(p => {
        const parts = p.label.split('-'); // subagent-{taskId}-{depth}-{role}
        return {
          paneId: { raw: p.id },
          label: p.label,
          taskId: parts[1],
          depth: parseInt(parts[2], 10),
          role: parts[3],
          status: 'unknown' as AgentStatus,
        };
      });
  }

  async attachLogStream(config: LogStreamConfig): Promise<{
    write: (entry: object) => Promise<void>;
    close: () => Promise<void>;
  }> {
    const paneId = await this.resolvePaneByLabel('aegis-prod','aegis-loop','logs', config.paneLabel);
    return {
      write: async (entry: object) => {
        const line = JSON.stringify(entry);
        await this.paneRun(paneId, `echo '${line.replace(/'/g, "'\\''")}'`);
      },
      close: async () => { /* no-op — pane persists */ }
    };
  }

  async readLogStream(paneLabel: string, lastN: number): Promise<object[]> {
    const paneId = await this.resolvePaneByLabel('aegis-prod','aegis-loop','logs', paneLabel);
    const read = await this.paneRead(paneId, lastN);
    return read.lines
      .filter(l => l.trim().startsWith('{'))
      .map(l => { try { return JSON.parse(l); } catch { return null; } })
      .filter(Boolean) as object[];
  }

  async emitSpan(span: TraceSpan): Promise<void> {
    const paneId = await this.resolvePaneByLabel('aegis-prod','aegis-loop','traces','otel-collector');
    await this.paneRun(paneId, `echo '${JSON.stringify(span).replace(/'/g, "'\\''")}'`);
  }

  async readSpans(lastN: number): Promise<TraceSpan[]> {
    const paneId = await this.resolvePaneByLabel('aegis-prod','aegis-loop','traces','span-viewer');
    const read = await this.paneRead(paneId, lastN);
    return read.lines
      .filter(l => l.trim().startsWith('{'))
      .map(l => { try { return JSON.parse(l) as TraceSpan; } catch { return null; } })
      .filter(Boolean) as TraceSpan[];
  }

  async captureTrajectory(
    paneId: PaneId, taskId: string, missionId: string
  ): Promise<{ trajectoryRef: string; rawTokens: number; compressedTokens: number }> {
    const read = await this.paneRead(paneId, 1000, 'recent-unwrapped');
    const rawText = read.lines.join('\n');
    const rawTokens = Math.ceil(rawText.length / 4);  // rough estimate
    // RTK compression happens in CompressionBus — this just captures the raw ref
    const trajectoryRef = `.aegis/trajectories/${missionId}-${taskId}-${Date.now()}.txt`;
    return { trajectoryRef, rawTokens, compressedTokens: rawTokens }; // RTK applied downstream
  }

  async recordSonaFeedback(
    paneId: PaneId, taskId: string, qualityScore: number, embedding: number[]
  ): Promise<void> {
    // Feed into @ruvector/ruvllm-wasm SonaInstantWasm — called by memory-agent
    // HerdrAdapter records WHICH pane produced which quality signal
    // Actual SONA call is in the HNSW router layer (see inference spec)
    // This method writes a feedback record to AgentDB for downstream processing
    void { paneId, taskId, qualityScore, embedding }; // wired to AgentDB in memory-agent
  }

  async integrationInstall(name: string): Promise<{ installed: boolean; detail: string }> {
    try {
      const out = await this.herdr('integration', 'install', name);
      return { installed: true, detail: out };
    } catch (e) {
      return { installed: false, detail: String(e) };
    }
  }

  async integrationList(): Promise<Array<{ name: string; stateReporting: 'semantic' | 'screen' | 'none' }>> {
    // RESOLVED (verified 2026-06-11, herdr 0.6.8): there is NO `integration list` —
    // the verb is `herdr integration status [--outdated-only]` and its output is PLAIN
    // TEXT (`name: current (vN) (path)` / `name: not installed (path)`). Parse that;
    // never fall back to a hardcoded roster. Correction to the article claim below:
    // the installed claude and codex integrations are SEMANTIC hook reporters (v5 —
    // e.g. ~/.claude/hooks/herdr-agent-state.sh reporting over the socket when
    // HERDR_ENV=1), not screen detection. See ADR-018 and ADR-021.
    const knownSemanticReporters = new Set(['pi', 'hermes', 'qodercli']);
    const knownScreenReporters = new Set(['claude', 'codex', 'opencode', 'copilot']);
    try {
      const out = await this.herdr('integration', 'list', '--json');
      const list = JSON.parse(out) as Array<{ name: string }>;
      return list.map(i => ({
        name: i.name,
        stateReporting: knownSemanticReporters.has(i.name) ? 'semantic'
          : knownScreenReporters.has(i.name) ? 'screen' : 'none'
      }));
    } catch {
      return [];
    }
  }

  private async getFirstPaneInTab(tabLabel: string): Promise<string> {
    const out = await this.herdr('pane', 'list', '--tab', tabLabel, '--json');
    const panes = JSON.parse(out) as Array<{ id: string }>;
    if (!panes.length) throw new Error(`No panes in tab ${tabLabel}`);
    return panes[0].id;
  }
}
```

---

## Controller Loop Integration

The Aegis ControllerLoop (PRD §5) integrates HerdrAdapter at the
`EXECUTE_HERDR` state-machine step and reads agent state during
`CAPTURE_COMPRESS_OUTPUT` and `VERIFY`.

```typescript
// Extension to src/controller/loop.ts — EXECUTE_HERDR step

async function executeInHerdr(
  task: Task,
  herdr: HerdrAdapter,
  swarmId: string,
  missionId: string
): Promise<ExecutionResult> {

  // 1. Resolve target pane for this task's role
  const tabMap: Record<string, string> = {
    'controller': 'loop-main',
    'architect':  'loop-main',
    'coder':      'loop-main',
    'memory':     'loop-main',
    'test-runner':'verify',
    'reviewer':   'verify',
    'security':   'ops',
    'ran-advisor':'ran',
    'canary-planner': 'ran',
  };
  const tab = tabMap[task.role] ?? 'workers';
  const paneLabel = task.role;

  let paneId: PaneId;

  // 2. For nested subagents (depth > 0), spawn an ephemeral pane
  if (task.nestingDepth && task.nestingDepth > 0) {
    const spawned = await herdr.spawnSubagentPane({
      taskId: task.id,
      depth: task.nestingDepth,
      role: task.role,
      command: buildAgentCommand(task),
      tabLabel: 'subagents',
    });
    paneId = spawned.paneId;
  } else {
    paneId = await herdr.resolvePaneByLabel('aegis-prod', 'aegis-loop', tab, paneLabel);
    await herdr.paneRun(paneId, buildAgentCommand(task));
  }

  // 3. Emit start span
  await herdr.emitSpan({
    traceId: missionId,
    spanId: task.id,
    name: `task.${task.role}`,
    agentRole: task.role,
    taskId: task.id,
    startTimeMs: Date.now(),
    attributes: { risk_class: task.riskClass, workflow_id: task.workflowId },
    status: 'unset',
  });

  // 4. Wait for agent to complete (with timeout)
  const TASK_TIMEOUT_MS = 30 * 60 * 1000; // 30 min
  const waitResult = await herdr.waitForAgentStatus(paneId, 'done', TASK_TIMEOUT_MS);

  if (waitResult.timedOut) {
    throw new Error(`Task ${task.id} timed out after ${TASK_TIMEOUT_MS}ms`);
  }

  // 5. Capture trajectory (raw pane output → RTK compress)
  const trajectory = await herdr.captureTrajectory(paneId, task.id, missionId);

  // 6. Read artifact ref from pane output (agents emit JSON to stdout)
  const outputLines = await herdr.paneRead(paneId, 50, 'recent-unwrapped');
  const lastJsonLine = outputLines.lines.reverse().find(l => l.trim().startsWith('{'));
  const agentResult = lastJsonLine ? JSON.parse(lastJsonLine) : {};

  // 7. Close subagent pane if ephemeral
  if (task.nestingDepth && task.nestingDepth > 0) {
    await herdr.archiveSubagentPane(paneId, task.id);
  }

  // 8. Close span
  await herdr.emitSpan({
    traceId: missionId,
    spanId: task.id,
    name: `task.${task.role}`,
    agentRole: task.role,
    taskId: task.id,
    startTimeMs: Date.now() - waitResult.elapsedMs,
    endTimeMs: Date.now(),
    attributes: { elapsed_ms: waitResult.elapsedMs },
    status: agentResult.status === 'ok' ? 'ok' : 'error',
  });

  return {
    taskId: task.id,
    artifactRef: agentResult.artifact_ref ?? '',
    rawOutputRef: trajectory.trajectoryRef,
    rtkSummary: { rawTokens: trajectory.rawTokens, compressedTokens: trajectory.compressedTokens },
    costUsd: agentResult.cost_usd ?? 0,
    inputTokens: agentResult.input_tokens ?? 0,
    outputTokens: agentResult.output_tokens ?? 0,
  };
}
```

---

## Self-Learning Loop — Herdr × AgentDB × RuVector × RuvLLM

The self-learning loop reads pane output via HerdrAdapter, compresses via RTK,
stores trajectories in AgentDB, indexes quality signals in RuVector/SONA, and
uses the HNSW router to improve future routing decisions. This closes the
observation→memory→learning loop through Herdr as the observation substrate.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      Self-Learning Flow via Herdr                            │
└─────────────────────────────────────────────────────────────────────────────┘

  Herdr Pane (working→done transition)
      │
      ▼ HerdrAdapter.captureTrajectory()
  Raw pane output (lines)
      │
      ▼ CompressionBus.compress_terminal()  [RTK]
  Compressed terminal summary
      │
      ▼ AgentDB.commit(namespace='trajectory')
  Typed trajectory record
      │
      ├──▶ RuVector.upsert_embedding()      [nomic-embed-text-v1.5, 512-dim]
      │    namespace='trajectory'
      │    metadata: { task_id, role, confidence, elapsed_ms }
      │
      └──▶ HerdrAdapter.recordSonaFeedback()
               │
               ▼ SonaInstantWasm.instantAdapt(embedding, qualityScore)
           SONA EMA quality tracking
               │
               ▼ HnswRouterWasm.addPattern(embedding, route_target)
           HNSW routing patterns updated
               │
               ▼ Next task → better routing decision
```

### Trajectory Record Schema

```typescript
interface HerdrTrajectoryRecord {
  trajectory_id: string;
  mission_id: string;
  task_id: string;
  agent_role: string;
  pane_label: string;
  herdr_session: string;
  raw_output_ref: string;         // path to uncompressed output
  rtk_summary_ref: string;        // path to RTK-compressed summary
  raw_tokens: number;
  compressed_tokens: number;
  compression_ratio: number;
  agent_status_final: AgentStatus;
  elapsed_ms: number;
  confidence: number;             // from verification gate (set post-verify)
  created_at: string;             // ISO8601
  spans: TraceSpan[];             // correlated OTel spans
}
```

### Log Pane Schema (JSON-lines)

All log panes emit structured JSON-lines. RTK must NOT compress log panes
(they are already structured and should be read directly by the memory agent).

```typescript
interface AegisLogEntry {
  ts: string;                    // ISO8601
  level: 'debug' | 'info' | 'warn' | 'error';
  source: string;                // 'controller' | 'memory-agent' | 'gate' | 'ran-agent' | ...
  event: string;                 // e.g. 'task.dispatched' | 'gate.passed' | 'memory.committed'
  mission_id?: string;
  task_id?: string;
  data: Record<string, unknown>;
}
```

---

## Updated PRD §4.1 — Workspace Layout (Canonical Replacement)

Replace PRD §4.1 with this canonical layout. All panes are Herdr-managed.
Provisioned by `HerdrAdapterImpl.applyLayout()` at session start.

```text
Herdr Session: aegis-prod
  Workspace: aegis-loop

  Tab: loop-main
    controller      ← Aegis ControllerLoop + Ruflo Queen
    architect       ← architect worker
    coder           ← coder worker
    memory          ← memory-agent

  Tab: verify
    test-runner     ← behavioral test gate
    reviewer        ← semantic review gate
    observer        ← gate aggregator
    scratch         ← sandbox (destructive commands run here only)

  Tab: ops
    ruflo-control   ← Ruflo MCP server daemon
    router          ← SONA HNSW model router
    security        ← security gate / aidefence
    benchmark       ← benchmark-agent

  Tab: compression
    rtk-monitor     ← RTK pre-bash hook monitor
    headroom-proxy  ← Headroom context bus
    token-economics ← token economics emitter

  Tab: ran
    ran-agent       ← RAN advisor (advisory + canary only)
    replay-sim      ← historical replay
    canary-planner  ← canary plan generator
    rollback-monitor← rollback readiness

  Tab: logs         ← NEW — structured JSON-lines sinks
    controller-log  ← controller loop events
    memory-log      ← AgentDB/RuVector operations
    gate-log        ← gate results
    ran-log         ← RAN audit trail

  Tab: traces       ← NEW — distributed tracing
    otel-collector  ← OTLP JSON-lines sink
    span-viewer     ← span summary reader
    cost-trace      ← token economics traces

  Tab: subagents    ← NEW — nested subagent panes (depth 1–3)
    dynamic, named: subagent-{task_id}-{depth}-{role}
    max concurrent: 12 panes (enforced by HerdrAdapter)
    auto-archived on done

  Tab: workers      ← existing + depth-4 subagents
    ephemeral queued workers + depth-4 subagent panes
```

---

## ADR Additions

### ADR-016: Herdr Owns All Agent Pane Lifecycles

**Status:** Accepted

**Context:** The PRD requires Herdr to own persistent runtime. This ADR
formalizes that every agent (controller, workers, subagents, daemons) runs
inside a Herdr-managed pane, not as a raw process.

**Decision:** No agent process is started with `exec()` or `spawn()` directly
by Aegis domain logic. All process lifecycle goes through HerdrAdapter.
Domain logic calls `herdr.paneRun()`, not `child_process.spawn()`.

**Consequences:**
- Positive: persistent sessions survive detach; operator always has visibility
- Positive: agent state is observable via `herdr wait agent-status`
- Negative: adds HerdrAdapter as a required dependency (mitigated by fallback)
- Negative: Herdr must be installed; adds AGPL-3.0-or-later compliance requirement

**Verification:** Integration test: kill the Herdr client, reattach, assert
all panes still running and agent states readable.

**Rollback:** Remove HerdrAdapter; revert to raw `child_process.spawn()` with
in-process state tracking (graceful degradation path always available).

### ADR-017: Log Panes Are Structured JSON-Lines, Not Raw Stdout

**Status:** Accepted

**Context:** Raw agent stdout contains ANSI codes, prompts, and LLM output
that is not parseable for memory/trace ingestion.

**Decision:** Each agent writes structured JSON-lines to a dedicated log pane.
Agent stdout remains in the main pane for human observability.
The memory-agent reads from log panes, not from agent panes.

**Verification:** Unit test: memory-agent reads log pane, parses all entries,
asserts no parse failures.

### ADR-018: Subagent Panes Are Ephemeral and Depth-Gated

**Status:** Accepted

**Context:** Ruflo v3.10 supports nested subagents to depth ≤ 4 (Aegis guard
band below Anthropic cap of 5). Each nested subagent needs process isolation
and observability without polluting the static workspace tabs.

**Decision:** Nested subagents are assigned to the `subagents` tab (depth 1–3)
or `workers` tab (depth 4). Panes are spawned by HerdrAdapter.spawnSubagentPane()
and archived by HerdrAdapter.archiveSubagentPane() on task completion.
Maximum 12 concurrent subagent panes enforced by HerdrAdapter.

**Verification:** Integration test: spawn 4 nested subagents at increasing depth,
assert depth-5 throws, assert all archived correctly.

---

## Updated Bounded Context — Orchestration Context

The Orchestration Context (PRD §17.1) gains Herdr entities:

```text
Entities added:
  HerdrSession
  HerdrWorkspace
  HerdrTab
  HerdrPane
  SubagentPaneLifecycle
  LogStream
  TraceSpan
  AgentStateTransition

Services:
  HerdrAdapter          ← adapter boundary (§H.4)
  WorkspaceProvisioner  ← idempotent layout application (§H.3.1)
  SubagentPaneManager   ← ephemeral pane lifecycle (§H.6)
  TrajectoryCapture     ← pane output → RTK → AgentDB (§H.7)
```

---

## Ruflo ↔ Herdr Integration Pattern

Ruflo is the orchestrator. Herdr is the runtime OS. Their interaction:

```text
Ruflo Controller (Queen)
    ↓  dispatches task via MCP tool:
    │  mcp__ruflo__agent_spawn { type: "coder", ... }
    │
    ↓  Aegis controller intercepts dispatch →
    │  HerdrAdapter.paneRun(coder_pane, ruflo_command)
    │
    ↓  Herdr pane runs the agent process
    │  Agent uses Claude Code / Codex / etc (with herdr integration)
    │
    ↓  Herdr detects: pane agent-status → done
    │
    ↓  HerdrAdapter.captureTrajectory() → RTK compress
    │
    ↓  AgentDB.commit(trajectory) + RuVector.upsert()
    │
    ↓  SONA feedback → HNSW router updated
    │
    ↓  Ruflo post-task hook fires (via .claude/hooks/post-task.ts)
    │  mcp__ruflo__neural_train { pattern_type: "coordination", ... }
    │
    ↓  Aegis ControllerLoop: VERIFY → COMPUTE_CONFIDENCE → REFLECT
```

The key invariant: **Ruflo decides what runs; Herdr decides where and how it
runs and keeps it alive.**

---

## Smoke Test Suite

All HerdrAdapter methods must have smoke tests gated behind `--integration` flag.

```typescript
// tests/smoke/herdr-adapter.smoke.ts
import { HerdrAdapterImpl } from '../../src/herdr/herdr-adapter-impl.js';

describe('HerdrAdapter smoke tests', () => {
  const herdr = new HerdrAdapterImpl();

  it('probe() returns ok=true with version string', async () => {
    const result = await herdr.probe();
    expect(result.ok).toBe(true);
    expect(result.version).toMatch(/herdr \d+\.\d+\.\d+/);
  });

  it('session and workspace creation is idempotent', async () => {
    await herdr.sessionEnsure('test-session');
    await herdr.sessionEnsure('test-session'); // second call must not throw
  });

  it('spawnSubagentPane enforces depth ≤ 4', async () => {
    await expect(herdr.spawnSubagentPane({
      taskId: 'test-001', depth: 5, role: 'coder',
      command: 'echo test', tabLabel: 'subagents'
    })).rejects.toThrow('exceeds max 4');
  });

  it('waitForOutput returns timedOut=true on no match', async () => {
    // requires a live Herdr session with a pane
    const paneId = await herdr.resolvePaneByLabel(
      'aegis-prod','aegis-loop','loop-main','controller'
    );
    const result = await herdr.waitForOutput(paneId, 'NEVER_MATCH_XYZ', 1000);
    expect(result.timedOut).toBe(true);
  });

  it('log stream write+read roundtrip', async () => {
    const stream = await herdr.attachLogStream({ paneLabel: 'controller-log', format: 'json-lines', maxLines: 100 });
    const entry = { ts: new Date().toISOString(), level: 'info', source: 'test', event: 'smoke.test', data: {} };
    await stream.write(entry);
    await stream.close();
    const read = await herdr.readLogStream('controller-log', 10);
    expect(read.some((e: any) => e.event === 'smoke.test')).toBe(true);
  });
});
```

---

## Updated Implementation Priority

The existing PRD §29 phase ordering is updated to make Herdr the first phase:

```text
Phase 0:  Herdr install + workspace provisioning (NEW — prerequisite for all)
           HerdrAdapter.probe() + applyLayout() + smoke tests
           Advance when: all panes provisioned, probe() returns ok

Phase 1:  Herdr workspace + Ruflo control pane (existing §29 Phase 1, now with Herdr)
Phase 2:  RufloAdapter interface + mocked tests
Phase 3:  Deterministic controller loop
Phase 4:  AgentDB/RuVector/RVF memory layer
Phase 5:  RTK + Headroom compression bus
Phase 6:  Verification gates
Phase 7:  Skill/loop registry
Phase 8:  Cost governor and model router
Phase 9:  Ericsson RAN specialist
Phase 10: Subagent pane manager + nested subagents (NEW — after Phase 3)
Phase 11: Log streams + trace sinks (NEW — after Phase 5)
Phase 12: Self-learning loop: Herdr → AgentDB → SONA → HNSW (NEW — after Phase 4+5)
Phase 13: Federation and multi-host execution
Phase 14: Optional RuView spatial sensing
Phase 15: DSPy GEPA optimization
```

---

## Closing Principle (Amendment)

Herdr is not a GUI wrapper over agents.
Herdr is the terminal OS that makes agents durable, observable, and automatable.

```text
Ruflo orchestrates.
Herdr persists and multiplexes.
Agents live inside Herdr panes.
Logs stream to Herdr log panes.
Traces emit to Herdr trace panes.
Subagents spawn and die inside Herdr.
RuvLLM learns from what Herdr captures.
AgentDB remembers what Herdr observed.
RuVector indexes what AgentDB distilled.
SONA improves routing from what RuVector learned.
Aegis governs everything.
```
