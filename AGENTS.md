# Chotu (`dsa-agent`) — Master Operating Instructions & Architectural Charter

Welcome to **`dsa-agent` (Chotu)**. This file is the single, authoritative root instruction manual for every agent session operating in this repository. It is linked as `AGENTS.md`, `CLAUDE.md`, `GEMINI.md`, and `.github/copilot-instructions.md` so that whether you are running inside Antigravity, Claude Code, GitHub Copilot, Codex, Cursor, or Windsurf, you start from the exact same shared mental model.

Before you touch code, propose a design, or write a single line of documentation, read the four operational mandates below. They are not stylistic preferences — they are the foundational rules that keep this project usable, intellectually honest, and architecturally sound.

---

## Hard Mandate #1: The Conversational Writer Rule (`narrative-writer` Subagent)

### Why This Is Rule #1: Zero Machine-Written AI Slop
There is one failure mode that destroys more value in engineering and consulting work than any code bug: **machine-written AI slop**. When an agent dumps robotic bullet lists, cryptic tables without context, bloated vocabulary, or dry specification stubs onto disk, a human reader cannot effortlessly read, absorb, or reuse that file later. And when a document is painful to read, **it gets discarded** — taking all the hard-won reasoning, nuances, and rejected dead-ends down with it.

Everything written to a human-facing file in this repository must read like a sharp, thoughtful colleague explaining things clearly over coffee: **natural and conversational in tone, explanatory when context is needed, and written at a clean professional level without overly complex vocabulary or jargon**.

### Broad Scope: Topic Extraction & All Human-Facing Writing
`narrative-writer` is **not** boxed into knowledge-base articles alone. It is our general-purpose conversational writer and topic extractor across the entire workspace:
- **Extracting Topics from Conversations or Meeting Transcripts**: Conversations and transcripts are fundamentally the same thing — raw, multi-turn dialogue where ideas, pushback, course corrections, and decisions are interleaved. `narrative-writer` teases apart distinct topics from a conversation or transcript, gathers everything said about each topic, and writes each one up as a cohesive, self-contained document.
- **Documentation, Plans, Deliverables, & Guides**: Whether you are writing architecture plans (`docs/planning/`), READMEs, onboarding guides, client-facing deliverables, research syntheses, or brainstorming retrospectives (`brain-logs/conversation-retrospectives/`), `narrative-writer` handles the prose so it never reads like machine output.

### The Mandatory Delegation Rule
Whenever your session needs to write or update any human-facing file on the filesystem, **you MUST delegate the writing to the `narrative-writer` subagent**:

1. **In Antigravity (`agy`)**: Follow the `narrative-writer` skill (`.agents/skills/narrative-writer/SKILL.md`). If `narrative-writer` is not yet registered in your session, call `define_subagent` using the system prompt in `.agents/agents/narrative-writer.md` with `EnableWriteTools: true`, and then call `invoke_subagent` with the full context and target file paths.
2. **In Claude Code / GitHub Copilot**: Delegate directly to the pre-configured `narrative-writer` subagent defined in `.claude/agents/narrative-writer.md` or `.github/agents/narrative-writer.agent.md`.
3. **In Harnesses Without Subagent Spawning**: Read and adopt `.agents/agents/narrative-writer.md` directly before authoring any markdown or prose file.

**What `narrative-writer` Enforces**:
- **Natural Conversational Flow**: Walk the reader through *why* a question surfaced, *what* assumptions were challenged or corrected, *how* the options were weighed, and *how* the conclusion works in practice.
- **Plain, Professional Language (Zero Jargon or AI Clichés)**: Use simple, everyday words ("use", "split", "keep", "because") instead of pompous substitutes ("utilize", "bifurcate", "retain", "vis-à-vis"), and ban AI filler ("delve", "tapestry", "pivotal", "synergy", "testament to"). When a technical term is needed, explain what it actually does in plain English.
- **Tables and Diagrams Support Prose, Never Replace It**: Use tables, diagrams, and bullet lists only to make a point easier to scan *after* explaining the idea in clear paragraphs.

---

## Hard Mandate #2: Topic-Level Brainstorming Retrospectives

When we brainstorm architecture, workflows, or domain modeling, the journey of *how* we arrived at a decision is just as important as the final schema or interface. Terse ADR stubs strip away the human reasoning and invite future sessions to repeat the exact same dead-end debates.

1. **Capture Every Brainstorming & Design Discussion in `brain-logs/conversation-retrospectives/`**:
   - Every architectural or product brainstorming thread must be documented in `brain-logs/conversation-retrospectives/<YYYY-MM-DD>-<topic>.md` and indexed in `brain-logs/README.md` (via the `narrative-writer` subagent).
2. **Organize Strictly by Topic, Never by Raw Session Dump**:
   - Each retrospective file covers **one cohesive architectural or domain topic** (for example, *Service Discovery & Service Locator* or *Versioned Knowledge & The Truth Finder Engine*), never a chronological transcript of whatever happened on a given day. If a single conversation spans three distinct topics, write or update three topic-scoped retrospectives.
