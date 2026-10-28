# Conversation Retrospective: Classifier-First Routing, Subagents-Only Invariant, Dynamic Workflows & Lifecycle Hooks

- **Date:** 2026-10-27
- **Topic:** Solving the orchestration gaps in prior projects through an upfront Classifier Gate, a strict "Subagents-Only" execution invariant, dynamic Workflow DAG composition, and a dual-mode Lifecycle Hook engine.

---

## 1. The Missing Piece in Prior Projects

After reviewing the initial architecture blueprint, you pointed out a fundamental limitation across all of the previous repositories (`dsa`, `okf-kb-manager`, `dsa-companion-config`):

> *"One of the things all the other projects don't do is route/classify each work and then route them to subagents and subagents only. I don't think we use hooks or define dynamic workflows where an orchestrator composes the available agents to achieve a particular task."*

In those earlier systems:
- The lead agent often mixed conversation, planning, and direct tool execution inside one monolithic loop.
- Workflows in `dsa` were static, hand-authored YAML scripts (`build-agent`, `build-tool`) with a single integer `current_step` and no dynamic composition.
- Cross-cutting concerns (like loading context, enforcing citeability, marking synthesis views stale, or logging retrieval telemetry) had to be manually remembered inside individual skills or prompts because there was no lifecycle hook mechanism.

To make Chotu Corp truly modular and self-organizing, we designed four interlocking mechanisms:
1. **The Upfront Work Classifier Gate (`WorkClassifierPort`)**
2. **The Strict "Subagents-Only" Execution Invariant**
3. **Dynamic Subagent Workflow DAG Composition & Probationary Crystallization**
4. **The Dual-Mode Lifecycle Hook Engine (`HookRegistryPort`)**

---

## 2. Upfront Work Classification & The "Subagents-Only" Invariant

### 2.1 The Work Classifier Gate (`WorkClassifierPort`)
Every inbound piece of work—whether a message from Telegram/Portal, a file dropped into `raw/`, or a scheduled trigger—passes through `WorkClassifierPort` **before** any orchestration or subagent dispatch occurs.

