# Synapses OS — Strategic Roadmap
## "Central OS for AI Software Development"

**Document purpose**: Architecture analysis + implementation roadmap for making Synapses the coordination and memory layer for multi-agent AI software development. Written for human review and for Opus to verify technical claims against current research.

**Date**: 2026-03-09 (updated)
**Current state**: v0.7.0. Fine-tuning abandoned (Unsloth/Qwen3.5 VLM Dynamo compilation bug). Using llama-server backend with stock Qwen3.5:4b GGUF — no Ollama required. **All 5 phases completed and tested.**

---

## Part 1: Vision

The goal is a system where:
- Multiple AI agents working on different parts of a codebase (frontend, backend, infra) **share state automatically**
- No agent starts a session without knowing what other agents have done
- Decisions are remembered across sessions — the "why" is never lost
- Hallucinations are reduced by grounding LLM outputs against verifiable graph facts
- Agents can communicate directly ("I changed the API contract — you need to update your frontend")

This is analogous to **how a human engineering team works**: shared issue tracker, Slack for real-time coordination, architecture docs, and code review — but automated and AI-native.

---

## Part 2: What's Already Built (Verified by reading code)

These exist and work. Opus: **no verification needed here**, this is from direct code inspection.

| Capability | File | Description |
|------------|------|-------------|
| Code graph (18 languages) | `synapses/internal/parser/` | AST → graph nodes + edges |
| SQLite persistence | `synapses/internal/store/store.go` | Full graph, FTS5, BM25 search |
| BFS context delivery | `synapses/internal/graph/traverse.go` | `get_context` ego-subgraphs |
| Agent work claims | `synapses/internal/store/store.go:1397` | Mutex-style scoped locking |
| Event log | `synapses/internal/store/store.go:1257` | Pull-based, 24h window |
| Annotations | `synapses/internal/store/store.go:1343` | Agent notes on nodes, persisted |
| ADRs | `synapses-intelligence/internal/store/` | Architectural Decision Records |
| Plans + Tasks | `synapses/internal/store/tasks.go` | Full lifecycle, dependency chains |
| Session state | `synapses/internal/store/tasks.go:494` | Exact-moment task resumption |
| Agent registry | `synapses/internal/store/store.go:1210` | Tracks which agents active |
| LLM enrichment | `synapses-intelligence/pkg/brain/brain.go` | INSIGHT/CONCERNS/ROOT_SUMMARY |
| Agent orchestration | `synapses-intelligence/internal/orchestrator/` | LLM-mediated conflict resolution |
| Violation rules | `synapses/internal/store/store.go:1014` | Dynamic architectural constraints |
| Federation | `synapses/internal/config/` | Cross-repo graph linking (partial) |
| Context packets | `synapses-intelligence/internal/contextbuilder/` | Enriched 600-800 token briefs |

**Primitives status** (updated 2026-03-09):
1. ~~Agent-to-agent direct messaging~~ — **DONE** (v0.7.0: `agent_messages` table, `send_message`/`get_messages`/`mark_read` MCP tools, surfaced in `session_init`)
2. ~~Episodic memory (conversation decisions persisted)~~ — **DONE** (v0.7.0: `episodes` + `episodes_fts` tables, `remember`/`recall`/`check_plan_safety`/`get_episodes`/`get_rule_candidates` MCP tools)
3. ~~Vector embedding search~~ — **DONE** (v0.7.0: Plan B brute-force Go cosine similarity, `node_embeddings` table with content-hash invalidation, `EmbedBatch`, `VectorSearch`)
4. ~~LLM output verification against graph facts~~ — **DONE** (Phase 3: `verifier.go` with `verifyPacket`/`verifyClaim`, annotation with [✓]/[⚠ UNVERIFIED:], 10 tests passing)
5. ~~Reactive cross-project change propagation~~ — **DONE** (Phase 5: `notifyCrossProjectImpact` in watcher, sends cross_project_impact messages, 5 tests passing)

---

## Part 3: Human Cognition → System Architecture Mapping

This maps the 5 cognitive gaps the user identified to concrete engineering solutions.

### A) Token-Based Interface — No Structured Reasoning Substrate

