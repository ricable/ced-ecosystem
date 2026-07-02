# ELEX RAN WASM — Multi-Host Deployment PRD

Source: consolidated from ruvllm-wasmelex.md. Deployment PRD for the ELEX RAN browser/multi-host agent — see /Users/cedric/work/dev/ced-ecosystem/PRD/README.md for the full document map. Related: /Users/cedric/work/dev/ced-ecosystem/adr/ADR-016-docling-local-ingestion-adapter.md and DDR-014/DDR-015 under /Users/cedric/work/dev/ced-ecosystem/ddd/.

---

## 1. Context

The `elex-ran-features` skill holds a 28,979-record Ericsson RAN knowledge base (12 domains, 384-d
all-MiniLM-L6-v2 embeddings). The goal is a single shared knowledge core deployable across nine
host environments — from a zero-server browser tab to GitHub Actions CI — without duplicating the
skill logic.

**Scope: `ricable/ruv-ecosystem` only.**

---

## 2. Host Matrix

| Host | What ships | Notes |
|---|---|---|
| Claude Code | MCP server + hooks + 3-scope settings | Richest surface; Ruflo-native |
| OpenAI Codex | MCP via `~/.codex/config.toml` | TOML, no hooks |
| pi.dev | Pi extension via `pi.registerTool()` | No MCP by design |
| Hermes | MCP runtime + `<think>` scrubbing | Per Hermes issue #741; also `config/elex-ran-hermes.json` |
| OpenClaw | `~/.openclaw/openclaw.json` + workspace skills | Personal-AI gateway |
| RVM | Bare-metal microhypervisor + capability tokens | Hardware isolation for untrusted peers |
| GitHub Copilot | MCP via `.vscode/mcp.json` | VSCode 1.99+ |
| OpenCode | MCP via `.opencode/opencode.json` | sst/opencode TUI |
| GitHub Actions | `.github/workflows/` + composite `action.yml` | Non-interactive CI/CD; default-deny via `permissions:` |

Plus the **browser WASM** deployment (no host agent required; standalone `dist/index.html`).

---

## 3. What Already Exists (reuse, don't recreate)

| Asset | Location | Role |
|---|---|---|
| 28,979-record manifest | `.claude/skills/elex-ran-features/data/rvf_memory.json` | Domain record counts, 384-d MiniLM confirmed |
| Domain graph edges | `.claude/skills/elex-ran-features/references/dependencies-extended.json` | Feature→feature dependencies (220 edges) |
| TypeScript HNSW | `.claude/skills/elex-ran-features/hnsw/hnsw-index.ts` | O(log n) cosine, M=32 — match in Rust at M=16 |
| RVF writer | `crates/rano-rvf-writer/` | RVF serialisation pattern |
| Embedder types | `crates/rano-ran-embed/` | `cdylib` + `wasm-bindgen` Cargo.toml to copy |
| RAN harness generator | `config/ran-harness-generator.json` | Existing use-case catalogue to extend |
| RAN agent harness | `config/ran-agent-harness.json` | Herdr command templates to extend |
| RVF ingest script | `.claude/skills/elex-ran-features/scripts/build_rvf_memory.py` | Builds `rvf_vectors.jsonl` (~4 min) |

### 3.1 Prerequisite: build data files

```bash
source .venv/bin/activate && \
  python3 .claude/skills/elex-ran-features/scripts/build_rvf_memory.py
```

---

## 4. Architecture

