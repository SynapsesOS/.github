# 🧠 Synapses OS — Code Intelligence for AI Agents

**Multi-agent AI development needs shared state. We build the coordination and memory layer.**

## The Problem

When multiple AI agents work on the same codebase:

- 🔄 **No shared context** — agents duplicate work, one doesn't know what another did
- ⚠️ **Conflicting changes** — overlapping modifications cause merge conflicts
- 📝 **No learning** — agents can't learn from past failures across sessions
- 🔍 **Grep is dumb** — string search finds false positives; agents need graph queries

## Our Solution

**Synapses** is a single-binary code intelligence server for AI agents:

```
┌──────────────────────────────────────────────────────────┐
│  Your IDE (Claude Code, Cursor, VS Code, Windsurf, Zed)  │
└────────────────────┬─────────────────────────────────────┘
                     │ MCP Protocol (stdio / HTTP)
                     ▼
┌──────────────────────────────────────────────────────────┐
│  synapses                                                │
│  ├─ Graph engine (nodes: functions, structs, files)      │
│  ├─ 40 MCP tools (context, graph, tasks, memory, skills) │
│  ├─ Episodic memory (past decisions, failures)           │
│  ├─ Agent message bus (broadcast work status)            │
│  ├─ Vector embeddings (built-in ONNX, zero external deps)│
│  ├─ Web cache & doc lookup (in-process)                  │
│  ├─ Optional brain LLM (Ollama, in-process)              │
│  └─ SQLite persistence (multi-session state)             │
└──────────────────────────────────────────────────────────┘
         Single binary · Local-first · 49+ languages
```

---

## Key Features

✅ **Structured Code Understanding** — Graph of functions, structs, calls, dependencies (not grep)
✅ **Multi-Agent Coordination** — Message bus + work claims prevent conflicting changes
✅ **Episodic Memory** — Agents learn: "I tried this pattern last week and it failed"
✅ **Vector Embeddings** — Built-in all-MiniLM-L6-v2 ONNX model, zero external dependencies
✅ **Optional Brain LLM** — In-process Ollama integration for summaries and ADRs
✅ **No Code Leaves Your Machine** — All processing local, no cloud
✅ **49+ Languages** — Go, TypeScript, Python, Java, Rust, C++, and 30+ more
✅ **Single Binary** — One MCP server, works with any IDE. Pre-built for macOS, Linux, Windows.
✅ **AI-Native** — Designed specifically for agent workflows (not human IDE features)

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

### 2. Initialize Your Project

```bash
cd /path/to/your/codebase
synapses init
```

That's it — two commands. The `init` wizard handles everything:

| Step | What it does |
|------|-------------|
| **[1/4] Project Setup** | Detects git, creates `synapses.json` with sensible defaults |
| **[2/4] Indexing** | Parses your codebase and builds the code graph (49+ languages) |
| **[3/4] Starting Engine** | Starts the singleton daemon and registers your project |
| **[4/4] Connect Agents** | Auto-detects installed AI agents and writes their MCP configs |

Auto-detects: Claude Code, Cursor, VS Code, Windsurf, Zed, Antigravity.

Then agents can use 40 MCP tools: `session_init`, `prepare_context`, `get_context`, `find_entity`, `recall`, `remember`, `send_message`, and more.

**Non-interactive mode:** `synapses init --yes --agents claude,cursor`

### 3. Uninstall (full control)

```bash
synapses uninstall                # Remove from current project
synapses uninstall --global       # Remove everything from your system
synapses uninstall --global --yes # Non-interactive full cleanup
```

Complete removal — stops daemon, deletes indexes, cleans agent configs, removes `~/.synapses`, and optionally deletes the binary. No data left behind.

---

## Repository

| Project | Language | Purpose | Status |
|---------|----------|---------|--------|
| **[synapses](https://github.com/SynapsesOS/synapses)** | Go | Graph engine + MCP server + brain + embeddings | ✅ Active |

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

With Synapses:
- ✅ No duplicate work (agents see what each other did)
- ✅ No surprises (changes propagate as messages)
- ✅ Learning loop (failures recorded, future agents learn from them)

---

## 40 MCP Tools

Organized across 9 categories:

| Category | Tools |
|----------|-------|
| **Session Bootstrap** | `session_init`, `end_session`, `explain_codebase`, `get_repo_map`, `discover_tools` |
| **Code Graph** | `prepare_context`, `get_context`, `find_entity`, `get_file_context`, `search`, `get_call_chain`, `get_impact`, `get_entity_history` |
| **Architecture & Rules** | `validate_plan`, `verify_implementation`, `get_violations`, `upsert_rule`, `upsert_gap`, `get_gaps` |
| **Task Memory** | `create_plan`, `get_pending_tasks`, `update_task`, `save_session_state`, `get_session_state`, `get_plans`, `link_task_nodes`, `annotate_node` |
| **Agent Coordination** | `get_agents`, `get_events` |
| **Agent Message Bus** | `send_message`, `get_messages` |
| **Episodic Memory** | `remember`, `recall`, `get_rule_candidates` |
| **ADRs** | `upsert_adr`, `get_adrs` |
| **Web & Docs** | `lookup_docs`, `web_annotate` |
| **Skills** | `list_skills`, `execute_skill` |

---

## Architecture Principles

🚫 **No Cloud** — All processing local
🔄 **Fail-Silent** — Brain down? Graph queries still work. No panics.
📦 **Single Binary** — One MCP server, works with any IDE
💾 **SQLite-Only** — Pure Go, no C dependencies at runtime
🔁 **Incremental** — File watcher re-parses only changed files
🧠 **AI-Native** — Designed for agents, not humans

---

## Privacy

**All your code stays on your machine.**

- Graph indexed locally in `~/.synapses/`
- Built-in embeddings (ONNX), no external service needed
- Optional brain LLM runs via Ollama on localhost
- Optional web search uses DuckDuckGo only when you call `lookup_docs`
- No telemetry, no tracking, no cloud uploads

---

## Getting Started

- **[synapses README](https://github.com/SynapsesOS/synapses)** — Full feature overview, all 40 MCP tools, configuration reference

---

## Community

- **Issues**: [github.com/SynapsesOS/synapses/issues](https://github.com/SynapsesOS/synapses/issues)
- **Discussions**: [github.com/SynapsesOS/synapses/discussions](https://github.com/SynapsesOS/synapses/discussions)
- **Security**: Report privately to security@synapsesos.dev

---

## Contributing

We welcome pull requests and issues! See [CONTRIBUTING.md](https://github.com/SynapsesOS/synapses/blob/main/CONTRIBUTING.md) for setup instructions.

```bash
make build && make test
```

---

## License

**MIT Licensed.**

---

**Made with ❤️ for multi-agent AI development.**

[📖 Read the Vision](https://github.com/SynapsesOS/synapses/blob/main/ROADMAP.md) · [🚀 Get Started](https://github.com/SynapsesOS/synapses#quick-start) · [💬 Discuss](https://github.com/orgs/SynapsesOS/discussions)