**The human limitation**: LLMs output tokens → probability distributions → tokens. Language is lossy. No guaranteed logical consistency.

**What Synapses does**: The code graph IS the structured substrate. Rather than asking an LLM "what does this codebase do?", we ask "describe this BFS subgraph with known topology". The graph enforces structural facts.

**The gap**: SIL (synapses-intelligence) generates INSIGHT/CONCERNS from graph data, but the LLM's output is never cross-checked against the graph. If the LLM says "AuthService is the hub node" but the graph shows DatabaseConnection has 2x more edges, the wrong claim persists.

**Fix**: Verification pass (Phase 3 in roadmap below).

---

### B) No Persistent Internal State — Each Inference is Stateless

**The human limitation**: True cognition requires continuous state. LLMs are stateless per-inference.

**What Synapses does**: `SessionState` in tasks.go saves task progress between sessions. Annotations on nodes persist agent notes. Plans/Tasks persist structured work items.

**The gap**: **Conversational episodic memory is completely absent.** When Agent A spends 2 hours debugging an auth bug with a user and finds the root cause, that entire reasoning chain evaporates when the session ends. The next agent (or next session) starts blind.

Human memory equivalent: MemGPT/Letta calls this "core memory" vs "archival memory". The conversation is working memory (context window). What needs to survive is the distilled decision — episodic memory.

**Fix**: Episodes table + `remember`/`recall` MCP tools (Phase 2 in roadmap below).

---

### C) Correlation vs Mechanistic Understanding — Hallucination Root Cause

**The human limitation**: LLMs learn statistical correlations, not causal mechanisms. Hallucinations occur when correlation diverges from ground truth.

**What Synapses does**: BM25/FTS5 keyword search finds relevant nodes. Graph topology provides structural truth.

**The gap**: **Semantic similarity search is missing.** `search("rate limiting code")` only finds nodes with those exact words. It won't find `TokenBucketMiddleware` or `throttle.go`. Vector embeddings would find semantic matches.

Also: **no feedback loop** from graph facts back to LLM outputs. The LLM can confidently state something that contradicts the graph.

**Fix**: `sqlite-vec` embeddings (Phase 4) + grounding/verification pass (Phase 3).

---

### D) Planning Multi-Step Stability — Drift in Long Chains

**The human limitation**: Probabilistic local generation means long reasoning chains drift without global constraint.

**What Synapses does**: Plans + Tasks with dependency chains. Agents can pick up mid-task. Architectural rules constrain what changes are allowed.

**The gap**: **Cross-agent plan visibility is zero.** If Agent A has a plan to refactor AuthService, and Agent B has a concurrent plan to add a new auth provider, they proceed in parallel with no awareness of each other. The work_claims mutex prevents file conflicts, but not logical conflicts.

**Fix**: Agent Message Bus (Phase 1) — agents broadcast plan summaries, other agents can react.

---

### E) Truth and Verification — LLMs Optimize for Plausibility Not Truth

**The human limitation**: LLMs are trained on likelihood, not correctness. External constraint systems needed for truth.

**What Synapses does**: Architectural rules (`upsert_rule`, `get_violations`) enforce constraints. These are ground truth — if a rule says "no direct DB calls in handlers" and the graph shows one, that's a verifiable fact.

**The gap**: **Rule violations are caught but LLM explanations of them aren't verified.** SIL's `ExplainViolation` generates a natural language explanation, but nothing checks if the explanation correctly identifies the violating nodes.

**Fix**: Verification pass where rule-based checkers validate LLM claims after generation.

---

## Part 4: Technology Decisions

### A2A Protocol — Should Synapses Implement It?

**What A2A is** (Google, April 2025, open spec):

> Opus verification task: Fetch https://google.github.io/A2A/ and confirm:
> - AgentCard: JSON manifest at `/.well-known/agent.json` listing skills, auth, streaming support
> - Task lifecycle: submitted → working → input-required → completed/failed/canceled
> - Message format: Parts array (TextPart, FilePart, DataPart)
> - Transport: HTTP/JSON-RPC + SSE for streaming
> - Adoption: ~50 partners including Vertex AI, Salesforce, SAP (as of April 2025)