```
Shared knowledge core
├── elex-ran.rvf        4-zone gzip binary (28,979 records, 384-d)
└── src/mcp/            MCP server  ←── 7 of 9 hosts consume via MCP
    elex-ran-server.ts

Host surfaces
├── Browser WASM        wasm/elex-ran-demo/dist/index.html  (zero-server, WebGPU)
│   ├── WASM core       crates/elex-ran-wasm → wasm32-unknown-unknown
│   ├── ONNX encoder    @xenova/transformers (Web Worker)
│   └── LLM             @mlc-ai/web-llm  Phi-3-mini WebGPU
│
├── Claude Code         .claude/settings.json  MCP entry + hooks
├── OpenAI Codex        ~/.codex/config.toml   MCP entry
├── pi.dev              src/pi/elex-ran-pi.js  pi.registerTool()
├── Hermes              config/elex-ran-hermes.json  MCP + <think> scrub
├── OpenClaw            ~/.openclaw/openclaw.json  MCP + workspace skills
├── RVM                 rvm/elex-ran-cap.toml  capability token definition
├── GitHub Copilot      .vscode/mcp.json
├── OpenCode            .opencode/opencode.json
└── GitHub Actions      .github/workflows/elex-ran.yml  composite action
```

### 4.1 MCP tool surface (shared by 7 hosts)

```typescript
// src/mcp/elex-ran-server.ts
tools:
  elex_ran_search(query: string, domain?: string, k?: number) → SearchResult[]
  elex_ran_feature(faj: string) → Feature
  elex_ran_parameters(feature_id: string) → Parameter[]
  elex_ran_counters(feature_id: string) → Counter[]
  elex_ran_dependencies(feature_id: string) → Dependency[]
  elex_ran_cmedit(feature_id: string, cell_id: string) → CMEditScript
```

---

## 5. File Structure

```
src/mcp/
  elex-ran-server.ts       # MCP server — shared by 7 hosts
  package.json             # @modelcontextprotocol/sdk ^1.12 (run: cd src/mcp && npm install)
  tools/
    search.ts
    feature.ts
    cmedit.ts

src/pi/
  elex-ran-pi.js           # pi.registerTool() adapter (no MCP)

crates/elex-ran-wasm/      # Browser WASM core
  Cargo.toml
  src/
    lib.rs
    zone_router.rs
    hnsw_search.rs
    rvf_loader.rs

wasm/elex-ran-demo/        # Browser app
  index.html
  package.json             # vite + vite-plugin-wasm + @xenova/transformers + @mlc-ai/web-llm
  src/
    main.js
    embedder-worker.js
    webllm-bridge.js
  skill/
    prompt-template.txt
    domain-graph.json
  scripts/
    export_rvf.sh
    pack_rvf.mjs
  public/                  # elex-ran.rvf (gitignored)

config/
  elex-ran-hermes.json     # Hermes harness config (extends ran-agent-harness.json pattern)

rvm/
  elex-ran-cap.toml        # RVM capability token definition

.vscode/
  mcp.json                 # GitHub Copilot MCP (VSCode 1.99+)

.opencode/
  opencode.json            # OpenCode MCP

.github/workflows/
  elex-ran.yml             # GitHub Actions CI
  elex-ran-action/
    action.yml             # composite action
```

---

## 6. Implementation Steps

### Step 0 · Data prerequisite (once)

```bash
source .venv/bin/activate && \
  python3 .claude/skills/elex-ran-features/scripts/build_rvf_memory.py
bash wasm/elex-ran-demo/scripts/export_rvf.sh
```

### Step 1 · MCP server (`src/mcp/elex-ran-server.ts`)

Node.js MCP server backed by the Python search scripts (subprocess) or the HNSW TypeScript
layer (`hnsw/hnsw-index.ts`). Exposes six tools listed above.

**Prerequisites (once):**

```bash
cd src/mcp && npm install    # installs @modelcontextprotocol/sdk ^1.12
cd ../..
```

> **SDK compat note:** `Tool` is type-only in `@modelcontextprotocol/sdk` ≥1.13. The server
> must use `import type { Tool }` (not a plain value import). This is already applied in
> `src/mcp/elex-ran-server.ts`.

Start: `node --experimental-strip-types src/mcp/elex-ran-server.ts`
(Node ≥22 strips TypeScript natively — `ts-node` is not required.)

### Step 2 · Browser WASM (`crates/elex-ran-wasm/` + `wasm/elex-ran-demo/`)

Rust/WASM core: `wasm32-unknown-unknown`, `wasm-bindgen`, zone router + HNSW + RVF loader.

