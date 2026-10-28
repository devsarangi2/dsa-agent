# Conversation Retrospective: Real-World Consulting Scale, Hardened Postgres + `pgvector` Infrastructure & Master Task Queues

- **Date:** 2026-10-27
- **Topic:** Why "single user" does not mean "small data", the real daily volume of engagements and personal research, why hardened PostgreSQL + `pgvector` is mandatory Tier-0 infrastructure, per-node DAG state management, and the 4-lane Master Task Queue.

---

## 1. The Turning Point: Rejecting the "Single User = Small Data" Fallacy

When we discussed whether the persistence and vector recall layer should use Postgres or an embedded database like SQLite, I initially suggested SQLite (`WAL` + `sqlite-vec`) on the assumption that a single-user system has small data volume and low concurrency.

You stopped that assumption in its tracks and laid out the reality of your daily workload—exposing why previous iterations had suffered when hardening and failsafing were skipped:

> *"I think I do want to keep `pgvector` for datastore and vector recalls. You think that because I'm one user, this is small. But that's what was assumed before and failsafing and hardening was 'logically' skipped. No one, including you, looked at how much data/meetings I do, how many documents I need to read through every day, and how many docs I generate per day. To give you an idea, my typical engagement project consumes about 10 meetings with at least three different perspectives, client-side knowledge, internal knowledge, personal knowledge, personal research at work, personal research for my own learning at various levels every single day. Then I need to create master data per engagement I'm on and then have to capture learnings from this to the next engagement or another similar engagement 6 months or 1 year after. This is the level of data I need managed. So think about referencing 20 documents every day (transcripts, documentations, presentations, etc.), create artifacts or iterate on existing ones like diagrams, architectures, designs. Then consult/research internal repository, or web resources to identify patterns to make it fit to the work I'm doing. All of this only for the 'work' mode. Personal mode I need to learn techniques that 'might' help me with work, design tools for myself and so on."*

---

## 2. Quantifying the Real Workload & Why Enterprise Hardening Is Non-Negotiable

Let's record the actual scale that Chotu's infrastructure must sustain every single day without breaking a sweat:

| Dimension | Daily Volume | 6–12 Month Cumulative Scale |
| :--- | :--- | :--- |
| **Meetings & Multi-Perspective Capture** | **~10 meetings/day**, each carrying $\ge 3$ perspectives (`client`, `internal_team`, `personal_take`, plus `work_research` & `personal_rd`) | **2,500+ meeting transcripts** with 7,500+ perspective overlays |
| **Documents Referenced & Ingested** | **~20 dense documents/day** (slide decks, specs, SOWs, architecture docs, internal repo docs, web research) | **5,000+ multi-page documents** |
| **Extracted Claims, Facts & Citations** | **1,000–3,000+ atomic facts, claims, and citation anchors/day** across BAITS pillars and frameworks | **300,000 to 1,000,000+ embedded vectors** and graph edges |
| **Artifacts & Iterations** | Daily creation and versioned iteration of diagrams, architectures, BAITS syntheses, and deliverables | Thousands of versioned artifacts with point-in-time provenance |
| **Engagement Master Data** | Canonical per-engagement tables of Stakeholders, Systems, Capabilities, Decisions, Metrics, and RAID registers updated after every meeting | Dozens of complete, queryable **Engagement Datasets** spanning years |
| **Operating Modes** | Concurrent **Work Mode** (multiple client engagements) + **Personal Mode** (technique learning, tool design, R&D) | Continuous background extraction, indexing, HITS/Truth-Finder evaluation, and cross-engagement recall |

### Why Previous "Single-User" Shortcuts Failed
1. **Write Concurrency Collisions**: Ingesting 20 documents and 10 meeting transcripts per day while simultaneously iterating on diagrams, updating Engagement Master Data, running HITS/Truth-Finder, and chatting live means **dozens of concurrent database writers**. Any single-writer datastore (like SQLite) will lock up under that load.
2. **Recall Latency at 500k+ Vectors**: Without true HNSW (`pgvector`) and inverted GIN (`tsvector` / `pg_trgm`) indexes partitioned by scope and schema, recall across 6–12 months of engagements degrades from milliseconds to seconds.
3. **The "Optional DB / Skip-on-Failure" Anti-Pattern**: Because `dsa-companion-config` treated Postgres as an optional sidecar that exited `0` and skipped writes when unreachable, facts were silently lost whenever a container hiccuped.

---

## 3. Hardened PostgreSQL + `pgvector` Infrastructure Architecture