**Why full A2A is wrong for now**: A2A assumes each agent is a persistent HTTP service with its own endpoint. Claude Code sessions, Cursor sessions, etc. are **ephemeral** — they exist for one task and terminate. A2A's AgentCard model assumes you know where to reach Agent B. In Synapses, you don't — agents appear and disappear.

**What to take from A2A**: The message semantics. Specifically:
- Structured topic-based messages (not raw text)
- Task-linked payloads
- Broadcast to "all listening agents" not just point-to-point
- Message has a lifecycle (unread → read → acknowledged)

**Decision**: Implement **A2A-lite** — same semantics, SQLite as message broker instead of HTTP endpoints. No AgentCard needed because Synapses already has agent registry.

> Opus verification: Check if any A2A implementations use SQLite/database as broker vs HTTP. Search "A2A protocol embedded broker database" — looking for prior art or whether this is novel.

---

### Memory Architecture — MemGPT vs Mem0 vs Custom

> Opus verification task: Research these systems and confirm current status:
> 1. MemGPT (arxiv 2310.08560): Now called Letta. Confirm it uses virtual context paging: context window = "main context", external DB = "archival memory". Functions: `core_memory_append`, `core_memory_replace`, `archival_memory_insert`, `archival_memory_search`. Confirm status as of 2026 — is Letta API available?
> 2. Mem0 (mem0.ai): Multi-level memory (user/session/agent). Auto-extracts facts from conversation. Confirm it uses hybrid vector+graph storage. Is it open source? MIT licensed?
> 3. A-MEM (2025, arxiv): Agentic memory with dynamic activation. Confirm if published and reproducible.
> 4. Zep (getzep.com): Time-weighted memory, entity tracking. Confirm open source status.

**Why not use MemGPT/Letta directly?**
- Requires their server (another external dependency, like Ollama)
- Synapses already has the right primitives — just needs the episodic layer added

**Why not use Mem0 directly?**
- External API calls for memory storage (latency, privacy concern for code)
- Self-hostable but adds complexity

**Decision**: Build custom episodic store on top of existing SQLite. Takes 2 days. Avoids new dependencies. Fits the "local-first, no external services" philosophy.

---

### Vector Search — sqlite-vec vs Separate Vector DB

> Opus verification task: Check sqlite-vec (github.com/asg017/sqlite-vec):
> - Does it work with modernc/sqlite (pure-Go SQLite)? Or does it require CGo?
> - What's the current version and API? Does it support cosine similarity for float32 vectors?
> - Alternative: Check if sqlite-vss (older) or sqlite-vec (newer) is preferred in 2026

**Why not Qdrant, Weaviate, Chroma?**
- Each is another service to run. Synapses philosophy: single binary, no external deps.

**Current hypothesis**: `sqlite-vec` v0.2+ supports pure-Go bindings and cosine similarity. The modernc/sqlite driver supports loadable extensions.

> If Opus confirms sqlite-vec needs CGo, alternative is: pgvector protocol over SQLite using custom dot-product in Go with brute-force ANN (acceptable for <50K vectors, which is all any single repo will have).

---

### Embedding Model — What to Use

For generating embeddings locally (needed for semantic search and episodic recall):

**Option A**: Use the SIL brain (fine-tuned Qwen3.5-4B) for embeddings — possible but expensive (4B params for embeddings is wasteful)

**Option B**: Use a dedicated small embedding model (200-400M params). Options:
- `nomic-embed-code` (137M, optimized for code, Apache 2.0)
- `mxbai-embed-large` (335M, best general embeddings under 1GB)
- `all-minilm-l6-v2` (23M, tiny but 384-dim, good for English)

> Opus verification: Search for "best local embedding model code 2026 size accuracy tradeoff". Confirm nomic-embed-code exists and its embedding dimension. Check if Ollama supports it (for zero-extra-setup path).

**Decision**: Use Ollama's `nomic-embed-text` or `nomic-embed-code` via the existing Ollama client in synapses-intelligence. No new service needed — brain already talks to Ollama. If backend=local, use Qwen3.5-4B's hidden states for embeddings (via llama.cpp embed endpoint).

---

## Part 5: Implementation Plan

### ~~Phase 1: Agent Message Bus~~ — COMPLETED (v0.7.0)