```bash
cargo build --package elex-ran-wasm --target wasm32-unknown-unknown --release
wasm-bindgen target/wasm32-unknown-unknown/release/elex_ran_wasm.wasm \
  --out-dir wasm/elex-ran-demo/src/wasm --target web
cd wasm/elex-ran-demo && npm install && npm run build
```

Browser stack: `@xenova/transformers` (ONNX encoder, Web Worker) + `@mlc-ai/web-llm` (Phi-3-mini,
WebGPU). Fallback: show retrieved records if WebGPU unavailable.

### Step 3 · Claude Code (`.claude/settings.json`)

```json
{
  "mcpServers": {
    "elex-ran": {
      "command": "node",
      "args": ["--experimental-strip-types", "src/mcp/elex-ran-server.ts"],
      "cwd": "/path/to/ruv-ecosystem"
    }
  }
}
```

Hooks: `post-edit` on `*.ts` files in `src/mcp/` → restart MCP server.

### Step 4 · OpenAI Codex (`~/.codex/config.toml`)

```toml
[[mcp_servers]]
name    = "elex-ran"
command = "node --experimental-strip-types /path/to/ruv-ecosystem/src/mcp/elex-ran-server.ts"
```

### Step 5 · pi.dev (`src/pi/elex-ran-pi.js`)

```js
import { elexRanSearch, elexRanCmedit } from '../mcp/tools/index.js';
export function register(pi) {
  pi.registerTool('elex_ran_search', {
    description: 'Search Ericsson RAN knowledge base',
    parameters: { query: 'string', domain: 'string?' },
    handler: ({ query, domain }) => elexRanSearch(query, domain),
  });
  pi.registerTool('elex_ran_cmedit', {
    description: 'Generate CMEdit script for a RAN feature',
    parameters: { feature_id: 'string', cell_id: 'string' },
    handler: ({ feature_id, cell_id }) => elexRanCmedit(feature_id, cell_id),
  });
}
```

### Step 6 · Hermes (`config/elex-ran-hermes.json`)

Extends `config/ran-agent-harness.json` pattern. Adds `<think>` scrubbing per Hermes issue #741.

```json
{
  "name": "elex-ran-bot",
  "mode": "advisory",
  "prompt": ".claude/skills/elex-ran-features/SKILL.md",
  "mcp": {
    "elex-ran": {
      "command": "node",
      "args": ["--experimental-strip-types", "src/mcp/elex-ran-server.ts"]
    }
  },
  "scrubThink": true,
  "herdr": {
    "session": "elex-ran",
    "agent": "elex-ran-specialist"
  }
}
```

**Run via config (existing flow):**

```bash
npx @metaharness/hermes elex-ran-bot --config config/elex-ran-hermes.json
```

**Run via agent-harness-generator USERGUIDE scaffold flow:**

```bash
# Scaffold the harness (any directory — the harness is self-contained)
npx @metaharness/hermes elex-ran-harness
cd elex-ran-harness

# Install kernel + Hermes host adapter
npm install

# Verify (all 4 checks must pass)
node ./bin/cli.js doctor
# → PASS kernel loads
# → PASS kernel reports a version
# → PASS kernel backend is native|wasm|js
# → PASS host adapter has a name
# → elex-ran-harness: all checks passed (kernel 0.1.0, wasm backend, host hermes)

# Boot the agent
node ./bin/cli.js init
# → elex-ran-harness — kernel 0.1.0 (wasm)
# → Host adapter: hermes

# The harness cli-config.yaml (auto-generated) picks up ANTHROPIC_API_KEY / OPENROUTER_API_KEY
# for the full Hermes agent loop:
hermes   # if the Hermes CLI is installed
```

> **Hermes bin note:** `elex-ran-harness doctor` requires the harness to be on `$PATH`. Use
> `node ./bin/cli.js doctor` from the harness directory, or `cd <harness> && npm link` once to
> register `elex-ran-harness` globally.

### Step 7 · OpenClaw (`~/.openclaw/openclaw.json`)

