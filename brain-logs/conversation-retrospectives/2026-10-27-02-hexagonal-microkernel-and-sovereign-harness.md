# Conversation Retrospective: Hexagonal Microkernel, OOP Philosophy & Sovereign Agent Harness

- **Date:** 2026-10-27
- **Topic:** Why Hexagonal Architecture (Ports & Adapters) and OOP are the foundation of Chotu, the relationship between an "Agent" and an "Agent Harness", the `platform/` vs. `inventory/` microkernel split, and sandbox isolation.

---

## 1. The Framework Question: Why Hexagonal OOP Beats Invasive AI Frameworks

Early in our discussion, you asked whether a framework like LangGraph or PydanticAI would be better for building Chotu, and then shared the engineering principles that matter most to you:

> *"I am a fan of ports and adapters and come from a world of OOP. So encapsulation, readability, type-safety and portability, observability, code cleanliness, naming and API design, sharing, and all enterprise-related functions including data optimization and a good persistence layer—all matter to me. I initially asked if a framework will help me do that. But framework loosely also means methodology, so explain to me in terms of which of these methods allow me to get to a flexible, extensible, secure, observable, and performant Chotu."*

### Why "Framework-First" Fails in Long-Lived Agent Systems
Most popular AI frameworks (LangChain, LangGraph, CrewAI) are **invasive frameworks**:
- They demand that your domain entities, state transitions, and tools inherit from their base classes (`StateGraph`, `BaseTool`, untyped `TypedDict` state blobs).
- When Chotu needs to dynamically register a new tool, onboard an external app, enforce custom citeability firewalls, or optimize how `pgvector` indexes facts, an invasive framework forces you to fight its internal abstractions.
- Worse, AI SDKs churn every few months. If your domain layer imports vendor or framework types, every upstream breaking change ripples through your entire codebase.