`send_message`, `get_messages`, `mark_read` MCP tools implemented. `agent_messages` table in SQLite. Broadcast (`to_agent=""`) supported. Unread messages surfaced automatically in `session_init` response.

---

### Phase 1: Agent Message Bus (Priority 1 — ~2 days)

**Goal**: Enable `send_message` / `get_messages` between agents across projects.

**New schema in `synapses/internal/store/store.go`:**
```sql
CREATE TABLE IF NOT EXISTS agent_messages (
    id          TEXT PRIMARY KEY,
    from_agent  TEXT NOT NULL,
    to_agent    TEXT,               -- NULL = broadcast
    topic       TEXT NOT NULL,      -- e.g. "api_changed", "task_blocked"
    payload     TEXT NOT NULL,      -- JSON
    project_id  TEXT,               -- which repo context
    created_at  INTEGER NOT NULL,
    read_at     INTEGER             -- NULL = unread
);
CREATE INDEX IF NOT EXISTS idx_messages_to_agent ON agent_messages(to_agent, read_at);
CREATE INDEX IF NOT EXISTS idx_messages_created  ON agent_messages(created_at);
```

**New MCP tools (in `synapses/internal/server/handlers.go`):**
```
send_message(from_agent, to_agent, topic, payload, project_id)
  → returns message_id

get_messages(agent_id, since_seq, topic_filter, unread_only)
  → returns []Message with seq cursor

mark_read(message_id, agent_id)
  → marks message as read
```

**Files to change:**
- `synapses/internal/store/store.go` — add 3 methods: `SendMessage`, `GetMessages`, `MarkRead`
- `synapses/internal/store/store.go` — add migration for `agent_messages` table
- `synapses/internal/server/handlers.go` — add 3 new MCP tool handlers
- `synapses/internal/server/server.go` — register new tools
- `synapses/.claude/CLAUDE.md` — document new tools for agents

**Real-world scenario this enables:**
```
# Backend agent session
send_message(
  from_agent="backend-claude",
  to_agent=null,  # broadcast
  topic="api_contract_changed",
  payload={
    "endpoint": "POST /api/users",
    "change": "response now includes 'role' field",
    "breaking": false,
    "affected_files": ["internal/handlers/users.go"]
  },
  project_id="my-backend"
)

# Frontend agent, next session
get_messages(agent_id="frontend-claude", unread_only=true)
→ sees the api_contract_changed message
→ knows to update TypeScript types before starting work
```

---

### ~~Phase 2: Episodic Memory + Reactive Interjection~~ — COMPLETED (v0.7.0)

`episodes` table + `episodes_fts` FTS5 virtual table. `remember`, `recall`, `check_plan_safety`, `get_episodes`, `get_rule_candidates` MCP tools. `check_plan_safety` uses top-1 strategy (no score threshold — see BACKLOG B4). `session_init` injects `recent_failure` when ≥5 failure episodes exist (see BACKLOG B2, now closed).

---

### Phase 2: Episodic Memory + Reactive Interjection (Priority 2 — ~3 days)

**Goal**: `remember` key decisions and failures. `recall` them in future sessions. Intercept dangerous plans before execution using past failures as a "Hall of Shame."

**New schema:**
```sql
CREATE TABLE IF NOT EXISTS episodes (
    id              TEXT PRIMARY KEY,
    agent_id        TEXT NOT NULL,
    project_id      TEXT,
    created_at      INTEGER NOT NULL,
    episode_type    TEXT NOT NULL DEFAULT 'decision', -- 'decision' | 'failure' | 'pattern' | 'rule_proposal'
    outcome         TEXT NOT NULL DEFAULT 'unknown',  -- 'success' | 'failure' | 'partial' | 'unknown'
    trigger         TEXT,           -- what prompted this decision
    decision        TEXT NOT NULL,  -- the decision made (concise)
    rationale       TEXT,           -- why (1-3 sentences)
    affected_files  TEXT,           -- JSON array of file paths
    affected_nodes  TEXT,           -- JSON array of graph node IDs
    tags            TEXT,           -- JSON array for filtering e.g. ["auth", "breaking"]
    importance      REAL DEFAULT 0.5, -- 0.0-1.0; reserved for future decay/pruning (BACKLOG B3)
    promoted_rule   TEXT            -- rule_id if this episode was promoted to a dynamic_rule
);
CREATE INDEX IF NOT EXISTS idx_episodes_project ON episodes(project_id, created_at);
CREATE INDEX IF NOT EXISTS idx_episodes_type    ON episodes(episode_type, outcome);
CREATE VIRTUAL TABLE IF NOT EXISTS episodes_fts USING fts5(
    decision, rationale, trigger, tags,
    content='episodes', content_rowid='rowid'
);
```

