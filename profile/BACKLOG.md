# Synapses OS — Backlog

Features parked here are architecturally sound but add complexity without enough immediate value.
Each item notes **why it was parked** and **what condition would make it worth revisiting**.

---

## B1: Reflective Synthesis (The Auditor) — ~~CLOSED~~

**Implemented**: `writeRetrospectiveAnnotations` in `synapses/internal/mcp/task_tools.go`.

When `update_task(status='done')` fires on a task with `linked_nodes`, a background goroutine annotates each linked node that has fanin > 3 with a system-generated retrospective note. The annotation records the task title, completing agent, and any completion notes. Visible in `get_context` and `find_entity` responses as `source='system'` annotations.

**Schema changes shipped**:
- `annotations.source TEXT NOT NULL DEFAULT 'agent'` column added (schema + ALTER TABLE migration)
- `Store.AddSystemAnnotation(nodeID, note)` — inserts with `source='system'`
- `Annotation.Source` field added to struct + `GetAnnotationsForNodes` scan

**Coupling filter**: fanin > 3 (not edge count > 5 as originally spec'd — adjusted for medium-sized graphs).

**Revisit if**: Brain sidecar provides semantic coupling scores → swap the fanin threshold for a smarter signal.

---

## B2: `session_init` Failure Episode Injection — ~~CLOSED~~

**Implemented**: `handleSessionInit` in `synapses/internal/mcp/tools.go`.

`session_init` now injects a `recent_failure` field when ≥5 failure-type episodes are recorded AND recent file changes are present (reducing noise on cold starts). Uses FTS BM25 recall against the most recently changed file name to pick the most contextually relevant failure episode. Returns a single episode object with `decision`, `rationale`, `outcome`, `created_at`.

**Design note**: Implemented as top-1 strategy (not top-3) and guarded by the ≥5 episode threshold to avoid noise during early use. Cross-project recall and score-threshold tuning remain in BACKLOG B4 and B7.

---

## B3: Episode Decay / Pruning

**What it was**: Low-importance episodes (importance < 0.3) auto-expire after 90 days. A background job or lazy-delete on read prunes stale data.

**Why parked**:
- Premature optimization. Until the episodes table actually grows large enough to cause performance issues, pruning adds complexity with zero benefit.
- The `importance` field is already in the schema — the pruning logic can be added later without schema changes.

**Revisit when**: An active team accumulates 1000+ episodes and query latency on `recall()` degrades measurably.

---

## B4: BM25 Threshold Calibration for `check_plan_safety`

**What it was**: The Reactive Interjection fires when FTS5 BM25 score > 0.6. A configurable threshold per project.

**Why parked**:
- FTS5 BM25 scores are not normalized 0-1. They are log-based, query-dependent, and vary significantly with corpus size. A fixed "0.6" threshold has no empirical basis.
- Needs real failure data to calibrate: record BM25 scores for true-positive and false-positive matches, then set threshold at the inflection point.

**Immediate approach**: For v1, use top-1 match strategy — return the single highest-scoring failure episode if the corpus is non-empty, and let the agent decide relevance. No threshold needed.

**Revisit when**: Enough episodes exist to analyze score distributions and set a meaningful threshold.

---

## B5: Guardrails ADRs (Privacy + Integrity)

**What it was**: Two architectural decisions documented as ADRs in the system:
1. Privacy ADR: "No code leaves the machine during inference — all LLM calls go to localhost."
2. Integrity ADR: "SIL is forbidden from inventing NodeIDs; it must only reference NodeIDs present in the GraphPacket."

**Why parked**:
- These are already true by design — documenting them as ADRs adds zero code value right now.
- ADRs require the `synapses-intelligence` brain sidecar to be running (ADRs are stored in the brain, not in synapses). Not worth requiring the sidecar for documentation.

**Revisit when**: The brain sidecar is stable and running in production. Then formalise these as ADRs so future agents see them in `get_context` output.

---

## B6: Intelligence Sidecar Telemetry (metrics.json)

**What it was**: A local `metrics.json` file in `synapses-intelligence` logging:
- Time-to-First-Token (TTFT)
- Tokens/second throughput
- Token Compression Ratio (e.g., "500 nodes → 200 token brief")

**Why parked**:
- Belongs in `synapses-intelligence`, not in the main synapses MCP server.
- The intelligence sidecar is not yet stable enough for performance optimization to be the priority.

**Revisit when**: SIL fine-tuning (Kaggle notebooks) completes and the sidecar is running in daily use. Then add metrics to the `/health` endpoint and log to `~/.synapses/metrics.json`.

---

## B7: Cross-Project `recall()` (project_id=null)

**What it was**: `recall()` with `project_id=null` searches failure episodes across ALL projects — useful for teams working across multiple repos with shared patterns.

**Why parked**:
- Most users run Synapses against a single project. Cross-project recall is a multi-tenant feature.
- FTS5 doesn't partition by default — a cross-project search on a large corpus is expensive without proper indexing.

**Revisit when**: A user reports needing failure pattern reuse across projects. At that point, add a `project_id IS NULL` branch with a separate FTS5 table or partition.

---

## B8: 4-Layer Memory Hierarchy (Documentation) — ~~CLOSED~~

**Implemented**: Updated `synapses-os/.claude/CLAUDE.md` with the full 4-layer memory hierarchy table.

All 4 layers are live and documented:
- Layer 1 (Raw): `events` table — `get_events(since_seq=N)`, 24h rolling
- Layer 2 (Sessions): `session_state` table — `save_session_state` / `get_session_state`
- Layer 3 (Patterns): `episodes` table — `remember`, `recall`, `check_plan_safety`
- Layer 4 (Injected): `session_init` response — combines all of the above on session start

The CLAUDE.md also documents the Agent Message Bus (v0.7.0) and Brain Backends (llama-server / ollama / local) as of v0.7.0.

---

## B9: `brain setup` UX Polish

**What it is**: Add `--quiet` / `--verbose` flags to `brain setup` and print a clean "ready" summary at the end (model path, port, estimated embed speed). Currently all progress goes to raw stderr with no structured output.

**Why parked**: Functional correctness first. The setup command works; polish is low priority until the embedding backend has been used in real sessions.

**Revisit when**: First external user reports confusion about setup progress or wants scriptable output.

---

## B10: Batch Embedding — ~~CLOSED~~

**Implemented**: `embed.Client.EmbedBatch` in `synapses/internal/embed/client.go`; rewritten `embedAllNodes` in `synapses/cmd/synapses/main.go`.

- `EmbedBatch` sends up to 16 nodes per request: Brain format (`{"input": [texts]}` → `{"embeddings": [[...]...]}`), OpenAI format (`{"model":..., "input": [texts]}` → `{"data": [{"embedding":[...]}]}`), Ollama falls back to `embedSerial()` (sequential, Ollama has no batch endpoint).
- `embedAllNodes` chunks nodeIDs into batches of 16. On batch failure, falls back to per-node embedding with individual 10s timeouts. Error on a single node is logged and skipped (non-fatal).
- `synapses-intelligence` POST `/v1/embed` already supported batch input (`json.RawMessage` polymorphic — detects array vs single string). No server-side changes needed.

---

## B11: Embedding Persistence (vector cache in SQLite) — ~~CLOSED~~

**Implemented**: `node_embeddings` table with `content_hash` column in `synapses/internal/store/store.go` and `synapses/internal/store/embeddings.go`.

- Schema: `node_embeddings(node_id PK, model, embedding BLOB, content_hash TEXT, indexed_at INTEGER)`. `content_hash` added via `ALTER TABLE` migration for existing DBs.
- `nodeContentHash(name, sig, doc)` — `sha256(name+" "+sig+" "+doc)[:4]` hex, 8 chars. Stored at embed time; recomputed at query time.
- `UpsertEmbedding` queries the node's `name/signature/doc` internally and stores the hash alongside the vector.
- `GetNodesWithoutEmbeddings(limit)` — LEFT JOIN on `node_embeddings`, recomputes hash in Go, returns nodes where hash differs (stale) or where no embedding exists (missing). Both missing and stale nodes are returned in one query.
- Test: `synapses/internal/store/embeddings_stale_test.go` — embed node → verify not in missing list → update node doc directly in DB → verify appears as stale.

---

## B12: synapses-scout Integration into MCP Tool Registry

**What it is**: Wire `web_search` / `web_fetch` / `web_deep_search` directly into the synapses MCP server so agents don't need to configure the scout sidecar separately. synapses would proxy to `http://localhost:11436` when available, and omit the tools when scout is not running.

**Why parked**: Scout is currently a standalone sidecar that works fine when configured directly. Proxying adds a layer and duplicates the transport. Agents can already use scout via its own MCP config.

**Revisit when**: Multiple users ask why web tools are missing from synapses when scout is running. At that point the UX benefit outweighs the proxy complexity.

---

## B13: `prepare_context` Intent Router — ~~Verification & Completion~~ CLOSED

All 6 intents (`modify`, `understand`, `review`, `debug`, `add`, `plan`) are fully implemented in `synapses/internal/mcp/intents.go`. No gaps. Closed.