```json
{
  "mcpServers": {
    "elex-ran": {
      "command": "node",
      "args": ["--experimental-strip-types", "/path/to/src/mcp/elex-ran-server.ts"]
    }
  },
  "workspaceSkills": [".claude/skills/elex-ran-features/SKILL.md"]
}
```

### Step 8 · RVM (`rvm/elex-ran-cap.toml`)

```toml
[capability]
name        = "elex-ran"
description = "Read-only access to Ericsson RAN knowledge base"
access      = ["elex_ran_search", "elex_ran_feature", "elex_ran_parameters",
               "elex_ran_counters", "elex_ran_dependencies"]
deny        = ["elex_ran_cmedit"]   # write-capable tool denied to untrusted peers
```

Grant: `rvm grant --capability rvm/elex-ran-cap.toml --peer <peer-id>`

### Step 9 · GitHub Copilot (`.vscode/mcp.json`)

```json
{
  "servers": {
    "elex-ran": {
      "type": "stdio",
      "command": "node",
      "args": ["--experimental-strip-types", "${workspaceFolder}/src/mcp/elex-ran-server.ts"]
    }
  }
}
```

Requires VSCode 1.99+ and the GitHub Copilot Chat extension.

### Step 10 · OpenCode (`.opencode/opencode.json`)

```json
{
  "mcp": {
    "elex-ran": {
      "command": "node",
      "args": ["--experimental-strip-types", "src/mcp/elex-ran-server.ts"]
    }
  }
}
```

### Step 11 · GitHub Actions (`.github/workflows/elex-ran.yml`)

```yaml
name: elex-ran
on: [workflow_dispatch, pull_request]
permissions:
  contents: read      # default-deny all other permissions
jobs:
  query:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: ./.github/workflows/elex-ran-action
        with:
          query: ${{ inputs.query || 'IFLB load balancing threshold' }}
```

`elex-ran-action/action.yml` (composite):
```yaml
inputs:
  query: { required: true }
runs:
  using: composite
  steps:
    - run: pip install -q -r .claude/skills/elex-ran-features/requirements.txt
      shell: bash
    - run: python3 .claude/skills/elex-ran-features/scripts/search.py "${{ inputs.query }}"
      shell: bash
```

---

## 7. Workspace Change

`Cargo.toml` (root):
```toml
members = [
  "crates/rano-ran-embed",
  "crates/rano-rvf-writer",
  "crates/rano-gnn-topology",
  "crates/elex-ran-wasm",    # add
]
```

---

## 8. Verification

```bash
# MCP server — start (Node 26 native TS, no ts-node)
cd src/mcp && npm install && cd ../..   # first time only
node --experimental-strip-types src/mcp/elex-ran-server.ts &
npx @modelcontextprotocol/cli call elex_ran_search '{"query":"IFLB"}'

# Python search layer (direct, no MCP server needed)
python3 .claude/skills/elex-ran-features/scripts/search.py IFLB --full-chain
python3 .claude/skills/elex-ran-features/scripts/search.py IFLB --cmedit
python3 .claude/skills/elex-ran-features/scripts/search.py --cognitive "RACH failure causes"

# Hermes harness (agent-harness-generator USERGUIDE flow)
npx @metaharness/hermes elex-ran-harness && cd elex-ran-harness
npm install
node ./bin/cli.js doctor   # all 4 checks must pass
node ./bin/cli.js init     # kernel 0.1.0 (wasm), host hermes

# Hermes via config
npx @metaharness/hermes elex-ran-bot --config config/elex-ran-hermes.json \
  --query "RACH access failure causes"

# Browser WASM
npm run preview --prefix wasm/elex-ran-demo
# → "IFLB load balancing threshold" → Zone A, 5 results, Phi-3-mini streams CMEdit

# GitHub Actions
gh workflow run elex-ran.yml --field query="IFLB parameters"

# RVM
rvm verify --capability rvm/elex-ran-cap.toml --tool elex_ran_cmedit  # must deny
rvm verify --capability rvm/elex-ran-cap.toml --tool elex_ran_search   # must allow
```