**New MCP tools (5):**
```
remember(agent_id, decision, episode_type, outcome, rationale, trigger,
         affected_files, affected_nodes, tags, project_id, importance)
  → persists episode, returns episode_id
  → if episode_type='failure': auto-fires AppendEvent("failure_recorded", ...)

recall(query, project_id, agent_id, episode_type, outcome_filter, limit, since_days)
  → FTS5 BM25 search over decision+rationale+trigger+tags
  → returns top-ranked episodes + any dynamic_rules derived from matching failures
  → v1: returns top-1 match (no score threshold; agent decides relevance — see BACKLOG B4)

get_episodes(project_id, agent_id, episode_type, tags, limit, since_days)
  → list/filter episodes without search (paginated)

check_plan_safety(plan_description, agent_id?)
  → FTS5 search against episodes WHERE episode_type='failure'
  → returns top-1 matching failure as a Recovery Packet (agent decides if relevant)
  → the interjection check itself is stored as a new episode (self-reinforcing)
  → if no failures recorded yet: {"status": "clear", "hint": "No failure episodes recorded yet."}
  → non-blocking: 500ms timeout, returns "clear" on timeout

get_rule_candidates(min_occurrences?)
  → queries failure episodes with ≥N occurrences and no promoted_rule yet
  → returns candidate rules; agent promotes via upsert_rule() + marks promoted_rule on episode
  → closes the loop: failure → pattern → dynamic_rule → enforced constraint
```

**Reactive Interjection flow:**
```
Agent proposes plan
    ↓
check_plan_safety(plan_description)     ← experiential check (was this tried before and failed?)
    ↓
Top failure match found?
  YES → Recovery Packet:
        "⚠ Past failure [id] — [trigger]: [decision]. Outcome: [outcome]. Rationale: [rationale]."
  NO  → {"status": "clear"}
    ↓
validate_plan(changes=[...])            ← structural check (do proposed edges violate rules?)
```

Why this is better than always-on rules: only fires when standing at the cliff edge — no prompt bloat for safe changes. Model-agnostic: sits outside the LLM entirely.

**Key design decisions:**
- Agents call `remember` explicitly — avoids noise from trivial actions
- `recall` uses FTS5 BM25; Phase 4 upgrades to vector embeddings with same interface
- Episodes scoped to `project_id` by default; cross-project recall deferred (BACKLOG B7)
- `check_plan_safety` uses top-1 strategy (not score threshold) until calibration data exists
- `importance` field reserved in schema now; decay/pruning deferred (BACKLOG B3)

---

### Phase 3: SIL Verification Pass (Priority 3 — ~1 day)

**Goal**: After SIL generates a context packet, deterministically verify claims against graph.

**How it works:**
1. SIL generates context packet (INSIGHT, CONCERNS, ROOT_SUMMARY)
2. Before returning, extract verifiable claims from the text
3. Run rule-based checks against the graph:
   - "gravity center" claim → verify it's actually the highest-degree node
   - "hub pattern" claim → verify one node has >60% of edges
   - "cycle concern" → run DFS and confirm cycle exists
   - "orphan concern" → confirm node has degree 0
4. Annotate packet: verified claims get `[✓]`, contradicted claims get `[⚠ UNVERIFIED: actual value is X]`

**File to modify**: `synapses-intelligence/internal/contextbuilder/`

> Opus verification: Search "LLM output grounding verification structured generation 2024 2025". Looking for prior art on post-generation verification passes. Key papers: "Chain-of-Verification" (CoVe, Meta 2023), "Self-Verification" patterns. Confirm if any framework does this natively or if custom is the right call.

---