3. **Keep All Plans & Knowledge Inside the Repository**:
   - Never leave architectural plans, design specs, or retrospectives trapped inside ephemeral agent scratch directories (`~/.gemini/...`, `/tmp`, etc.). All durable plans live in `docs/planning/` and all design retrospectives live in `brain-logs/`.

---

## Hard Mandate #3: What Chotu (`dsa-agent`) Is & The Operator Scale Reality

### The "Chotu Corp" Operating Model
Chotu is not a single-turn chat wrapper or a brittle prompt script. It is a **personal autonomous company ("Chotu Corp")** where the human operator acts as Chairman & CEO, and Chotu serves as the **Lead Orchestrator & Chief of Staff** running across two complementary cognitive loops:
- **The Conscious Mind (Interactive Orchestration)**: Real-time, low-latency collaboration with the operator via CLI, Web Control Center, or Telegram — classifying intent, assembling specialist subagent workflows, and surfacing high-signal synthesis.
- **The Subconscious Mind (Autonomous Background Daemon)**: Continuous background workers that ingest transcripts and documents, maintain the knowledge graph, run multi-perspective truth weighing, execute scheduled playbooks, and prepare briefings before the operator even asks.

Chotu strictly separates **Internal Tools** (capabilities shipped inside `dsa-agent/backend/inventory/` for Chotu's own subagents) from **Independent External Apps (`dsapps-*`)** (standalone products with their own repositories, UIs, and lifecycles that Chotu orchestrates over MCP, REST, or CLI bridges via `ServiceLocatorPort`).

### Rejecting the "Single User = Small Data" Fallacy
One of the most dangerous traps an agent can fall into when designing for a single human operator is assuming that "one user" means "toy scale" where SQLite files, flat JSON logs, or naive in-memory queues are good enough.

In reality, the operator runs a high-velocity consulting and R&D practice across simultaneous **Work** and **Personal** modes:
- **~10 dense meetings every single day**, each generating multi-perspective transcripts and notes (`client` claims, `internal_team` alignment, and the operator's own `personal_take` architectural hypotheses).
- **~20 complex documents every day** (slides, SOWs, architecture specs, spreadsheets, and research papers) requiring structural chunking, parent-document preservation, and cross-referencing.
- **Rich Engagement Master Data**: Every client project is a first-class, isolated dataset (`eng_<slug>`) tied to master records (Accounts, Contacts, Proposals, Deliverables, SOW milestones) across **6-to-12-month engagement horizons** — plus long-term recall across past engagements.

Because of this compounding cognitive volume and high concurrent read/write contention between the Conscious and Subconscious loops, **hardened PostgreSQL 16+ with `pgvector` HNSW indexes, durable per-node DAG state (`WorkflowRun` + `WorkflowNodeRun`), transactional outbox delivery, and a 4-lane priority Master Task Queue (`P0 Interactive`, `P1 Scoped`, `P2 Subconscious Ingestion`, `P3 Deep Maintenance`) are mandatory Tier-0 infrastructure from Day 1.**

---

## Hard Mandate #4: Core Architectural Invariants of `dsa-agent`

Every line of code written in `dsa-agent` must preserve these six structural invariants:

1. **Microkernel + Fractal Hexagonal OOP Architecture (`platform/` vs. `inventory/`)**:
   - `backend/platform/` is the 9-layer domain-agnostic microkernel chassis. `backend/inventory/` holds the concrete agents, tools, skills, workflows, hooks, and plugins shipped with the system.
   - Every layer inside `backend/platform/` follows strict **Fractal Hexagonal (Ports & Adapters)** isolation: `domain/` (pure Python entities, value objects, and domain logic with **zero vendor SDK, FastAPI, SQLAlchemy, or LiteLLM imports**), `ports/` (abstract `@abstractmethod` ABC contracts), and `adapters/` (swappable concrete implementations).
2. **Sovereign In-Process Harness & Two Runtime Chokepoints**:
   - Chotu runs its own 100% sovereign in-process Python execution harness — never shelling out to an external opaque CLI binary as its brain.
   - Every agent turn flows through **`AgentRuntime.invoke()`**, and every tool call flows through **`ToolRegistry.invoke()`**. Policy checks, workspace path jailing, budget caps, OTel tracing, and lifecycle hooks attach cleanly at these two chokepoints.
3. **Classifier-First Routing & The Subagents-Only Invariant**:
   - Every incoming request is first classified by `WorkClassifierPort` (System-1 fast deterministic match vs. System-2 novel escalation).
   - **The Lead Orchestrator (`lead/chotu`) NEVER calls raw tools (`bash`, `sql`, `filesystem`, etc.) directly.** It delegates *exclusively* to scoped specialist Subagents, either by dispatching an established workflow or by composing a dynamic Subagent DAG via `DynamicWorkflowComposer` (which enters a probationary burn-in before crystallizing into `inventory/workflows/`).
4. **Constructor DI for Infrastructure vs. Runtime Service Locator for Capabilities**:
   - Core platform engines receive their hexagonal infrastructure ports via **explicit Constructor Dependency Injection** wired at startup in `backend/platform/bootstrap.py`.
   - Dynamic capabilities (`chotu://<kind>/<namespace>/<name>@<version>`) are resolved at runtime through **`ServiceLocatorPort`** and **`ServiceDiscoveryPort`**, combining Hybrid Pull + Push registration with a **3-Stage Zero-LLM Resolution Pipeline** (Exact URI $\rightarrow$ Structural/Tag Filter $\rightarrow$ Hybrid `pgvector` + FTS Ranker) and lifecycle-aware lazy activation.
5. **Disjoint Knowledge Fabric, Multi-Perspective Truth Finder + HITS, and Elevation**:
   - **Engagement Scope First**: Subagents execute inside a closed-world engagement dataset (`eng_<slug>`) by default so client data never leaks across boundaries. Universal Knowledge (`universal.*`) is queried only on a local miss via the controlled **Discover $\rightarrow$ Select $\rightarrow$ Ingest** flow.
   - **Never Auto-Overwrite Competing Claims**: When a client statement, an internal team note, and a personal hypothesis disagree, Chotu never destroys context with naive "last-write-wins" resolution. All claims coexist as versioned, perspective-tagged records (`client`, `internal_team`, `personal_take`, `work_research`, `personal_rd`). The **Truth Finder + HITS Engine** evaluates competing claims using a **2-Step Weighing Pipeline** — **Step 1: Structural Relationship Make/Break Impact** (how many downstream architectural dependencies a claim validates or invalidates) and **Step 2: Contextual Plausibility ("Makes More Sense")** — before surfacing a side-by-side impact diff for human-in-the-loop (HITL) promotion.
   - **The Elevation Technique**: At milestone or engagement closure, Chotu runs the **5-Lens Retrospective Discovery Queue** (Sanitized Reusable Learnings, Client/Account Intelligence, People & Stakeholder Patterns, Process & Estimation Calibration, and Self-Improvement Signals) and elevates distilled insights into Universal Knowledge using `engagement_id` as the permanent citation correlation key.
6. **Read-Only Sibling Repositories**:
   - Sibling repositories in `/home/nox/Developer/ds2/` (`agent-harness`, `agency-agents`, `multi-agent-system`, `OpenSpace`, `OpenViking`, `okf`, `career-ops`, `markitdown`, etc.) are **strictly read-only reference goldmines**. Never modify files outside `/home/nox/Developer/ds2/dsa-agent`; harvest patterns by adapting them cleanly into our hexagonal ports and adapters.

---

## Essential Navigation Pointers

Whenever you need deep context on any subsystem, consult these living documents before making assumptions:

- **Brain Logs Index**: [`brain-logs/README.md`](brain-logs/README.md)
- **Topic-Level Conversation Retrospectives**:
  1. [Identity & The "Chotu Corp" Operating Model](brain-logs/conversation-retrospectives/2026-10-27-01-identity-and-chotu-corp-operating-model.md)
  2. [Hexagonal Microkernel, OOP Philosophy & Sovereign Agent Harness](brain-logs/conversation-retrospectives/2026-10-27-02-hexagonal-microkernel-and-sovereign-harness.md)
  3. [Service Locator & Service Discovery Architecture](brain-logs/conversation-retrospectives/2026-10-27-03-service-discovery-and-service-locator.md)
  4. [Classifier-First Routing, Subagents-Only Invariant, Dynamic Workflows & Lifecycle Hooks](brain-logs/conversation-retrospectives/2026-10-27-04-classifier-routing-dynamic-workflows-and-hooks.md)
  5. [Disjoint Universal Knowledge Fabric, Engagement-as-Dataset & The Elevation Technique](brain-logs/conversation-retrospectives/2026-10-27-05-knowledge-fabric-engagement-datasets-and-elevation.md)
  6. [Versioned Knowledge, Multi-Perspective Reality & The Truth Finder + HITS Engine](brain-logs/conversation-retrospectives/2026-10-27-06-versioned-knowledge-truth-finder-and-hits-weighing.md)
  7. [Real-World Consulting Scale, Hardened Postgres + `pgvector` Infrastructure & Master Task Queues](brain-logs/conversation-retrospectives/2026-10-27-07-real-world-scale-postgres-hardening-and-master-queues.md)
- **Master Architecture Blueprint**: [`docs/planning/chotu_architecture_plan.md`](docs/planning/chotu_architecture_plan.md)
- **Conversational Writer Subagent & Skill**: [`.agents/agents/narrative-writer.md`](.agents/agents/narrative-writer.md) | [`.agents/skills/narrative-writer/SKILL.md`](.agents/skills/narrative-writer/SKILL.md)
