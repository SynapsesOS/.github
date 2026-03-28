# Synapses OS — Code Intelligence for AI Agents

**Multi-agent AI development needs shared state. We build the coordination and memory layer.**

## The Problem

When multiple AI agents work on the same codebase:

- **No shared context** — agents duplicate work, one doesn't know what another did
- **Conflicting changes** — overlapping modifications cause merge conflicts
- **No learning** — agents can't learn from past failures across sessions
- **Grep is dumb** — string search finds false positives; agents need graph queries

## Our Solution

**Synapses** is a single-binary code intelligence server for AI agents:

```
┌──────────────────────────────────────────────────────────┐
│  Your IDE (Claude Code, Cursor, VS Code, Windsurf, Zed)  │
└────────────────────┬─────────────────────────────────────┘
                     │ MCP Protocol (stdio / HTTP)
                     ▼
┌──────────────────────────────────────────────────────────┐
│  synapses daemon (:11435)                                │
│  ├─ Graph engine (nodes: functions, structs, files)      │
│  ├─ 12 MCP tools (context, graph, tasks, memory, rules)  │
│  ├─ Episodic memory (past decisions, failures)           │
│  ├─ Agent message bus (broadcast work status)            │
│  ├─ Vector embeddings (nomic-embed-text-v1.5 ONNX)      │
│  ├─ Web cache & doc lookup (in-process)                  │
│  ├─ Optional brain LLM (Ollama, in-process)              │
│  └─ SQLite persistence (multi-session state)             │
└──────────────────────────────────────────────────────────┘
         Single binary · Local-first · 49 languages
```

---

## Key Features

- **Structured Code Understanding** — Graph of functions, structs, calls, dependencies (not grep)
- **Multi-Agent Coordination** — Message bus + work claims prevent conflicting changes
- **Episodic Memory** — Agents learn: "I tried this pattern last week and it failed"
- **Vector Embeddings** — Built-in nomic-embed-text-v1.5 ONNX model, zero external dependencies
- **Optional Brain LLM** — In-process Ollama integration for summaries and ADRs
- **No Code Leaves Your Machine** — All processing local, no cloud
- **49 Languages** — Go, TypeScript, Python, Java, Rust, C++, and 30+ more
- **Single Binary** — One MCP server, works with any IDE. Pre-built for macOS and Linux.
- **Desktop App** — Visual dashboard for project management, onboarding, and brain setup
- **AI-Native** — Designed specifically for agent workflows (not human IDE features)

---

## Quick Start

### 1. Install

**Homebrew (recommended):**
```bash
brew tap SynapsesOS/tap
brew install synapses
```