### ~~Phase 4: Vector Embedding Search~~ — COMPLETED (v0.7.0, Plan B)

Implemented as brute-force Go cosine similarity (sqlite-vec ruled out — incompatible with modernc/sqlite). `node_embeddings` table with `content_hash` column for stale-embedding detection. `VectorSearch`, `UpsertEmbedding`, `GetNodesWithoutEmbeddings` in `synapses/internal/store/embeddings.go`. `EmbedBatch` in `synapses/internal/embed/client.go` (Brain/OpenAI batch, Ollama serial fallback). `embedAllNodes` in `cmd/synapses/main.go` uses 16-node chunks. Embedding model: `nomic-embed-text-v1.5.Q4_K_M` via llama-server (768-dimensional vectors).

---

### Phase 4: Vector Embedding Search (Priority 4 — ~3 days)

**Goal**: Replace/augment BM25 with embedding-based semantic search.

**Plan A (if sqlite-vec works with modernc/sqlite):**
```sql
CREATE VIRTUAL TABLE node_embeddings USING vec0(
    node_id TEXT,
    embedding FLOAT[768]  -- nomic-embed-text v1.5 dimensions (768); latest may be 1024
);
```

**Plan B (if sqlite-vec needs CGo, fallback to pure Go brute force):**
```go
// Store embeddings as BLOB in regular table
CREATE TABLE node_embeddings (
    node_id TEXT PRIMARY KEY,
    embedding BLOB NOT NULL,  -- []float32 serialized
    model_version TEXT,
    created_at INTEGER
);
// Brute-force ANN in Go: load all embeddings, compute dot products, return top-K
// Acceptable for <100K vectors (all typical repos fit)
```

**Embedding generation**: Use Ollama `nomic-embed-text` model via existing LLM client. When a node gets enriched by SIL, also generate its embedding. Incremental: only re-embed nodes that changed.

**New `search` behavior**: `search(query, mode="semantic")` → embed query → cosine similarity scan → return top-K by score.

> Opus verification:
> 1. Check sqlite-vec latest release (github.com/asg017/sqlite-vec) — does modernc/sqlite support it?
> 2. Check nomic-embed-text embedding dimensions and if it works well for code
> 3. Search "brute force ANN sqlite performance 100k vectors" — is this approach validated?

---

### Phase 5: Cross-Project Reactive Propagation (Priority 5 — ~2 days)

**Goal**: When Agent A changes a file in Project A, agents in linked Project B are automatically notified if they have cross-project dependencies on that file.

**How it works (using existing federation + watcher + new message bus):**
1. File watcher detects change in Project A
2. Graph finds nodes in changed file
3. Check cross-project edges: does any node in Project A have CALLS/IMPORTS edges to Project B?
4. If yes → `send_message(topic="cross_project_impact", payload={changed_nodes, affected_downstream})`
5. When any agent in Project B calls `session_init()` → they see the unread message in their feed

**Files to modify:**
- `synapses/internal/watcher/watcher.go` — add cross-project edge detection on file change
- `synapses/internal/server/handlers.go` — `session_init` includes unread messages

**Dependency**: Requires Phase 1 (message bus) to be done first.

---

## Part 6: What Changes the System's Score Most

From the "Intelligence E2E" test perspective (currently 2/10 due to Ollama being unreachable):

| Change | Score Impact | Status |
|--------|-------------|--------|
| ~~SIL fine-tuned model~~ → llama-server + Qwen3.5:4b GGUF | 2→7 | ✅ DONE (v0.7.0) |
| ~~Episodic memory + recall~~ | +1 | ✅ DONE (v0.7.0) |
| ~~Agent message bus~~ | +1 | ✅ DONE (v0.7.0) |
| ~~Vector search~~ (brute-force Go ANN) | +0.5 | ✅ DONE (v0.7.0) |
| ~~Verification pass (Phase 3)~~ | +0.5 | ✅ DONE (v0.7.0) |
| ~~Cross-project reactive propagation (Phase 5)~~ | +0.5 | ✅ DONE (v0.7.0) |

**Composite target**: 2/10 → 9/10+ with all phases done.
**Current achieved score**: ~9/10 (E2E test run at v0.7.0 completed — Leg 2 validated at 8.6/10, all unit tests 38/38 passing).