Drawing on the taxonomy and calibrated checkpoint philosophy from your [`classifier-training`](file:///home/nox/Developer/ds2/classifier-training/CLAUDE.md) repository (Laya checkpoints, deterministic rules, and fast embeddings), the classifier produces a structured, non-prose decision object:

```python
WorkClassification(
    intent="ingest_and_synthesize_transcript",
    mode="work_engagement",          # work_engagement | personal_rd | universal_research
    engagement_slug="acme-cloud",
    departments=["knowledge", "delivery"],
    baits_pillars=["business", "security"],
    scope="client",                  # engagement | client | internal | reference | prior-project
    perspective="client_stated",
    complexity="standard_dag",       # fast_path_dag | dynamic_dag | novel_escalation
    risk_level="low",
)
```

### 2.2 The Hard Invariant: "The Lead Orchestrator Never Touches Tools"
We locked in a strict architectural rule enforced at the runtime port level:
- **The Lead Orchestrator (`lead/chotu`) NEVER executes raw domain tools (`python`, `bash`, `grep`, `vault_write`, etc.) or mutates engagement/vault state directly.**
- Infact, `lead/chotu`'s runtime context is not even wired to `ToolRegistry.invoke` for raw tools—it can **only** invoke `WorkClassifierPort`, `ServiceDiscoveryPort`, and `WorkflowDAGRunner` to dispatch **Subagents (`chotu://agent/...`)**.
- **All actual work happens inside scoped Department Subagents**, each running in its own isolated context window with only the tools, skills, and workspace jail permitted for its role.

---

## 3. Dynamic Workflow DAG Composition & Probationary Crystallization

Instead of static linear scripts, Chotu's Orchestrator acts as a **Dynamic Workflow Composer (`DynamicWorkflowComposer`)**:

### 3.1 Fast-Path Lookup vs. Dynamic DAG Composition
1. **Cached / Established DAG Lookup**:
   - Using the `WorkClassification` signature, the Orchestrator first queries `WorkflowRegistry`.
   - If an `ESTABLISHED` `WorkflowPlan` DAG already matches with high confidence and all its referenced Subagents are `HEALTHY`, Chotu instantiates that DAG immediately with **zero LLM planning overhead**.
2. **On-the-Fly Subagent DAG Composition**:
   - When the task is novel or requires a custom combination of perspectives/pillars, the Orchestrator queries `ServiceDiscoveryPort` for available Subagents and dynamically composes a typed **Directed Acyclic Graph (`WorkflowPlan`)** of subagent nodes:
     - **Sequential Edges**: `parse_transcript_agent` $\rightarrow$ `csa_extract_agent` $\rightarrow$ `concept_writer_agent`.
     - **Parallel Fan-Out / Fan-In**: Dispatching `business_architect_agent` $\parallel$ `security_architect_agent` concurrently on the extracted facts, then fanning their outputs into `csa_reviewer_agent`.
     - **Approval & Quality Gate Nodes**: Pausing a specific branch via `ApprovalBus` while independent branches continue executing.
3. **Adaptive Mid-Flight Splicing**:
   - If a Subagent node mid-DAG encounters a `CAPABILITY_MISS` or `NEEDS_REPLAN`, the Orchestrator can dynamically splice a `tool_builder`, `skill_builder`, or `csa_researcher` subagent node into the active DAG, resolve the gap, and resume the downstream nodes.

### 3.2 Closing the "Poisoned Fast-Path" Loophole: Probationary DAG Crystallization
During our loophole review, we noted that automatically saving every newly composed dynamic DAG into the permanent System-1 fast path after a single run could lock a fluke or brittle one-off workflow into Chotu's habits.
- **The Probationary Rule**:
  - A newly composed dynamic DAG is first recorded in `WorkflowRegistry` with `status = CANDIDATE`.
  - It only graduates to `status = ESTABLISHED` (eligible for zero-LLM System-1 fast-path routing) after **3 consecutive verified executions** (with zero citation, policy, or faithfulness gate failures) **OR explicit one-click promotion by you** in the Portal/Telegram.

---

## 4. The Dual-Mode Lifecycle Hook Engine (`HookRegistryPort`)

To keep subagents focused purely on their domain work without littering every agent with boilerplate code for context fetching, citeability checking, staleness tracking, and telemetry, Chotu introduces a declarative and programmatic **Lifecycle Hook Engine** across:
- `pre_classify` / `post_classify`
- `pre_workflow_compose` / `post_workflow_compose`
- `pre_subagent_dispatch` / `post_subagent_dispatch`
- `pre_tool_invoke` / `post_tool_invoke`
- `on_facts_persisted`
- `on_capability_miss`
- `on_workflow_complete`

### 4.1 Mode 1: Blocking Interceptor Hooks (Inline Guard & Mutator)
Run synchronously in the execution path and can **enrich inputs** or **block/retry completion**:
- **Auto-Context & Skill Injection (`pre_subagent_dispatch`)**: Automatically queries the engagement's local `pgvector` index (via the Librarian) and attaches the relevant `ContextPacket` and active BAITS capability-bundle skills (`bvf`, `raid`) to the subagent's input envelope.
- **Scope, Jail & Policy Enforcement (`pre_tool_invoke`)**: Verifies workspace path jailing and `PolicyEngine` rules before any tool executes.
- **Citeability & Anchor Verification Gate (`post_subagent_dispatch`)**: Verifies that synthesis views and deliverables respect the 6-value `scope:` firewall and contain valid `[[concept#^fact]]` anchors before marking the subagent node `COMPLETED`.

### 4.2 Mode 2: Async Background Observer Hooks (Zero-Latency Outbox)
Enqueued atomically to the `P2 outbox_hooks` lane of the Master Task Queue so they **never add latency** to your interactive turn:
- **OKF-to-`pgvector` Projection & Staleness Hook (`on_facts_persisted`)**: Whenever `okf-wiki/` concept files are written, triggers the OKF Indexer to update `pgvector` embeddings and marks affected pillars stale in `.csa/synthesis-status.md`.
- **Retrieval Learning & DAG Crystallization (`on_workflow_complete`)**: Scores which facts in the `ContextPacket` were actually cited by the subagent (updating `RetrievalTrace` weights) and increments the success counter on `CANDIDATE` workflow DAGs.

### 4.3 Closing the "Infinite Hook Recursion" Loophole
If an `on_workflow_complete` hook spawns a follow-up background workflow (e.g., background synthesis or research), that background workflow will itself complete and fire `on_workflow_complete`—risking an infinite storm of self-triggering workflows.
- **The Origin-Tagged Fence**: Every `WorkflowRun` carries `origin: USER | HOOK | SUBCONSCIOUS` and `hop_depth: int`. Observer hooks are only permitted to enqueue new workflows when `origin == USER` and `hop_depth == 0`. Workflows with `origin in (HOOK, SUBCONSCIOUS)` record telemetry only and never spawn recursive child workflows.