In Chotu, **PostgreSQL 16+ with `pgvector` (HNSW) and Full-Text Search (`tsvector` + `pg_trgm`) is mandatory, Tier-0 core infrastructure**:

1. **Schema-Level Isolation Inside a Hardened, Pooled Cluster**:
   - **`core.*` schema**: Platform registries (`CapabilityRegistry`, `WorkflowRegistry`, `HookRegistry`), the 4-lane `master_task_queue`, `workflow_runs` & `workflow_node_runs`, `approval_bus`, `policy_rules`, and OpenTelemetry trace spans.
   - **`universal.*` schema**: The disjoint Universal Knowledge Layer's `pgvector` + FTS index (`concepts`, `facts`, `pillar_sections`, `citations`, `elevated_learnings` with `engagement_id` correlation keys, and `retrieval_traces`).
   - **`personal.*` schema**: Dedicated workspace index for Personal Mode R&D, technique learning, and self-built tool experimentation.
   - **`eng_<slug>.*` per-engagement schemas**: Hard schema isolation for every client engagement (`concepts`, `facts`, `pillar_sections`, `citations`, **Engagement Master Data** tables for `stakeholders`, `systems`, `capabilities`, `decisions`, `metrics`, `raid_items`, plus `truth_finder_conflicts` and HITS scores).
2. **Zero Silent Data Loss (Transactional Outbox + Sync Ledger)**:
   - Every file landing in `raw/` or `okf-wiki/` is recorded in a durable ingestion/sync ledger.
   - If an embedding model or parser sidecar is temporarily busy or down, the task stays in the durable Postgres queue with exponential backoff, dead-letter protection, and live status visibility in the Web Portal—**never silently dropped**.

---

## 4. State Management & The 4-Lane Priority Master Task Queue

Inspecting `dsa/backend/dsa_platform/orchestration/workflow_runner.py` revealed that `dsa` previously used a single integer `current_step` and a shared mutable `run.state: dict` executed synchronously inside HTTP requests.

For Chotu's dynamic Subagent DAGs and heavy daily document throughput, we replaced that with two hardened subsystems:

### 4.1 Per-Node DAG State Machine (`WorkflowRun` + `WorkflowNodeRun`)
- **`WorkflowRun`**: Tracks the overall DAG instance (`id`, `classification`, `origin: USER | HOOK | SUBCONSCIOUS`, `hop_depth`, `status`, `created_at`).
- **`WorkflowNodeRun`**: Every subagent node in the DAG has its own database row and atomic state machine:
  `PENDING` $\rightarrow$ `READY` (all upstream parent nodes `COMPLETED`) $\rightarrow$ `QUEUED` $\rightarrow$ `RUNNING` $\rightarrow$ `AWAITING_APPROVAL` $\rightarrow$ `COMPLETED` | `FAILED` | `CANCELLED`.
- **Immutable Node Outputs (Zero Shared-Dict Race Conditions)**:
  - Parallel subagents (e.g., `business-architect` $\parallel$ `security-architect`) never mutate a shared dictionary.
  - Each node writes its own immutable, typed `NodeOutput` record (`output_payload` + `artifact_refs`) upon completion, and a database trigger / coordinator transitions downstream child nodes to `READY` as soon as their parent dependencies finish.

### 4.2 The 4-Lane Priority Master Task Queue (`FOR UPDATE SKIP LOCKED`)
All work in Chotu Corp flows through a durable, lease-based PostgreSQL Master Task Queue (with worker heartbeats, stale-lease reclamation, and token/concurrency budgets) organized into **four strict priority lanes**:

| Priority Lane | Purpose | Concurrency & Preemption Policy |
| :--- | :--- | :--- |
| **P0 — `interactive`** | Live user requests (Portal, Telegram, CLI) and human approval resumes | **Highest priority.** Reserved worker slots so a live request from you starts in `<50ms` even while 20 documents are ingesting. |
| **P1 — `workflow_dag`** | Ready `WorkflowNodeRun` subagent dispatches for active project deliverables and syntheses | High priority; executes parallel DAG branches across available worker slots. |
| **P2 — `outbox_hooks`** | Near-real-time lifecycle observer hooks: `okf-wiki/` $\rightarrow$ `pgvector` projection, HITS/Truth-Finder recalculation, staleness markers | Drained continuously in small micro-batches right after node completion. |
| **P3 — `subconscious_bulk`** | Bulk document/transcript parsing, Subconscious pattern recognition, Engagement Closure Retrospectives, Universal Knowledge research & hygiene | **Budget-throttled.** Yields capacity whenever P0/P1 work is present so background jobs never starve interactive work. |