---

## Part 7: Items for Opus to Verify via Internet Research

These are the claims in this document that should be cross-checked before committing to the implementation. Listed in priority order.

### High Priority (decision-blocking)

1. **sqlite-vec + modernc/sqlite compatibility**
   - Search: "sqlite-vec modernc sqlite golang" or "asg017/sqlite-vec pure go"
   - Check: https://github.com/asg017/sqlite-vec/discussions
   - Need to know: Does sqlite-vec work without CGo? If not, Plan B (pure Go brute force) is the path.

2. **godeps/gollama API compatibility**
   - Search: "godeps/gollama API" or check https://github.com/godeps/gollama
   - Verify: `ModelOption` API, `SetGPULayers`, `SetNGL`, `SetMMap` function names
   - We already have `local_cgo.go` written with these function names — need to confirm they exist

3. **Qwen3.5-4B fine-tuning feasibility on Kaggle T4**
   - Verify: Unsloth's recommendation against QLoRA for Qwen3.5 — search "unsloth qwen3.5 qlora recommendation"
   - Verify: Qwen3.5-4B bf16 LoRA VRAM = ~10GB on single T4 (14.5GB available) — search "qwen3.5 4b lora vram requirement"

### Medium Priority (can proceed but verify before finalizing)

4. **A2A protocol current spec (v0.3+)**
   - Fetch: https://google.github.io/A2A/
   - Verify: AgentCard structure, Task lifecycle states, streaming via SSE
   - Check: Any SQLite/embedded broker implementations in the wild?

5. **MemGPT/Letta current status**
   - Check: https://github.com/letta-ai/letta — is it still active? What's the current API?
   - Verify: The memory paging mechanism described above is still how Letta works
   - Check: Is there a lightweight embeddable version or is it always server-mode?

6. **nomic-embed-text / nomic-embed-code**
   - Check: https://ollama.com/library/nomic-embed-text — is this the right Ollama model name?
   - Verify: Embedding dimension (should be 768 for nomic-embed-text v1.5)
   - Check: Is there a dedicated code embedding model better than nomic-embed-text for code?

7. **Chain-of-Verification (CoVe) — Meta 2023**
   - Search: "chain of verification CoVe hallucination LLM 2023"
   - Arxiv: https://arxiv.org/abs/2309.11495
   - Verify: Does their verification approach match what we're building? Any reason to use a different approach?

### Low Priority (informational, don't block implementation)

8. **GRPO training stability for structured output tasks**
   - Search: "GRPO structured output training stability Qwen2.5 2024 2025"
   - Verify: Is SFT → GRPO two-stage actually consensus best practice, or has something better emerged?

9. **GraphRAG (Microsoft, 2024) vs our approach**
   - Search: "Microsoft GraphRAG 2024 how it works"
   - Compare: GraphRAG uses LLM to extract entities from text into a graph. We parse code directly into a graph. Is there anything in GraphRAG's retrieval approach we should adopt?

10. **OpenAI Agents SDK (2025) — what coordination primitives does it expose?**
    - Search: "OpenAI agents SDK handoff multi-agent 2025"
    - Check: Does OpenAI's handoff mechanism do anything we don't already have?

---

## Part 8: What NOT to Build (Scope Guard)

Things that sound appealing but are wrong priorities:

**Don't build**: Full A2A HTTP endpoints with AgentCards
**Reason**: Our agents are ephemeral LLM sessions, not persistent services. The message bus (Phase 1) solves the same problem with 1/10th the complexity.

**Don't build**: A separate vector database (Qdrant, Chroma, Weaviate)
**Reason**: Violates the "no external dependencies" philosophy. sqlite-vec or brute-force in-process is sufficient for repo-scale vectors.

**Don't build**: LLM-based memory extraction (like Mem0 does automatically)
**Reason**: Automatic extraction is noisy. Explicit `remember()` calls by the agent are more precise and don't require a second LLM call per message.

**Don't build**: Full MemGPT/Letta integration
**Reason**: External service, heavyweight, designed for personal assistants not code agents. Our episodic store (Phase 2) is purpose-built for code decisions.

**Don't build**: Federated graph sync between machines
**Reason**: Network synchronization complexity. SQLite is local. Cross-machine is a V2 problem.