**Or direct binary** from [GitHub Releases](https://github.com/SynapsesOS/synapses/releases/latest). **Or from source:**
```bash
curl -fsSL https://raw.githubusercontent.com/SynapsesOS/synapses/main/install.sh | sh
```

**Desktop app:** Download from [synapses-app releases](https://github.com/SynapsesOS/synapses-app/releases/latest) — includes the daemon binary, visual dashboard, and guided onboarding.

### 2. Initialize Your Project

```bash
cd /path/to/your/codebase
synapses init
```

That's it — two commands. The `init` wizard handles everything:

| Step | What it does |
|------|-------------|
| **[1/4] Project Setup** | Detects git, creates `synapses.json` with sensible defaults |
| **[2/4] Indexing** | Parses your codebase and builds the code graph (49 languages) |
| **[3/4] Starting Engine** | Starts the singleton daemon and registers your project |
| **[4/4] Connect Agents** | Auto-detects installed AI agents and writes their MCP configs |

Auto-detects: Claude Code, Cursor, VS Code, Windsurf, Zed, Antigravity.

**Non-interactive mode:** `synapses init --yes --agents claude,cursor`

### 3. Uninstall (full control)

```bash
synapses uninstall                # Remove from current project
synapses uninstall --global       # Remove everything from your system
synapses uninstall --global --yes # Non-interactive full cleanup
```

---

## Repositories

| Project | Language | Purpose | Status |
|---------|----------|---------|--------|
| **[synapses](https://github.com/SynapsesOS/synapses)** | Go | Graph engine + MCP server + brain + embeddings | Active |
| **[synapses-app](https://github.com/SynapsesOS/synapses-app)** | Rust + TypeScript | Desktop app (Tauri) — visual dashboard + bundled daemon | Active |

---

## 12 MCP Tools

Each tool is a consolidated interface handling multiple sub-actions:

| Tool | What it does |
|------|-------------|
| `session_init` | Session bootstrap — returns tasks, project identity, working state, events |
| `get_context` | Code graph queries — intent-based context, BFS ego-subgraph, entity lookup, call chains, repo map, tool discovery |
| `validate` | Architecture enforcement — pre-write plan validation and post-write verification |
| `get_file_context` | All entities in a file ordered by line number |
| `search` | Keyword + FTS5 BM25 full-text search with CamelCase splitting |
| `annotate` | Attach notes to entities, persist web findings, record quality gaps |
| `get_impact` | Blast-radius analysis — reverse-BFS with confidence tiers |
| `tasks` | Plan creation, task lifecycle, session state for cross-session resumption |
| `end_session` | Persist session knowledge as structured memories |
| `rules` | Dynamic architectural rules, violations, and Architectural Decision Records |
| `lookup_docs` | Cached package documentation and URL content |
| `memory` | Episodic memory (remember/recall), agent coordination, message bus |

Plus **8 MCP resources** for live data subscriptions (active-context, violations, repo-map, analytics, etc.).

---

## Minimal Example: Multi-Agent Workflow

```
Session 1 (Frontend Agent):
  agent_id = "frontend-claude"
  session_init()
  → returns: "backend-claude modified /api/auth.go 2h ago"

  Ah, I need to update TypeScript types to match the new auth API
  remember(decision="Updated LoginRequest shape to include mfa_enabled",
           outcome="success")

Session 2 (Backend Agent):
  agent_id = "backend-claude"
  get_messages(agent_id="backend-claude")
  → returns: frontend's memory + recent changes

  Backend can now safely modify /api/checkout.go knowing frontend is aware
```

---

## Architecture Principles

- **No Cloud** — All processing local
- **Fail-Silent** — Brain down? Graph queries still work. No panics.
- **Single Binary** — One MCP server, works with any IDE
- **SQLite-Only** — Pure Go, no C dependencies at runtime
- **Incremental** — File watcher re-parses only changed files
- **AI-Native** — Designed for agents, not humans

---

## Privacy

**All your code stays on your machine.**

- Graph indexed locally in `~/.synapses/`
- Built-in embeddings (ONNX), no external service needed
- Optional brain LLM runs via Ollama on localhost
- No telemetry, no tracking, no cloud uploads

---

## Getting Started

- **[synapses](https://github.com/SynapsesOS/synapses)** — CLI tool, full docs, configuration reference
- **[synapses-app](https://github.com/SynapsesOS/synapses-app)** — Desktop app with visual dashboard

---

## Community

- **Issues**: [github.com/SynapsesOS/synapses/issues](https://github.com/SynapsesOS/synapses/issues)
- **Discussions**: [github.com/SynapsesOS/synapses/discussions](https://github.com/SynapsesOS/synapses/discussions)
- **Security**: Report privately to security@synapsesos.dev

---

## Contributing

We welcome pull requests and issues! See [CONTRIBUTING.md](https://github.com/SynapsesOS/synapses/blob/main/CONTRIBUTING.md) for setup instructions.

---

## License

**MIT Licensed.**

---

[Read the Docs](https://github.com/SynapsesOS/synapses) · [Get Started](https://github.com/SynapsesOS/synapses#quick-start) · [Discuss](https://github.com/orgs/SynapsesOS/discussions)
