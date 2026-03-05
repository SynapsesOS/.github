SynapsesOS — Project Overview
What Is Synapses-OS?
Synapses-OS is an agentic control plane — a local-first, open-source platform designed to make AI coding agents smarter, faster, and more structurally aware. It solves a core problem: AI agents (Claude Code, Cursor, etc.) typically either grep raw files or hallucinate when asked about a codebase. Synapses replaces that with a relational code graph — a living, indexed model of your project that agents query with precision.
The monorepo lives at /synapses-os/ and contains 3 active sub-projects and 2 planned (empty) sub-projects.

System Architecture


┌─────────────────────────────────────────────────────────┐
│              AI Agent (Claude Code / Cursor)             │
│             MCP Protocol (stdio / .mcp.json)            │
└─────────────────┬───────────────────────────────────────┘
                  │
      ┌───────────▼───────────┐
      │   synapses (Core)     │  :MCP Server (stdio)
      │   Go — graph engine   │
      │   18-lang AST parser  │
      │   SQLite persistence  │
      └──────┬────────┬───────┘
             │        │
   ┌──────────▼──┐  ┌──▼────────────────────┐
   │  synapses-  │  │   synapses-scout       │
   │intelligence │  │   Python — :11436      │
   │  Go — :11435 │  │   Web search/fetch    │
   └──────┬──────┘  └────────────────────────┘
          │
   ┌──────▼──────┐
   │   Ollama    │  :11434 (local LLM runtime)
   │ qwen3.5 0.8B│
   │ up to 9B    │
   └─────────────┘

Sub-Project 1: 

synapses (Core MCP Server)
Language: Go 1.26Role: The heart of the system. Parses your codebase into a graph, serves it to AI agents via MCP.
What it does
* Parses 18 languages using tree-sitter AST extraction (Go, TS/JS, Python, Java, Rust, C/C++, Swift, etc.)
* Builds an in-memory relational graph of your code (nodes = entities, edges = relationships)
* Persists the graph to SQLite (modernc/sqlite — pure Go, no CGo)
* Serves 24 MCP tools to agents over stdio
* Watches files live and incrementally re-parses on save (~100ms)
Key Internal Packages
Package	Purpose
internal/graph	Core graph engine: BFS traversal, node/edge types, relevance carving
internal/parser	tree-sitter → graph mapper for 18 languages
internal/resolver	Post-parse CALLS + IMPLEMENTS edge resolution
internal/mcp	MCP server + all 24 tool handlers
internal/store	SQLite persistence (graph, tasks, rules, annotations)
internal/watcher	fsnotify live file watcher (150ms debounce)
internal/brain	HTTP client to synapses-intelligence sidecar
internal/scout	HTTP client to synapses-scout sidecar
internal/peer	Federation / monorepo cross-project graph linking
internal/config	synapses.json loader + architectural rule checker
internal/dataflow	DATA_FLOWS edge type support
internal/metrics	Performance/operational metrics
Graph Model
Node Types: 

file, package, function, method, struct, interface, variable

Edge Types (with relevance weights):
Edge	Weight	Meaning
CALLS	1.0	Function/method invocation
DATA_FLOWS	0.95	Data dependency
IMPLEMENTS	0.9	Interface satisfaction
EMBEDS	0.85	Struct embedding
DEPENDS_ON	0.8	Package dependency
IMPORTS	0.7	File imports
EXPORTS	0.5	Symbol export
DEFINES	0.15	File → entity (low to avoid hub inflation)
Context Carving Algorithm
When an agent calls get_context("AuthService", depth=2):
1. Find the node
2. BFS outward up to depth hops
3. Score: relevance = edge_weight × decay^hop (default decay = 0.5)
4. Sort by relevance, prune to token_budget (default 4000)
5. Return a ranked subgraph — not a file dump
Performance targets: cold parse 10k files < 30s, get_context < 10ms, incremental re-parse < 100ms.
Key MCP Tools
* Session: session_init, get_project_identity, get_working_state
* Context: get_context, get_file_context, find_entity, search, get_call_chain, get_impact
* Architecture: validate_plan, get_violations, upsert_rule, annotate_node
* AI Brain: upsert_adr, get_adrs
* Web: web_search, web_fetch, web_deep_search, web_annotate
* Task Memory: create_plan, get_pending_tasks, update_task, save_session_state
Key Files
* cmd/synapses/ — CLI entry (init, index, start, list, reset, status)
* internal/mcp/tools.go (57KB) — Main tool handler implementations
* internal/mcp/server.go (33KB) — MCP server setup
* internal/graph/types.go — Node/Edge types, CarveConfig, SubGraph, ProjectIdentity
* internal/graph/traverse.go — BFS ego-graph carver
* synapses.example.json — Full config reference