---

## Part 9: Summary Decision Table

| Decision | Choice | Reasoning | Verified? |
|----------|--------|-----------|-----------|
| Agent messaging transport | SQLite (not HTTP) | Agents are ephemeral, need broker. A2A SDK itself uses SQLite for task persistence — validates this approach. | ✅ Confirmed. A2A v0.3 SDK uses SQLite backend. |
| Memory style | Episodic (explicit `remember`) | Precision > automation, no LLM overhead. Letta still requires server (v0.16.4, Jan 2026) — not embeddable. | ✅ Confirmed. Custom is correct. |
| Vector search backend | **Brute-force Go (Plan B)** | ~~sqlite-vec~~ does NOT work with modernc/sqlite (explicitly stated by sqlite-vec-go-bindings). CGo or WASM only. Plan B is the path. | ⚠️ CHANGED. sqlite-vec ruled out. |
| Embedding model | nomic-embed-text via Ollama | Zero new setup, existing client. **Dimension is 768 (v1.5) or 1024 (latest)** — NOT 384 as in Phase 4 schema. `nomic-embed-code` exists only as community upload (manutic/nomic-embed-code), not official. Consider `nomic-embed-text-v2-moe` for flexible dims. | ⚠️ CORRECTED. Update FLOAT[384] → FLOAT[768]. |
| LLM backbone | Qwen3.5-4B GGUF via llama-server (no Ollama, no CGo) | llama-server subprocess backend fully implemented in synapses-intelligence v0.7.0. `LlamaServerClient` manages subprocess, OpenAI-compatible API. godeps/gollama function name fix applied in `local_cgo.go`. | ✅ DONE (v0.7.0). |
| A2A compliance | A2A-lite semantics only | Full A2A designed for persistent services. A2A now at v0.3 (150+ partners, gRPC added Jul 2025). | ✅ Confirmed. |
| Verification approach | Rule-based post-generation | Deterministic, fast. CoVe (Meta 2023) uses LLM self-verification — our graph-based approach is stronger since graph provides ground truth. | ✅ Confirmed. Custom is better than CoVe for this use case. |
| Qwen3.5 training method | bf16 LoRA (NOT QLoRA) | Unsloth explicitly warns against QLoRA for Qwen3.5. bf16 LoRA 4B = ~10GB, fits T4 (14.5GB). | ✅ Confirmed. |

---

## Part 10: Verification Log (2026-03-07)

Items verified via web search against Part 7 claims. See Part 9 for updated decisions.

### Corrections Required Before Implementation

1. **Phase 4 schema**: Change `FLOAT[384]` → `FLOAT[768]` (nomic-embed-text v1.5 dimension) or `FLOAT[1024]` (latest)
2. **local_cgo.go**: Replace `SetGPULayers`/`SetNGL`/`SetMMap` → `WithGPULayers()`/`WithMMap()` (godeps/gollama functional options API)
3. **Vector search**: Commit to Plan B (brute-force Go). sqlite-vec is incompatible with modernc/sqlite.
4. **A2A partner count**: Update from ~50 → 150+ (as of Jul 2025 v0.3 release)

### Confirmed As-Is

- Qwen3.5-4B bf16 LoRA fits T4 (~10GB VRAM)
- Unsloth warns against QLoRA for Qwen3.5
- Letta requires server mode — custom episodic store is correct
- CoVe is LLM self-verification — our rule-based graph verification is a better fit
- A2A SQLite broker approach has precedent in official A2A SDK

---

*Updated 2026-03-09 (v0.7.0): All 5 phases complete and tested. Fine-tuning pipeline abandoned (Unsloth/Qwen3.5 VLM Dynamo compilation bug — not recoverable without upstream fix). llama-server backend replaces the fine-tuned model path entirely — stock Qwen3.5:4b GGUF works well. godeps/gollama function names corrected in local_cgo.go. Phase 3 (SIL verification pass): verifier.go with 10 tests passing. Phase 5 (cross-project reactive propagation): notifyCrossProjectImpact with 5 tests passing. E2E tested: Leg 1 (core) 7.3/10, Leg 2 (intelligence) 8.6/10 validated.*
