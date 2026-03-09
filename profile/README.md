# 🧠 Synapses OS — Code Intelligence for AI Agents

**Multi-agent AI development needs shared state. We build the coordination and memory layer.**

## The Problem

When multiple AI agents work on the same codebase:

- 🔄 **No shared context** — agents duplicate work, one doesn't know what another did
- ⚠️ **Conflicting changes** — overlapping modifications cause merge conflicts
- 📝 **No learning** — agents can't learn from past failures across sessions
- 🔍 **Grep is dumb** — string search finds false positives; agents need graph queries

## Our Solution

**Synapses OS** is three specialized services working together:

```
┌─────────────────────────────────────────────────────────┐
│              Your IDE (Claude Code, Cursor, Zed, etc.)   │
└────────────────────┬────────────────────────────────────┘
                     │ MCP Protocol (stdio)
                     ▼
┌─────────────────────────────────────────────────────────┐
│  synapses (core)                                         │
│  ├─ Graph engine (nodes: functions, structs, files)    │
│  ├─ 38 MCP tools (context, graph queries, tasks)       │
│  ├─ Episodic memory (past decisions, failures)         │
│  ├─ Agent message bus (broadcast work status)          │
│  └─ SQLite persistence (multi-session state)           │
└────────┬────────────────────────────────────┬──────────┘
         │ HTTP                               │ HTTP
         ▼                                     ▼
    ┌──────────────────┐            ┌──────────────────┐
    │ synapses-        │            │ synapses-        │
    │ intelligence     │            │ scout            │
    │ ├─ 4-tier LLMs  │            │ ├─ Web search   │
    │ ├─ Summaries    │            │ ├─ URL fetch    │
    │ ├─ Insight gen  │            │ ├─ Distillation │
    │ └─ Embeddings   │            │ └─ YouTube      │
    └──────────────────┘            └──────────────────┘
         llama-server                  DuckDuckGo/Tavily
         (local, no Ollama)            (optional API key)
```

---

## Key Features

✅ **Structured Code Understanding** — Graph of functions, structs, calls, dependencies (not grep)
✅ **Multi-Agent Coordination** — Message bus + work claims prevent conflicting changes
✅ **Episodic Memory** — Agents learn: "I tried this pattern last week and it failed"
✅ **Local LLMs** — Brain sidecar (Qwen/Mistral/etc) on llama-server, no cloud
✅ **No Code Leaves Your Machine** — All inference + search local-first
✅ **18 Languages** — Go, TypeScript, Python, Java, Rust, C++, and more
✅ **AI-Ready** — Design specifically for agent workflows (not human IDE features)

---

## Quick Start

### 1. Install Synapses

```bash
curl -fsSL https://raw.githubusercontent.com/SynapsesOS/synapses/main/install.sh | sh
```

### 2. Index Your Project

```bash
cd /path/to/your/codebase
synapses init       # Parse code, build graph
```

### 3. Start the Server

```bash
synapses start -path /path/to/your/codebase
# Ready for IDE connection
```

### 4. Optional: Start Brain Sidecar

```bash
brain setup --llama-server  # Download llama-server + model
brain serve                  # Starts on :11435
```

### 5. Use in Your IDE

```bash
# Configure for Claude Code, Cursor, Zed, etc.
synapses mcp-setup --agent <cursor|claude|zed|windsurf>
```

Then agents can use 38+ MCP tools: `get_context`, `find_entity`, `recall`, `remember`, `send_message`, and more.

---

## The Three Repositories

| Project | Language | Purpose | Status |
|---------|----------|---------|--------|
| **[synapses](https://github.com/SynapsesOS/synapses)** | Go | Graph engine + MCP server | ✅ v0.7.0 (all 5 phases) |
| **[synapses-intelligence](https://github.com/SynapsesOS/synapses-intelligence)** | Go | AI brain sidecar (4-tier LLM) | ✅ v0.7.0 |
| **[synapses-scout](https://github.com/SynapsesOS/synapses-scout)** | Python | Web intelligence sidecar | ✅ v0.0.5 |

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

## Architecture Principles

🚫 **No Cloud** — All processing local (CPU/GPU)
🔄 **Fail-Silent** — Brain down? Graph queries still work. No panics.
📦 **Single Binary** — One MCP server, works with any IDE
💾 **SQLite-Only** — Pure Go, no C dependencies at runtime
🔁 **Incremental** — File watcher re-parses only changed files
🧠 **AI-Native** — Designed for agents, not humans

---

## Privacy

**All your code stays on your machine.**

- Graph indexed locally in `~/.cache/synapses/`
- LLM inference on localhost (llama-server, no Ollama required)
- Optional web search uses DuckDuckGo (or your Tavily key) only when you call `web_search`
- No telemetry, no tracking, no cloud uploads

---

## Getting Started

- **[synapses README](https://github.com/SynapsesOS/synapses)** — Full feature overview + all MCP tools
- **[synapses-intelligence README](https://github.com/SynapsesOS/synapses-intelligence)** — Brain setup (CPU/GPU auto-detect)
- **[synapses-scout README](https://github.com/SynapsesOS/synapses-scout)** — Web search + distillation

---

## Community

- **Issues**: Report bugs per-repository (synapses / intelligence / scout)
- **Discussions**: Ask questions on GitHub Discussions
- **Security**: Report privately to security@synapsesos.dev

---

## Contributing

We welcome pull requests and issues! Each project has contribution guidelines:

- **Go projects** (synapses, synapses-intelligence): `make build && make test`
- **Python project** (synapses-scout): `pip install -e ".[dev]" && make test`

---

## License

All three projects are **MIT Licensed**.

---

**Made with ❤️ for multi-agent AI development.**

[📖 Read the Vision](https://github.com/SynapsesOS/synapses/blob/main/ROADMAP.md) · [🚀 Get Started](https://github.com/SynapsesOS/synapses#quick-start) · [💬 Discuss](https://github.com/orgs/SynapsesOS/discussions)
