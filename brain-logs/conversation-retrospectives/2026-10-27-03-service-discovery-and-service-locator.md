# Conversation Retrospective: Service Locator & Service Discovery Architecture

- **Date:** 2026-10-27
- **Topic:** Designing Service Locator and Service Discovery for an expansive, self-extending agentic enterprise without violating Hexagonal OOP principles.

---

## 1. The Catalyst: Why Chotu Corp Needs Service Discovery & Location

As we mapped out Chotu Corp's departments, subagents, skills, internal tools, sidecars (Docling, local embedders), and independent external apps (`dsapps-checkin`, `dsapps-journal`, project management apps), you raised a key architectural requirement:

> *"Because Chotu Corp is expansive and extendible, we need to have a Service Locator and a Service Discovery pattern. Anything you want to add to this thought?"*

In a system that grows over time—where new tools are built at runtime, new capability-bundles (`bvf`, `raid`) are operationalized, and external applications come online or go to sleep—hardcoding imports or tool lists inside agents is a dead end. Awareness must be a **runtime registry query**, never hardcoding.

---

## 2. Resolving the Classic OOP Tension: Constructor DI vs. Service Locator

In enterprise OOP and Hexagonal Architecture, a naive **Global Static Service Locator** (where domain classes call `ServiceLocator.get(DatabasePort)` hidden deep inside business methods) is an anti-pattern because it hides class dependencies and makes unit testing brittle.

We resolved this cleanly by drawing a **strict two-plane boundary**:

1. **Plane 1 — Core Platform Infrastructure (Constructor Dependency Injection)**:
   - Platform ports (`LLMPort`, `WorkflowStorePort`, `TaskQueuePort`, `KnowledgeIndexPort`, `ObservabilityPort`) are wired via **Constructor Dependency Injection** at the Composition Root (`harness/bootstrap.py`).
   - Domain and application services declare their required Ports explicitly in their `__init__` constructors, keeping unit and contract testing pure and explicit.
2. **Plane 2 — Runtime Capabilities of Chotu Corp (`ServiceDiscoveryPort` & `ServiceLocatorPort`)**:
   - Dynamic capabilities—**Subagents, Skills/Playbooks, Internal Tools, Capability-Bundles, Sidecars, and External Apps**—live on the Runtime Capability Plane behind the two invocation chokepoints (`AgentRuntime.invoke` and `ToolRegistry.invoke`).
   - Here, Service Discovery ("what healthy capabilities can solve this intent?") and Service Locator ("resolve `chotu://skill/baits/make-raid@1.0` to its executable adapter") are essential first-class patterns.

---

## 3. Canonical Capability URI & Descriptor Contract

Every invocable capability across Chotu Corp registers a strongly-typed `CapabilityDescriptor` addressed by a canonical URI:

```text
chotu://<kind>/<namespace>/<name>@<version>
```

Examples:
- `chotu://agent/architects/business@1.0.0` (Department Subagent)
- `chotu://skill/baits/make-raid@1.2.0` (Operationalized Capability-Bundle Skill)
- `chotu://tool/core/grep@1.0.0` (Internal Tool)
- `chotu://app/external/dsapps-journal@2.1.0` (Onboarded External App)
- `chotu://sidecar/ingestion/docling@1.0.0` (Out-of-process Sidecar)

Each `CapabilityDescriptor` declares:
- **`uri` & `aliases` / `trigger_phrases`**: For exact $O(1)$ location.
- **`department` & `baits_pillar`**: Organizational and ontological ownership (`executive`, `knowledge`, `delivery`, `engineering`, `intelligence`; `business`, `application`, `information`, `technology`, `security`).
- **`io_contract`**: Typed JSON Schema for inputs and outputs + `applies_to_kinds` (e.g., `[transcript, pdf]`).
- **`permissions` & `risk_level`**: Required scopes and side-effect classification enforced by `PolicyEngine`.
- **`lifecycle_policy`**: `always_on` | `lazy_start` | `external_managed`.
- **`fallback_uri`**: Optional graceful degradation target if the primary capability is unavailable.

---

## 4. Hybrid Registration (Pull + Push)

To support both file-based plugins (`inventory/` and `~/.chotu/`) and running external network applications (`dsapps-*`, MCP servers, sidecars), Chotu uses **Hybrid Registration**:

1. **Pull Registration (File-Watcher & Manifest Reconciliation)**:
   - At startup and via a live filesystem watcher, the registry scans `backend/inventory/` (shipped capabilities) and `~/.chotu/{agents,tools,skills,apps}/` (user-grown and onboarded capabilities) for `manifest.yaml` and `SKILL.md` descriptors.
   - **Fail-Loud Reconciliation**: If a manifest or grown tool fails schema or contract validation at startup, the registry marks it `DISABLED` with a recorded diagnostic reason rather than crashing mid-turn.
2. **Push Registration (Self-Registration & Heartbeat API)**:
   - External apps (`dsapps-checkin`, `dsapps-journal`), MCP servers, and sidecars can dynamically announce themselves when they boot via `POST /api/v1/registry/register` and maintain liveness via `/api/v1/registry/heartbeat`.

---

## 5. The 3-Stage Zero-LLM Resolution Pipeline

When the Orchestrator or a Subagent needs to locate or discover capabilities, it does **not** stuff the entire catalog into an expensive LLM prompt. Instead, it runs a fast, deterministic **3-Stage Resolution Pipeline**:

1. **Stage 1 — $O(1)$ Exact Locator (`locate(uri_or_trigger)`)**:
   - When an Established Workflow DAG or user command specifies a known URI (`chotu://skill/baits/make-raid`) or exact trigger phrase (`csa ingest`), the Service Locator resolves the adapter in `<1ms` via hash lookup.
2. **Stage 2 — Faceted Hard-Constraint Filter**:
   - Filters candidates by:
     - `health_status == HEALTHY` (or `SLEEPING` if `lifecycle_policy == lazy_start`)
     - Caller's `allowlist ∩ PolicyEngine` rules
     - Target `department`, `baits_pillar`, and `io_contract` compatibility.
3. **Stage 3 — Hybrid BM25 + Vector + Historical Reliability Ranking (`discover(intent)`)**:
   - Ranks matching candidates using a weighted combination of full-text keyword match, cosine similarity over capability descriptions (`pgvector`), and **historical execution success rate** (how reliably this capability succeeded on similar intents in the past).
   - **Capability-Miss Ledger Integration**: If no candidate exceeds the calibrated confidence threshold, Service Discovery automatically records a structured entry in the **Capability-Miss Ledger** and signals the Orchestrator to escalate to System-2 (e.g., splicing in `tool_builder` / `skill_builder` or consulting the LLM Architect).

---

## 6. Lifecycle-Aware Activation, Lazy Start & Fallback Chains

Because external apps and sidecars (like Docling or specialized local services) should not have to burn workstation RAM 24/7 when idle, the Service Locator is **lifecycle-aware**:

- **`always_on`**: In-process Python subagents, skills, and tools—invoked immediately.
- **`lazy_start`**: Declares a `start_command` (or Docker Compose profile), `health_endpoint`, and `idle_timeout_s`. When located while `SLEEPING`, the Service Locator automatically starts the service on demand, awaits a healthy readiness probe, executes the call, and shuts it down after the idle timeout.
- **`external_managed` + `fallback_uri` + Circuit Breaker**: If an externally managed service fails its health check or times out, the circuit breaker trips and transparently routes the invocation to its declared `fallback_uri` (for example, falling back from the Docling container sidecar to a local Python PDF parser) before failing the workflow node.