Sub-Project 2: synapses-intelligence (AI Brain Sidecar)
Language: Go 1.26Role: An HTTP sidecar that adds local LLM-powered semantic reasoning to the graph.Port: localhost:11435
What it does
* Generates prose briefings (2-3 sentences) for code entities via Ollama
* Builds Context Packets — structured semantic documents that replace raw graph nodes, cutting token usage by ~80%
* Explains architectural rule violations in plain English
* Provides SDLC phase + quality mode management
* Runs a co-occurrence learning loop — learns "when you edit X, also check Y"
* Resolves multi-agent conflicts (who owns what scope)
4-Tier Model Architecture
The intelligence system uses the smallest capable model per task:
Tier	Name	Model	RAM	Latency	Tasks
0	Reflex	qwen3.5:0.8b	1GB	~3s	Prose briefings, web pruning
1	Sensory	qwen3.5:2b	2GB	~5s	Violation explanations
2	Specialist	qwen3.5:4b	4GB	~12s	Architectural enrichment
3	Architect	qwen3.5:9b	8GB	~25s	Multi-agent coordination
All tiers are fail-silent — if Ollama is down, raw context is returned without errors.
Key Internal Packages
Package	Purpose
internal/ingestor	Tier 0: generates prose briefings via Ollama
internal/enricher	Tier 2: architectural insight for a code neighbourhood
internal/guardian	Tier 1: explains rule violations in plain English
internal/orchestrator	Tier 3: multi-agent conflict resolution
internal/contextbuilder	Assembles ContextPacket from all sources
internal/sdlc	SDLC phase/quality mode management
internal/store	SQLite for summaries, patterns, cache
internal/llm	Ollama HTTP client abstraction
internal/pruner	Strips boilerplate from web content
HTTP API Endpoints
* GET /v1/health — LLM connectivity check
* POST /v1/ingest — Generate prose briefing for a code entity
* POST /v1/prune — Strip boilerplate from raw web text
* GET /v1/summary/{nodeId} — Fetch stored summary (no LLM)
* POST /v1/enrich — 2-sentence architectural insight for a neighbourhood
* POST /v1/explain-violation — Plain-English explanation + fix for a rule violation
* POST /v1/coordinate — Suggest work distribution for conflicting agents
* POST /v1/context-packet — Assemble a full phase-aware Context Packet
* GET|PUT /v1/sdlc — SDLC phase and quality mode management
* POST /v1/decision — Log agent action (feeds learning loop)
* GET /v1/patterns — View learned co-occurrence patterns

Sub-Project 3: synapses-scout (Web Intelligence Layer)
Language: Python 3.11+Role: The "sensory cortex" — gives agents the ability to search and read the web.Port: localhost:11436
What it does
* Unified scout.fetch() interface — auto-detects: search query, web URL, or YouTube
* Fast-path extraction: httpx + trafilatura for ~80% of pages (<1s)
* JS-heavy fallback: Crawl4AI browser automation (3-8s)
* Orchestrated deep search: Expands queries into multiple angles, fans out in parallel, deduplicates by URL
* YouTube intelligence: yt-dlp metadata + auto-generated transcript extraction
* Intelligence distillation: Sends content to synapses-intelligence /v1/prune → /v1/ingest for LLM cleaning
* SQLite cache: TTL-based (search: 6h, web: 24h, YouTube: 7d)
* News + image search via DuckDuckGo (or Tavily with API key)
Key Source Files
File	Purpose
scout.py	Core Scout class — unified fetch/search interface
orchestrator.py	Multi-query orchestration (query expansion, fan-out, dedup)
server.py	Starlette HTTP server
cli.py	Click CLI (scout fetch, scout serve, scout deep-search, etc.)
cache.py	SQLite TTL cache with URL normalization
searcher/	DuckDuckGo + Tavily search adapters
extractor/	httpx + trafilatura + Crawl4AI extraction
media/	yt-dlp YouTube handler
distiller/	Intelligence distillation pipeline
Python Dependencies
pydantic, httpx, aiosqlite, starlette, uvicorn, crawl4ai, yt-dlp, ddgs, trafilatura

Planned Sub-Projects (Not Yet Started)
Directory	Status	Likely Purpose
synapses-pulse	Empty	Real-time event streaming / live monitoring layer
synapses-remote	Empty	Remote/cloud deployment or remote team collaboration
Monorepo Configuration
Root 

synapses.json defines:

* Brain URL: http://localhost:11435
* Scout URL: http://localhost:11436
* Architectural rule: Graph layer files (*/graph/*.go) must not directly call Store.* methods
Port Allocation:
Service	Port
Ollama	11434
synapses-intelligence (brain)	11435
synapses-scout	11436
synapses MCP server	stdio (not HTTP)
Key Design Principles
1. Local-first, no cloud required — everything runs on-device
2. Fail-silent — if any sidecar (intelligence, scout) is unavailable, synapses degrades gracefully; agents always get something usable
3. No CGo — modernc/sqlite (pure Go) used throughout Go projects
4. Zero operational overhead — single binary for core; no Docker, no background daemons
5. Token efficiency — BFS carving with relevance decay; Context Packets cut raw node size by ~80%
6. Agent task memory — tasks/plans/session state persist across LLM conversations via SQLite
7. Architectural rules — enforced at parse time AND queryable by agents; new rules can be created at runtime and persist immediately