### The Resolution: Your Hexagonal Domain Architecture *Is* the Framework
We established that **Hexagonal Architecture (Ports & Adapters)** combined with **Microkernel (Plugin) Architecture** (codified in your [`dsa-prj/docs/standards/hexagonal.md`](file:///home/nox/Developer/ds2/dsa-prj/docs/standards/hexagonal.md) and [`code-structure.md`](file:///home/nox/Developer/ds2/dsa-prj/docs/standards/code-structure.md)) is the methodology:

1. **Zero Vendor SDKs in Domain Code**:
   - Every subsystem repeats `domain/ | ports/ | adapters/`.
   - `domain/` contains pure Python 3.12+ OOP entities, value objects, and domain services (`Pydantic v2`, `mypy --strict`). It never imports LiteLLM, Anthropic, Google GenAI, FastAPI, or database drivers.
2. **Ports as Strict Contracts (`<Thing>Port`)**:
   - Inbound and outbound boundaries are defined as explicit `Protocol` / `ABC` interfaces (`LLMPort`, `WorkClassifierPort`, `ServiceDiscoveryPort`, `ServiceLocatorPort`, `HookRegistryPort`, `TaskQueuePort`, `WorkflowStorePort`, `KnowledgeIndexPort`, `ObservabilityPort`).
   - Every port has a **port-contract test suite** that every adapter (`<Vendor><Thing>Adapter`) must pass before merging.
3. **Tiered Multi-Provider `LLMPort` Adapter**:
   - Libraries like `LiteLLM` / `PydanticAI` or native provider SDKs sit strictly inside `platform/brain/adapters/`, allowing Chotu to route fast classification/extraction tasks to local or flash models and complex System-2 architecture or coding tasks to frontier models via configuration alone.

---

## 2. Demystifying "Agent" vs. "Agent Harness": Why Chotu Must Be Self-Contained

When we compared ways for Chotu to execute subagents, skills, and code-building tasks, I initially suggested a hybrid option where Chotu might shell out to an external CLI coding agent (like `agy`) to build apps. You immediately pushed back:

> *"Why is Option 1 [100% Self-Contained In-Process Python Engine] not a recommended option? This is where I was asking you to clarify/educate me. Is Chotu looking more like harness engineering now? Have I misled what Chotu is because of my lack of knowledge?"*

Your pushback exposed a real architectural flaw in delegating to an external black-box CLI, and clarified what "harness engineering" actually means:

1. **What an "Agent Harness" Actually Is**:
   - A raw LLM is just a stateless text/JSON predictor. It cannot run a loop, read files, execute Python/bash, enforce permissions, wake up on a schedule, or spawn subagents.
   - The runtime software that wraps the model—the execution loop, progressive skill loader (`SKILL.md`), subagent spawner, tool dispatcher, approval bus, and observability tracer—is what the industry calls the **agent harness**.
   - Therefore, **building that runtime layer is not a detour from building an agent; it IS what makes Chotu an agent.**
2. **Why Delegating to an External Black-Box CLI Breaks Chotu's Purpose**:
   - If Chotu shells out to another CLI agent to do its work, Chotu becomes a hollow middleman around someone else's black-box runtime.
   - Inside that black box, you lose your Hexagonal Port guarantees, your OpenTelemetry trace tree, your 6-value `scope:` citeability firewall, and—most critically—**Chotu's ability to learn and crystallize new workflows**, because Chotu cannot see the intermediate reasoning steps, tool failures, or retrieval hits that happened inside the external CLI.
3. **Why Option 1 (100% Self-Contained In-Process Python Engine) Wins**:
   - In our Hexagonal architecture, a **Subagent**, a **Skill**, an **Internal Tool**, and an **External App Adapter** are first-class Python objects inside Chotu's own runtime.
   - Every cross-module call passes through two clean chokepoints—`AgentRuntime.invoke()` and `ToolRegistry.invoke()`—emitting structured OpenTelemetry spans so the entire semantic call stack is visible in the Portal's `TraceDrawer`.

---

## 3. The Two Trees (`platform/` vs. `inventory/`) and Workspace Jailing

### 3.1 `platform/` (The Chassis) vs. `inventory/` (The Content)
Following the doctrine from `dsa-prj`:
- **`backend/platform/` (The 9-Layer Chassis)**:
  - `brain/`, `tools/`, `orchestration/`, `agentic_loop/`, `harness/`, `memory/`, `knowledge/`, `environment/`, `multiagent/`.
  - Contains *mechanism only*—never hardcoded agent prompts, BAITS checklists, or domain tools. `platform/` **never** imports `inventory/`.
- **`backend/inventory/` (The Shipped Content of Chotu Corp)**:
  - Contains the lead orchestrator (`lead/chotu`), department subagents (`architects/`, `builders/`, `ingestion/`, `kb/`, `extractors/`, `classifiers/`), `workflows/`, `hooks/`, `tools/`, `skills/`, and `packs/` (`bvf/`, `raid/`).
  - Shipped inventory loads through the exact same registries as runtime-grown capabilities under `~/.chotu/`.

### 3.2 Closing the Filesystem Backdoor (Workspace Jailing)
During our loophole review, we identified a critical security risk: if a subagent has `python`, `bash`, or `grep` tools, a naive subprocess implementation would let that subagent read or write `CHOTU_VAULT_PATH` or another client's engagement folder (`workspaces/<other_slug>/`) directly on disk, bypassing the Knowledge Librarian and the 6-value `scope:` firewall.
- **The Safeguard**:
  - `platform/environment/` enforces a **Workspace Jail** on all code- and shell-executing tools (`python`, `bash`, `grep`, `file_ops`).
  - A delivery or extraction subagent running inside engagement `<slug>` is jailed strictly to `workspaces/<slug>/`.
  - Any attempt to read or write outside that boundary—including `CHOTU_VAULT_PATH` or other engagements—is blocked by both `pre_tool_invoke` policy hooks and OS path/sandbox containment. Only the `knowledge` department's custodial tools are permitted to access the Universal Knowledge Vault.
