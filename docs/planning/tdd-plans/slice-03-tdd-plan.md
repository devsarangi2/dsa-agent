# Slice Group 3: Chotu Corp Inventory, BAITS Architects, Capability-Bundles, Extractors & Subconscious Daemon Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Populate `backend/inventory/` with Chotu Corp's Lead Orchestrator (`lead/chotu`), BAITS specialist architects, ingestion/KB custodial subagents, multi-format document/transcript extractors, operationalized `csa-*` skills, zero-code capability-bundle packs (`bvf/`, `raid/`, `baits/`), the 3-tier Subconscious background daemon, multi-channel comms adapters, and the `ExternalAppAdapter` bridge for independent external applications (`dsapps-*`).

**Architecture:** `backend/inventory/` ships as declarative content (`manifest.yaml`, `SKILL.md`, `pack.yaml`, and pure domain handlers) registered into the Slice 1 microkernel (`ServiceDiscoveryPort` / `ServiceLocatorPort`) without a single upward import from `backend/platform/` to `backend/inventory/`. Every inbound user turn, file drop, or scheduled wake-up passes through `WorkClassifierPort` into `LeadOrchestrator` (which strictly obeys `allowed_tools: []` and delegates exclusively to subagents via `WorkflowDAGRunner`), while `SubconsciousScheduler` drives background ingestion, HITS/Truth-Finder weighing, synthesis refresh, and the Morning Executive Briefing on the `P2` and `P3` queue lanes behind an origin-tagged recursion fence.

**Tech Stack:** Python 3.12+, `pydantic>=2.9`, `pyyaml`, `httpx` (Docling sidecar + REST external apps), `pymupdf` / `python-docx` / `python-pptx` / `openpyxl` (local extractor fallbacks), `watchfiles` (Obsidian/workspace watcher), `fastapi` + `sse-starlette`, `pytest` + `pytest-asyncio`, `mypy --strict`.

**Spec:** [`docs/planning/slice-03-chotu-corp-inventory-and-subconscious-loop.md`](file:///home/nox/Developer/ds2/dsa-agent/docs/planning/slice-03-chotu-corp-inventory-and-subconscious-loop.md)

## Global Constraints

- **Zero Upward Imports (`platform/` $\not\rightarrow$ `inventory/`)**: `backend/platform/` must contain zero imports from `backend/inventory/` (enforced by AST/grep test in Task 1).
- **Strict Fractal Hexagonal Boundaries**: Zero vendor SDK imports (`litellm`, `fastapi`, `sqlalchemy`, `httpx`, `pymupdf`, `telegram`) inside any `domain/` package across `backend/platform/` or `backend/inventory/`.
- **Subagents-Only Invariant**: `chotu://agent/lead/chotu@1.0.0` MUST declare `allowed_tools: []` and raise `SubagentOnlyViolationError` if any code path attempts to invoke `ToolRegistryPort.invoke()` under `lead/chotu`'s execution context.
- **Canonical Capability URI Contract**: Every agent, skill, extractor, pack, and external app registers under `chotu://<kind>/<namespace>/<name>@<version>` with explicit JSON Schema `input_schema` and `output_schema`, `department`, `baits_pillars`, `lifecycle_policy` (`always_on` | `lazy_start` | `external_managed`), and optional `fallback_uri`.
- **6-Value Scope & 5-Value Perspective Preservation**: Every extracted claim must carry both `scope` (`engagement` | `client` | `internal` | `reference` | `prior-project` | `deliverable`) and `perspective` (`client_stated` | `internal_team` | `personal_take` | `work_research` | `personal_rd`). Competing claims never overwrite one another in `okf-wiki/`.
- **Origin-Tagged Recursion Fence**: Background workflows spawned by hooks (`origin="HOOK"`) or the Subconscious daemon (`origin="SUBCONSCIOUS"`) carry `hop_depth >= 1` and are prohibited from spawning recursive child workflows on completion.

## Review Focus

1. **`lead/chotu` Direct Tool Execution Bypass**: If `LeadOrchestrator` receives a prompt asking for a quick bash command or file read and attempts to call `ToolRegistryPort.invoke("chotu://tool/builtin/bash@1.0.0", ...)` directly instead of dispatching a subagent DAG, the runtime must immediately raise `SubagentOnlyViolationError` (pinned in **Task 1**: `test_lead_chotu_blocks_direct_raw_tool_invocation`).
2. **Scope Citeability Firewall Leak in `csa-reviewer`**: When `csa-reviewer` audits a client-facing deliverable (`target_scope="deliverable"`) that cites a fact tagged `scope="internal"` or `perspective="personal_take"`, it must reject the node output with a `CiteabilityFirewallError` listing the offending `[[concept#^f-id]]` anchor rather than letting internal notes leak to the client (pinned in **Task 2**: `test_csa_reviewer_rejects_internal_or_personal_take_fact_in_deliverable`).
3. **Multi-Topic & Multi-Perspective Transcript Flattening + Docling Sidecar Outage**: When a single meeting transcript contains 3 interleaved topics and statements from a Client VP (`client_stated`), an Internal Architect (`internal_team`), and the operator's post-call memo (`personal_take`), `parse-transcript` must split the dialogue into distinct topic segments while preserving per-speaker perspectives; and when the Docling HTTP sidecar is unreachable (`ConnectError` / `TimeoutException`), `parse-pdf` must seamlessly fall back to its `fallback_uri` (`parse-pdf-local`) without dropping the ingestion job (pinned in **Task 3**: `test_parse_transcript_preserves_three_topics_and_three_perspectives` and `test_parse_pdf_falls_back_to_local_when_docling_sidecar_times_out`).
4. **Silent Overwrite of Competing Claims During Concept Writing**: When `csa-concept-writer` processes a new `client_stated` claim ("Legacy Billing decommissions in Q2 2027") that contradicts an existing `internal_team` claim ("Legacy Billing decommissions in Q4 2027") on the same concept, it must append the new versioned `^f-id` anchor under the concept's `## Application` section and emit a `ConflictDetectedEvent` for `truth_finder_agent` rather than overwriting the existing line (pinned in **Task 4**: `test_concept_writer_preserves_competing_perspectives_and_emits_conflict`).
5. **Malformed Capability-Bundle `pack.yaml` or Subconscious P3 Starving P0 Interactive Turns / External App Timeout**: If a drop-in capability pack in `inventory/packs/` has an invalid BAITS pillar name or missing rubric template in `pack.yaml`, `PackLoader` must raise `InvalidPackManifestError` at registration time (pinned in **Task 5**: `test_pack_loader_rejects_invalid_pack_yaml_schema`); if `P0 interactive` tasks arrive while `SubconsciousScheduler` is running a `P3` batch, `SubconsciousScheduler.tick_p3()` must immediately yield without dequeuing further `P3` items (pinned in **Task 6**: `test_subconscious_p3_yields_when_p0_interactive_queue_is_non_empty`); and if `dsapps-journal` times out 3 consecutive times, `ExternalAppAdapter` must trip its circuit breaker to `OPEN` and return a structured `ExternalAppUnavailableResult` without hanging the calling workflow (pinned in **Task 7**: `test_external_app_adapter_trips_circuit_breaker_on_timeout`).

---

### Task 1: Inventory Manifest Loader & `LeadOrchestrator` (`inventory/agents/lead/chotu/`)

Before any specialist department can run, Chotu needs a clean mechanism to load declarative `manifest.yaml` definitions from `backend/inventory/` into `ServiceDiscoveryPort`, and it needs its CEO/Chief of Staff orchestrator (`lead/chotu`) wired to classify incoming work and compose or look up Subagent DAGs. We keep `InventoryBootstrapper` inside `backend/platform/multiagent/adapters/inventory_loader.py` as a filesystem scanner that reads standard `manifest.yaml` files from any directory path passed to it—meaning `backend/platform/` never imports a single Python module from `backend/inventory/`.

Inside `backend/inventory/agents/lead/chotu/`, `LeadOrchestrator` implements the executive routing loop: it classifies every inbound turn via `WorkClassifierPort`, checks `WorkflowRegistryPort` for an `ESTABLISHED` fast-path DAG (System-1), and falls back to `DynamicWorkflowComposerPort` (System-2) when the request is novel or requires a custom multi-pillar fan-out. The tests in this task verify three critical invariants: zero upward imports from `platform/` to `inventory/`, System-1 vs. System-2 DAG routing, and the hard `SubagentOnlyViolationError` guard whenever `lead/chotu` attempts to touch a raw tool directly.

**Files:**
- Create: `backend/platform/multiagent/adapters/inventory_loader.py`
- Create: `backend/inventory/agents/lead/chotu/manifest.yaml`
- Create: `backend/inventory/agents/lead/chotu/orchestrator.py`
- Create: `backend/inventory/classifiers/work_classifier.py`
- Test: `backend/tests/inventory/test_lead_chotu_orchestrator.py`

**Interfaces:**
- Consumes (from Slice 1 `backend/platform/`):
  - `WorkClassifierPort.classify(request: InboundWorkRequest) -> WorkClassification`
  - `ServiceDiscoveryPort.register(descriptor: CapabilityDescriptor) -> None`
  - `ServiceLocatorPort.resolve(uri: str) -> CapabilityDescriptor`
  - `WorkflowDAGRunnerPort.submit_plan(plan: WorkflowPlan, origin: Literal["USER", "HOOK", "SUBCONSCIOUS"], hop_depth: int = 0) -> WorkflowRun`
  - `ToolRegistryPort.invoke(caller_agent_uri: str, tool_uri: str, payload: dict[str, Any]) -> dict[str, Any]`
- Produces:
  - `InventoryManifestLoader.scan_and_register(inventory_root: Path, discovery: ServiceDiscoveryPort) -> list[CapabilityDescriptor]`
  - `LeadOrchestrator.handle_turn(request: InboundWorkRequest) -> WorkflowRun`
  - `RuleAndCheckpointWorkClassifier.classify(request: InboundWorkRequest) -> WorkClassification`
  - `SubagentOnlyViolationError(PermissionError)`

- [ ] **Step 1: Write the failing test**

Create `backend/tests/inventory/test_lead_chotu_orchestrator.py` testing: (a) AST check that `backend/platform/` has zero imports from `backend/inventory/`, (b) System-1 established workflow lookup vs. System-2 dynamic BAITS fan-out composition, and (c) Review Focus #1 (`SubagentOnlyViolationError` when `lead/chotu` attempts direct tool invocation):

```python
from __future__ import annotations

import ast
from pathlib import Path
import pytest

from backend.inventory.agents.lead.chotu.orchestrator import (
    LeadOrchestrator,
    SubagentOnlyViolationError,
)
from backend.inventory.classifiers.work_classifier import RuleAndCheckpointWorkClassifier
from backend.platform.multiagent.adapters.inventory_loader import InventoryManifestLoader
from backend.platform.orchestration.domain.models import (
    InboundWorkRequest,
    WorkflowStatus,
)


def test_platform_never_imports_inventory() -> None:
    platform_root = Path(__file__).resolve().parents[2] / "platform"
    offenders: list[str] = []
    for py_file in platform_root.rglob("*.py"):
        tree = ast.parse(py_file.read_text(encoding="utf-8"), filename=str(py_file))
        for node in ast.walk(tree):
            if isinstance(node, ast.Import):
                for alias in node.names:
                    if "inventory" in alias.name.split("."):
                        offenders.append(f"{py_file}:{node.lineno} -> {alias.name}")
            elif isinstance(node, ast.ImportFrom) and node.module:
                if "inventory" in node.module.split("."):
                    offenders.append(f"{py_file}:{node.lineno} -> {node.module}")
    assert offenders == [], f"Forbidden upward imports from platform to inventory: {offenders}"


@pytest.mark.asyncio
async def test_lead_chotu_routes_established_vs_dynamic_dag(
    in_memory_discovery,
    in_memory_workflow_registry,
    fake_dag_runner,
    inventory_root: Path,
) -> None:
    loader = InventoryManifestLoader()
    descriptors = await loader.scan_and_register(inventory_root, in_memory_discovery)
    uris = {d.uri for d in descriptors}
    assert "chotu://agent/lead/chotu@1.0.0" in uris

    classifier = RuleAndCheckpointWorkClassifier()
    orchestrator = LeadOrchestrator(
        classifier=classifier,
        discovery=in_memory_discovery,
        workflow_registry=in_memory_workflow_registry,
        dag_runner=fake_dag_runner,
    )

    # Novel multi-pillar architectural review -> System-2 CANDIDATE DAG with parallel BAITS fan-out
    req = InboundWorkRequest(
        request_id="req-001",
        channel="portal",
        text="Evaluate the Acme Cloud SOW across business value and zero-trust security architecture.",
        engagement_slug="acme-cloud",
        origin="USER",
    )
    run = await orchestrator.handle_turn(req)
    assert run.plan.status == WorkflowStatus.CANDIDATE
    node_agents = [n.agent_uri for n in run.plan.nodes]
    assert "chotu://agent/architects/business-architect@1.0.0" in node_agents
    assert "chotu://agent/architects/security-architect@1.0.0" in node_agents
    assert "chotu://agent/architects/csa-reviewer@1.0.0" in node_agents


@pytest.mark.asyncio
async def test_lead_chotu_blocks_direct_raw_tool_invocation(
    in_memory_discovery,
    in_memory_workflow_registry,
    fake_dag_runner,
    governed_tool_registry,
    inventory_root: Path,
) -> None:
    loader = InventoryManifestLoader()
    await loader.scan_and_register(inventory_root, in_memory_discovery)
    orchestrator = LeadOrchestrator(
        classifier=RuleAndCheckpointWorkClassifier(),
        discovery=in_memory_discovery,
        workflow_registry=in_memory_workflow_registry,
        dag_runner=fake_dag_runner,
        tool_registry=governed_tool_registry,
    )

    with pytest.raises(SubagentOnlyViolationError, match="lead/chotu is prohibited from invoking raw tools"):
        await orchestrator.invoke_raw_tool(
            tool_uri="chotu://tool/builtin/bash@1.0.0",
            payload={"command": "ls -la"},
        )
```

- [ ] **Step 2: Run test to verify it fails**

Run: `uv run pytest backend/tests/inventory/test_lead_chotu_orchestrator.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'backend.inventory.agents.lead.chotu.orchestrator'`

- [ ] **Step 3: Implement `InventoryManifestLoader`, `RuleAndCheckpointWorkClassifier`, and `LeadOrchestrator`**

1. In `backend/inventory/agents/lead/chotu/manifest.yaml`, declare `uri: "chotu://agent/lead/chotu@1.0.0"`, `kind: "agent"`, `department: "executive"`, `allowed_tools: []`, `lifecycle_policy: "always_on"`, and typed JSON schemas for `InboundWorkRequest` and `WorkflowRun`.
2. In `backend/platform/multiagent/adapters/inventory_loader.py`, implement `InventoryManifestLoader.scan_and_register(inventory_root: Path, discovery: ServiceDiscoveryPort) -> list[CapabilityDescriptor]` to recursively discover `manifest.yaml`, `SKILL.md` frontmatter, and `pack.yaml` files, validate their URI and I/O schemas, and register them with `discovery.register(...)`.
3. In `backend/inventory/classifiers/work_classifier.py`, implement `RuleAndCheckpointWorkClassifier.classify(request: InboundWorkRequest) -> WorkClassification` to extract `intent`, `mode` (`work_engagement` when `engagement_slug` is present, `personal_rd` for personal triggers, `universal_research` for cross-engagement queries), `departments`, `baits_pillars` (`business`, `application`, `information`, `technology`, `security`), 6-value `scope`, and 5-value `perspective`.
4. In `backend/inventory/agents/lead/chotu/orchestrator.py`, implement `LeadOrchestrator`:

```python
class SubagentOnlyViolationError(PermissionError):
    """Raised if lead/chotu or any orchestrator attempts to call raw tools directly."""


class LeadOrchestrator:
    AGENT_URI = "chotu://agent/lead/chotu@1.0.0"

    def __init__(
        self,
        classifier: WorkClassifierPort,
        discovery: ServiceDiscoveryPort,
        workflow_registry: WorkflowRegistryPort,
        dag_runner: WorkflowDAGRunnerPort,
        tool_registry: ToolRegistryPort | None = None,
    ) -> None:
        self._classifier = classifier
        self._discovery = discovery
        self._workflow_registry = workflow_registry
        self._dag_runner = dag_runner
        self._tool_registry = tool_registry

    async def invoke_raw_tool(self, tool_uri: str, payload: dict[str, Any]) -> dict[str, Any]:
        raise SubagentOnlyViolationError(
            f"{self.AGENT_URI} is prohibited from invoking raw tools ({tool_uri}); delegate to a scoped subagent."
        )

    async def handle_turn(self, request: InboundWorkRequest) -> WorkflowRun:
        classification = await self._classifier.classify(request)
        established_plan = await self._workflow_registry.find_established(classification)
        if established_plan is not None and await self._all_nodes_healthy(established_plan):
            return await self._dag_runner.submit_plan(
                established_plan, origin=request.origin, hop_depth=request.hop_depth
            )
        candidate_plan = await self._compose_dynamic_dag(request, classification)
        await self._workflow_registry.save_candidate(candidate_plan)
        return await self._dag_runner.submit_plan(
            candidate_plan, origin=request.origin, hop_depth=request.hop_depth
        )
```

- [ ] **Step 4: Run test to verify it passes**

Run: `uv run pytest backend/tests/inventory/test_lead_chotu_orchestrator.py -v`
Expected: PASS (3 passed)

- [ ] **Step 5: Commit**

```bash
git add backend/platform/multiagent/adapters/inventory_loader.py \
        backend/inventory/agents/lead/chotu/ \
        backend/inventory/classifiers/work_classifier.py \
        backend/tests/inventory/test_lead_chotu_orchestrator.py
git commit -m "feat(inventory): add InventoryManifestLoader, WorkClassifier, and Subagents-Only LeadOrchestrator"
```

---

### Task 2: BAITS Consulting & Architecture Department (`inventory/agents/domain/architects/`)

In a high-velocity consulting practice, a single architectural question—such as migrating a client's core payment system to a cloud-native event bus—touches business KPIs, application API seams, canonical data lineage, infrastructure latency SLAs, and zero-trust IAM controls simultaneously. Rather than collapsing all five lenses into one generic prompt, Task 2 harvests the 8 CSA specialist agents from [`dsa-companion-config`](file:///home/nox/Developer/ds2/dsa-companion-config/README.md) plus our conversational `narrative-writer` agent into `backend/inventory/agents/domain/architects/`.

Each specialist has its own declarative `manifest.yaml` (specifying its BAITS pillar tag, allowed scoped tools, and JSON I/O contract) and is governed by `csa-reviewer`'s **Citeability Firewall**. A subtle failure mode in consulting automation is accidentally citing an internal team hypothesis (`scope="internal"`) or an unverified personal voice memo (`perspective="personal_take"`) inside a client-facing deliverable (`target_scope="deliverable"`). Our test suite in this task explicitly pins `csa-reviewer` so any deliverable containing non-citeable scopes or missing `[[concept#^f-id]]` anchors is rejected with a structured `CiteabilityFirewallError`.

**Files:**
- Create: `backend/inventory/agents/domain/architects/business_architect/manifest.yaml`
- Create: `backend/inventory/agents/domain/architects/application_architect/manifest.yaml`
- Create: `backend/inventory/agents/domain/architects/information_architect/manifest.yaml`
- Create: `backend/inventory/agents/domain/architects/technology_architect/manifest.yaml`
- Create: `backend/inventory/agents/domain/architects/security_architect/manifest.yaml`
- Create: `backend/inventory/agents/domain/architects/csa_researcher/manifest.yaml`
- Create: `backend/inventory/agents/domain/architects/csa_reviewer/manifest.yaml`
- Create: `backend/inventory/agents/domain/architects/csa_scribe/manifest.yaml`
- Create: `backend/inventory/agents/domain/architects/narrative_writer/manifest.yaml`
- Create: `backend/inventory/agents/domain/architects/handlers.py`
- Test: `backend/tests/inventory/test_baits_architects_and_reviewer_firewall.py`

**Interfaces:**
- Consumes:
  - `ContextPacket(engagement_slug: str, facts: list[HydratedFact], active_packs: list[str], unresolved_conflicts: list[ConflictSummary])`
- Produces:
  - `PillarAssessmentOutput(pillar: Literal["business", "application", "information", "technology", "security"], markdown_body: str, cited_anchors: list[str], raid_items: list[dict[str, str]])`
  - `CSAReviewerAuditResult(passed: bool, target_scope: str, invalid_anchors: list[str], firewall_violations: list[str])`
  - `CiteabilityFirewallError(ValueError)`
  - `CSAReviewerAgent.audit_output(markdown_body: str, target_scope: str, fact_index: dict[str, FactMetadata]) -> CSAReviewerAuditResult`
  - `CSAResearcherAgent.resolve_knowledge_gap(query: str, engagement_slug: str, local_librarian: LibrarianPort, universal_service: UniversalKnowledgePort) -> list[HydratedFact]`

- [ ] **Step 1: Write the failing test**

Create `backend/tests/inventory/test_baits_architects_and_reviewer_firewall.py` testing: (a) discovery of all 9 architect department manifests with their exact `chotu://agent/architects/...` URIs, (b) `csa-researcher` querying local engagement scope first and only invoking Universal Knowledge Discover $\rightarrow$ Select $\rightarrow$ Ingest on a local miss, and (c) Review Focus #2 (`CiteabilityFirewallError` when `scope="internal"` or `perspective="personal_take"` is cited in a `deliverable` artifact):

```python
from __future__ import annotations

from pathlib import Path
import pytest

from backend.inventory.agents.domain.architects.handlers import (
    CSAResearcherAgent,
    CSAReviewerAgent,
    CiteabilityFirewallError,
    FactMetadata,
)
from backend.platform.multiagent.adapters.inventory_loader import InventoryManifestLoader


@pytest.mark.asyncio
async def test_all_nine_architect_department_agents_register(
    in_memory_discovery,
    inventory_root: Path,
) -> None:
    loader = InventoryManifestLoader()
    await loader.scan_and_register(inventory_root, in_memory_discovery)

    expected_uris = {
        "chotu://agent/architects/business-architect@1.0.0",
        "chotu://agent/architects/application-architect@1.0.0",
        "chotu://agent/architects/information-architect@1.0.0",
        "chotu://agent/architects/technology-architect@1.0.0",
        "chotu://agent/architects/security-architect@1.0.0",
        "chotu://agent/architects/csa-researcher@1.0.0",
        "chotu://agent/architects/csa-reviewer@1.0.0",
        "chotu://agent/architects/csa-scribe@1.0.0",
        "chotu://agent/architects/narrative-writer@1.0.0",
    }
    registered = {d.uri for d in await in_memory_discovery.list_by_department("delivery")}
    assert expected_uris.issubset(registered)


def test_csa_reviewer_rejects_internal_or_personal_take_fact_in_deliverable() -> None:
    reviewer = CSAReviewerAgent()
    fact_index = {
        "payment-gateway#^f-101": FactMetadata(
            anchor="payment-gateway#^f-101",
            scope="client",
            perspective="client_stated",
        ),
        "payment-gateway#^f-102": FactMetadata(
            anchor="payment-gateway#^f-102",
            scope="internal",
            perspective="personal_take",
        ),
    }
    draft_deliverable = (
        "The payment gateway processes 4,000 TPS [[payment-gateway#^f-101]]. "
        "However, the client team lacks Kafka experience [[payment-gateway#^f-102]]."
    )

    with pytest.raises(CiteabilityFirewallError) as exc_info:
        reviewer.audit_output(
            markdown_body=draft_deliverable,
            target_scope="deliverable",
            fact_index=fact_index,
            raise_on_violation=True,
        )
    assert "payment-gateway#^f-102" in str(exc_info.value)


@pytest.mark.asyncio
async def test_csa_researcher_queries_local_engagement_before_universal_discover_ingest(
    fake_local_librarian,
    fake_universal_knowledge_service,
) -> None:
    researcher = CSAResearcherAgent()
    # Case 1: Local engagement has facts -> Universal Knowledge is NEVER queried
    fake_local_librarian.seed_fact("acme-cloud", "zero-trust-iam#^f-01", "Client uses Okta Workforce Identity.")
    local_hits = await researcher.resolve_knowledge_gap(
        query="What identity provider does the client use?",
        engagement_slug="acme-cloud",
        local_librarian=fake_local_librarian,
        universal_service=fake_universal_knowledge_service,
    )
    assert len(local_hits) == 1
    assert fake_universal_knowledge_service.query_count == 0

    # Case 2: Local miss -> Discover -> Select -> Ingest into engagement with scope='reference'
    fake_universal_knowledge_service.seed_universal_fact(
        "nist-zero-trust#^f-99", "NIST SP 800-207 mandates per-session policy evaluation."
    )
    imported_hits = await researcher.resolve_knowledge_gap(
        query="NIST SP 800-207 policy evaluation standard",
        engagement_slug="acme-cloud",
        local_librarian=fake_local_librarian,
        universal_service=fake_universal_knowledge_service,
    )
    assert fake_universal_knowledge_service.query_count == 1
    assert imported_hits[0].scope == "reference"
    assert imported_hits[0].engagement_slug == "acme-cloud"
```

- [ ] **Step 2: Run test to verify it fails**

Run: `uv run pytest backend/tests/inventory/test_baits_architects_and_reviewer_firewall.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'backend.inventory.agents.domain.architects.handlers'`

- [ ] **Step 3: Implement the 9 Architect manifests and `handlers.py` (`CSAReviewerAgent`, `CSAResearcherAgent`, `BAITSArchitectAgent`, `NarrativeWriterAgent`)**

1. Create the 9 `manifest.yaml` files under `backend/inventory/agents/domain/architects/<agent_name>/manifest.yaml`, setting `department: "delivery"`, appropriate `baits_pillars` (`["business"]`, `["application"]`, `["information"]`, `["technology"]`, `["security"]`, or all five for `csa-researcher`, `csa-reviewer`, `csa-scribe`, and `narrative-writer`), and scoped `allowed_tools` (`chotu://tool/kb/librarian-query@1.0.0`, `chotu://tool/kb/okf-read@1.0.0`).
2. In `backend/inventory/agents/domain/architects/handlers.py`, implement regex extraction for `\[\[([a-z0-9\-_]+#\^f-[a-z0-9\-_]+)\]\]` inside `CSAReviewerAgent.audit_output(...)` and enforce the 6-value scope citeability matrix: when `target_scope == "deliverable"`, only facts with `scope in {"engagement", "client", "reference", "deliverable"}` and `perspective != "personal_take"` are permitted; any violation appends to `firewall_violations` and raises `CiteabilityFirewallError` when `raise_on_violation=True`.
3. Implement `CSAResearcherAgent.resolve_knowledge_gap(...)` so it queries `local_librarian.retrieve(engagement_slug, query)` first and only calls `universal_service.discover_and_ingest(query, target_engagement=engagement_slug, target_scope="reference")` when local retrieval returns zero hits above threshold.

- [ ] **Step 4: Run test to verify it passes**

Run: `uv run pytest backend/tests/inventory/test_baits_architects_and_reviewer_firewall.py -v`
Expected: PASS (3 passed)

- [ ] **Step 5: Commit**

```bash
git add backend/inventory/agents/domain/architects/ \
        backend/tests/inventory/test_baits_architects_and_reviewer_firewall.py
git commit -m "feat(inventory): add BAITS specialist architects, csa-researcher, and csa-reviewer citeability firewall"
```

---

### Task 3: Multi-Format Document & Transcript Extractors (`inventory/agents/internal/extractors/`)

With ~10 multi-speaker meetings and ~20 complex documents (PDFs, Word SOWs, PowerPoint decks, Excel sheets, HTML specs, whiteboard photos) landing every day, Chotu's ingestion pipeline depends on rock-solid extractors in `backend/inventory/agents/internal/extractors/` harvested from [`dsappz-dsrag-prj`](file:///home/nox/Developer/ds2/dsappz-dsrag-prj/CLAUDE.md) and [`dsa-companion-config`](file:///home/nox/Developer/ds2/dsa-companion-config/README.md).

Two real-world failure modes matter most here. First, raw meeting transcripts never arrive neatly organized by topic, and a single 45-minute call routinely mixes statements from client stakeholders (`client_stated`), internal delivery teammates (`internal_team`), and your own post-meeting voice notes (`personal_take`). If `parse-transcript` flattens the transcript into one generic summary, Chotu loses both topic boundaries and perspective attribution. Second, while the Docling sidecar (`parsing-service/`) gives superior table and bounding-box layout extraction for complex PDFs, sidecar containers can be offline or time out under heavy load. Therefore, `parse-pdf` is configured with `lifecycle_policy: "lazy_start"` and `fallback_uri: "chotu://agent/extractors/parse-pdf-local@1.0.0"`, automatically falling back to local PyMuPDF extraction when the Docling HTTP call fails.

**Files:**
- Create: `backend/inventory/agents/internal/extractors/parse_transcript/manifest.yaml`
- Create: `backend/inventory/agents/internal/extractors/parse_pdf/manifest.yaml`
- Create: `backend/inventory/agents/internal/extractors/parse_office/manifest.yaml`
- Create: `backend/inventory/agents/internal/extractors/parse_html/manifest.yaml`
- Create: `backend/inventory/agents/internal/extractors/parse_image/manifest.yaml`
- Create: `backend/inventory/agents/internal/extractors/parse_local/manifest.yaml`
- Create: `backend/inventory/agents/internal/extractors/extractors.py`
- Test: `backend/tests/inventory/test_extractors_transcript_and_docling_fallback.py`

**Interfaces:**
- Consumes:
  - `RawSourceEnvelope(source_path: Path, mime_type: str, engagement_slug: str, speaker_roster: dict[str, Literal["client_stated", "internal_team", "personal_take"]])`
- Produces:
  - `ExtractedTurnSegment(speaker: str, timestamp: str, text: str, perspective: Literal["client_stated", "internal_team", "personal_take", "work_research", "personal_rd"], scope: Literal["engagement", "client", "internal", "reference", "prior-project", "deliverable"])`
  - `TopicSegment(topic_slug: str, topic_title: str, baits_pillars: list[str], turns: list[ExtractedTurnSegment], narrative_summary: str)`
  - `ExtractedDocumentArtifact(source_sha256: str, extractor_uri: str, used_fallback: bool, markdown_content: str, topics: list[TopicSegment], tables: list[dict[str, Any]])`
  - `TranscriptExtractor.extract(envelope: RawSourceEnvelope) -> ExtractedDocumentArtifact`
  - `PDFExtractor.extract(envelope: RawSourceEnvelope) -> ExtractedDocumentArtifact`

- [ ] **Step 1: Write the failing test**

Create `backend/tests/inventory/test_extractors_transcript_and_docling_fallback.py` testing Review Focus #3: (a) `TranscriptExtractor` teasing apart 3 interleaved topics while preserving `client_stated`, `internal_team`, and `personal_take` perspectives per turn, and (b) `PDFExtractor` falling back to local extraction when the Docling sidecar raises `httpx.ConnectError` or `httpx.TimeoutException`:

```python
from __future__ import annotations

from pathlib import Path
import httpx
import pytest

from backend.inventory.agents.internal.extractors.extractors import (
    PDFExtractor,
    RawSourceEnvelope,
    TranscriptExtractor,
)


def test_parse_transcript_preserves_three_topics_and_three_perspectives(tmp_path: Path) -> None:
    transcript_file = tmp_path / "2026-10-28-architecture-sync.vtt"
    transcript_file.write_text(
        "[00:01:10] Dana (Client VP): [Topic: Legacy Billing Decommission] We must retire Legacy Billing by Q2 2027 to cut mainframe licensing costs.\n"
        "[00:02:45] Raj (Internal Lead): [Topic: Legacy Billing Decommission] The SOW milestone schedule assumes Q4 2027 because of ledger migration dependencies.\n"
        "[00:15:00] Dana (Client VP): [Topic: Zero Trust SSO] All customer-facing APIs must enforce OIDC token exchange through Okta.\n"
        "[00:28:30] Operator (Personal Memo): [Topic: Kafka Event Bus Sizing] My personal hypothesis is that 3 partitions per tenant will bottleneck during holiday peak.\n",
        encoding="utf-8",
    )

    envelope = RawSourceEnvelope(
        source_path=transcript_file,
        mime_type="text/vtt",
        engagement_slug="acme-cloud",
        speaker_roster={
            "Dana (Client VP)": "client_stated",
            "Raj (Internal Lead)": "internal_team",
            "Operator (Personal Memo)": "personal_take",
        },
    )

    artifact = TranscriptExtractor().extract(envelope)
    assert len(artifact.topics) == 3
    topic_slugs = [t.topic_slug for t in artifact.topics]
    assert topic_slugs == [
        "legacy-billing-decommission",
        "zero-trust-sso",
        "kafka-event-bus-sizing",
    ]

    decomm_turns = artifact.topics[0].turns
    assert len(decomm_turns) == 2
    assert decomm_turns[0].perspective == "client_stated"
    assert decomm_turns[0].scope == "client"
    assert decomm_turns[1].perspective == "internal_team"
    assert decomm_turns[1].scope == "internal"

    memo_turn = artifact.topics[2].turns[0]
    assert memo_turn.perspective == "personal_take"
    assert memo_turn.scope == "internal"


@pytest.mark.asyncio
async def test_parse_pdf_falls_back_to_local_when_docling_sidecar_times_out(tmp_path: Path) -> None:
    pdf_file = tmp_path / "sow.pdf"
    pdf_file.write_bytes(b"%PDF-1.4\n% Fake SOW Content: Milestone 1 Deliverable due 2027-03-01\n")

    async def failing_docling_client(path: Path) -> dict[str, object]:
        raise httpx.ConnectError("Docling sidecar container offline at http://localhost:8090")

    def working_local_fallback(path: Path) -> str:
        return "# Statement of Work\n\nMilestone 1 Deliverable due 2027-03-01."

    extractor = PDFExtractor(
        docling_client=failing_docling_client,
        local_fallback_fn=working_local_fallback,
    )
    envelope = RawSourceEnvelope(
        source_path=pdf_file,
        mime_type="application/pdf",
        engagement_slug="acme-cloud",
        speaker_roster={},
    )

    artifact = await extractor.extract(envelope)
    assert artifact.used_fallback is True
    assert artifact.extractor_uri == "chotu://agent/extractors/parse-pdf-local@1.0.0"
    assert "Milestone 1 Deliverable due 2027-03-01" in artifact.markdown_content
```

- [ ] **Step 2: Run test to verify it fails**

Run: `uv run pytest backend/tests/inventory/test_extractors_transcript_and_docling_fallback.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'backend.inventory.agents.internal.extractors.extractors'`

- [ ] **Step 3: Implement the 6 Extractor manifests and `extractors.py` (`TranscriptExtractor`, `PDFExtractor`, `OfficeExtractor`, `HTMLExtractor`, `ImageExtractor`, `LocalFileExtractor`)**

1. Create `manifest.yaml` for `parse-transcript`, `parse-pdf` (with `lifecycle_policy: "lazy_start"` and `fallback_uri: "chotu://agent/extractors/parse-pdf-local@1.0.0"`), `parse-office`, `parse-html`, `parse-image`, and `parse-local` under `backend/inventory/agents/internal/extractors/`.
2. In `backend/inventory/agents/internal/extractors/extractors.py`, implement `TranscriptExtractor.extract(envelope: RawSourceEnvelope) -> ExtractedDocumentArtifact` to parse `[timestamp] Speaker: ...` lines, resolve speaker perspective via `envelope.speaker_roster` (mapping `client_stated` $\rightarrow$ `scope="client"`, `internal_team` $\rightarrow$ `scope="internal"`, `personal_take` $\rightarrow$ `scope="internal"`), group turns by explicit `[Topic: ...]` markers or heading shifts into `TopicSegment` objects while preserving chronological order within each topic, and compute SHA-256 content hashes.
3. Implement `PDFExtractor.extract(envelope: RawSourceEnvelope) -> ExtractedDocumentArtifact` to attempt the Docling sidecar call first and catch `(httpx.HTTPError, TimeoutError, OSError)` to invoke `local_fallback_fn`, setting `used_fallback=True` and `extractor_uri="chotu://agent/extractors/parse-pdf-local@1.0.0"`.

- [ ] **Step 4: Run test to verify it passes**

Run: `uv run pytest backend/tests/inventory/test_extractors_transcript_and_docling_fallback.py -v`
Expected: PASS (2 passed)

- [ ] **Step 5: Commit**

```bash
git add backend/inventory/agents/internal/extractors/ \
        backend/tests/inventory/test_extractors_transcript_and_docling_fallback.py
git commit -m "feat(inventory): add multi-topic/multi-perspective transcript extractor and PDF Docling fallback"
```

---

### Task 4: Custodial Knowledge Warehouse Department (`domain/ingestion/` & `domain/kb/`) & Self-Extension Builders (`domain/builders/`)

Once an extractor turns a raw PDF or transcript into structured topic segments, the Custodial Knowledge Warehouse department takes over. This department is split into the **Ingestion Pipeline** (`csa-ingest` $\rightarrow$ `csa-extract` $\rightarrow$ `csa-concept-writer` $\rightarrow$ `okf-indexer`) and the **Knowledge Custodians** (`context_retrieval_agent`, `csa-synthesize`, `truth_finder_agent`, `retro_elevator_agent`, `vault_hygiene`), alongside the **Self-Extension Builders** (`agent_builder`, `tool_builder`, `skill_builder`, `app_onboarding`) in `domain/builders/`.

The most important epistemic rule in Chotu Corp lives inside `csa-concept-writer` and `truth_finder_agent`: **competing claims are never auto-overwritten**. When a newly extracted `client_stated` claim contradicts an existing `internal_team` claim on the same concept and BAITS pillar, `csa-concept-writer` writes the new claim as a versioned `^f-id` line in `okf-wiki/<concept>.md` alongside the original claim and emits a `ConflictDetectedEvent`. `truth_finder_agent` then runs the Slice 2 Truth Finder + HITS engine to compute Step-1 structural Make/Break impact (how many SOW milestones and ontology dependencies each claim makes vs. breaks) and Step-2 plausibility scores. Likewise, for self-extension, `tool_builder` must run tests inside a sandbox worktree and block activation until an `ApprovalBus` gate is approved.

**Files:**
- Create: `backend/inventory/agents/domain/ingestion/manifests.yaml`
- Create: `backend/inventory/agents/domain/ingestion/pipeline.py`
- Create: `backend/inventory/agents/domain/kb/manifests.yaml`
- Create: `backend/inventory/agents/domain/kb/custodians.py`
- Create: `backend/inventory/agents/domain/builders/manifests.yaml`
- Create: `backend/inventory/agents/domain/builders/sandbox_builders.py`
- Test: `backend/tests/inventory/test_kb_custodians_and_sandbox_builders.py`

**Interfaces:**
- Consumes:
  - `ExtractedDocumentArtifact` (from Task 3)
  - `TruthFinderEnginePort.weigh_competing_claims(engagement_slug: str, concept_slug: str, claims: list[ExtractedFactRecord]) -> TruthFinderWeighingReport` (from Slice 2)
  - `ApprovalBusPort.request_approval(item_uri: str, summary: str, sandbox_test_passed: bool) -> ApprovalTicket` (from Slice 1)
- Produces:
  - `ExtractedFactRecord(fact_id: str, concept_slug: str, pillar: str, statement: str, scope: str, perspective: str, observed_at: str, ingested_at: str, version: int, subject_entity: str, predicate: str, object_value: str)`
  - `ConceptWriteResult(concept_path: Path, written_fact_ids: list[str], conflicts_detected: list[ConflictDetectedEvent])`
  - `CSAConceptWriter.write_facts(wiki_root: Path, facts: list[ExtractedFactRecord]) -> list[ConceptWriteResult]`
  - `TruthFinderAgent.evaluate_conflicts(engagement_slug: str, conflicts: list[ConflictDetectedEvent]) -> list[TruthFinderWeighingReport]`
  - `SandboxToolBuilder.build_and_stage_tool(spec: ToolBuildSpec) -> StagedCapabilityResult`

- [ ] **Step 1: Write the failing test**

Create `backend/tests/inventory/test_kb_custodians_and_sandbox_builders.py` testing Review Focus #4: (a) `CSAConceptWriter` preserving both `client_stated` ("Q2 2027") and `internal_team` ("Q4 2027") claims in `okf-wiki/legacy-billing.md` without overwriting and triggering `TruthFinderAgent` Make/Break weighing, and (b) `SandboxToolBuilder` blocking production registration until sandbox tests pass and `ApprovalBus` approves:

```python
from __future__ import annotations

from pathlib import Path
import pytest

from backend.inventory.agents.domain.builders.sandbox_builders import (
    SandboxToolBuilder,
    ToolBuildSpec,
    UnapprovedCapabilityError,
)
from backend.inventory.agents.domain.ingestion.pipeline import (
    CSAConceptWriter,
    ExtractedFactRecord,
)
from backend.inventory.agents.domain.kb.custodians import TruthFinderAgent


def test_concept_writer_preserves_competing_perspectives_and_emits_conflict(
    tmp_path: Path,
    fake_truth_finder_engine,
) -> None:
    wiki_root = tmp_path / "okf-wiki"
    writer = CSAConceptWriter()

    fact_internal = ExtractedFactRecord(
        fact_id="f-201",
        concept_slug="legacy-billing",
        pillar="application",
        statement="Legacy Billing decommissions in Q4 2027 after ledger migration.",
        scope="internal",
        perspective="internal_team",
        observed_at="2026-10-27T10:00:00Z",
        ingested_at="2026-10-27T10:05:00Z",
        version=1,
        subject_entity="legacy-billing",
        predicate="decommission_target_quarter",
        object_value="Q4 2027",
    )
    fact_client = ExtractedFactRecord(
        fact_id="f-202",
        concept_slug="legacy-billing",
        pillar="application",
        statement="Legacy Billing decommissions in Q2 2027 to eliminate mainframe fees.",
        scope="client",
        perspective="client_stated",
        observed_at="2026-10-28T14:00:00Z",
        ingested_at="2026-10-28T14:02:00Z",
        version=2,
        subject_entity="legacy-billing",
        predicate="decommission_target_quarter",
        object_value="Q2 2027",
    )

    writer.write_facts(wiki_root, [fact_internal])
    results = writer.write_facts(wiki_root, [fact_client])

    concept_md = (wiki_root / "legacy-billing.md").read_text(encoding="utf-8")
    assert "^f-201" in concept_md
    assert "Q4 2027" in concept_md
    assert "^f-202" in concept_md
    assert "Q2 2027" in concept_md

    assert len(results[0].conflicts_detected) == 1
    conflict = results[0].conflicts_detected[0]
    assert conflict.subject_entity == "legacy-billing"
    assert conflict.predicate == "decommission_target_quarter"

    tf_agent = TruthFinderAgent(engine=fake_truth_finder_engine)
    reports = tf_agent.evaluate_conflicts("acme-cloud", results[0].conflicts_detected)
    assert len(reports) == 1
    assert reports[0].make_break_diff["f-202"]["breaks_relationships"] > 0


@pytest.mark.asyncio
async def test_sandbox_tool_builder_enforces_hitl_approval_before_activation(
    tmp_path: Path,
    in_memory_discovery,
    fake_approval_bus,
) -> None:
    builder = SandboxToolBuilder(
        sandbox_root=tmp_path / "sandbox",
        discovery=in_memory_discovery,
        approval_bus=fake_approval_bus,
    )
    spec = ToolBuildSpec(
        uri="chotu://tool/custom/csv-column-profiler@1.0.0",
        description="Profiles null ratios in CSV files.",
        source_code="def run(payload: dict) -> dict:\n    return {'rows': len(payload.get('rows', []))}\n",
        test_code="def test_run(fn):\n    assert fn({'rows': [1, 2, 3]}) == {'rows': 3}\n",
    )

    staged = await builder.build_and_stage_tool(spec)
    assert staged.sandbox_tests_passed is True
    assert staged.status == "AWAITING_APPROVAL"

    with pytest.raises(UnapprovedCapabilityError):
        await builder.activate_staged_tool(staged.ticket_id)

    fake_approval_bus.approve(staged.ticket_id, approver="operator")
    descriptor = await builder.activate_staged_tool(staged.ticket_id)
    assert descriptor.uri == "chotu://tool/custom/csv-column-profiler@1.0.0"
```

- [ ] **Step 2: Run test to verify it fails**

Run: `uv run pytest backend/tests/inventory/test_kb_custodians_and_sandbox_builders.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'backend.inventory.agents.domain.ingestion.pipeline'`

- [ ] **Step 3: Implement `CSAConceptWriter`, `TruthFinderAgent`, `RetroElevatorAgent`, `VaultHygieneAgent`, and `SandboxToolBuilder`**

1. Create declarative manifests in `backend/inventory/agents/domain/ingestion/manifests.yaml` (`csa-ingest`, `csa-extract`, `csa-concept-writer`, `okf-indexer`), `backend/inventory/agents/domain/kb/manifests.yaml` (`context-retrieval-agent`, `csa-synthesize`, `truth-finder-agent`, `retro-elevator-agent`, `vault-hygiene`), and `backend/inventory/agents/domain/builders/manifests.yaml` (`agent-builder`, `tool-builder`, `skill-builder`, `app-onboarding`).
2. In `backend/inventory/agents/domain/ingestion/pipeline.py`, implement `CSAConceptWriter.write_facts(wiki_root: Path, facts: list[ExtractedFactRecord]) -> list[ConceptWriteResult]` so that each concept file (`<concept_slug>.md`) maintains `## <Pillar>` sections with structured fact lines `- [scope:<scope>|perspective:<perspective>|subject:<subject>|pred:<pred>|val:<val>] <statement> ^<fact_id>`. When an incoming fact has the same `(subject_entity, predicate)` as an existing fact in the file but a different `object_value`, append the new fact line without deleting the existing one and emit a `ConflictDetectedEvent`.
3. In `backend/inventory/agents/domain/kb/custodians.py`, implement `TruthFinderAgent.evaluate_conflicts(...)`, `RetroElevatorAgent.run_five_lens_elevation(...)`, and `VaultHygieneAgent.audit_vault(...)`.
4. In `backend/inventory/agents/domain/builders/sandbox_builders.py`, implement `SandboxToolBuilder` to execute `spec.test_code` against `spec.source_code` in an isolated temporary directory, create an `ApprovalTicket` via `ApprovalBusPort`, and raise `UnapprovedCapabilityError` if `activate_staged_tool` is called before the ticket is marked approved.

- [ ] **Step 4: Run test to verify it passes**

Run: `uv run pytest backend/tests/inventory/test_kb_custodians_and_sandbox_builders.py -v`
Expected: PASS (2 passed)

- [ ] **Step 5: Commit**

```bash
git add backend/inventory/agents/domain/ingestion/ \
        backend/inventory/agents/domain/kb/ \
        backend/inventory/agents/domain/builders/ \
        backend/tests/inventory/test_kb_custodians_and_sandbox_builders.py
git commit -m "feat(inventory): add custodial ingestion/KB agents, multi-perspective concept writer, and sandbox tool builder"
```

---

### Task 5: Operationalized `csa-*` Skills (`inventory/skills/`) & Drop-In Capability-Bundle Packs (`inventory/packs/bvf/`, `raid/`, `baits/`)

To make Chotu Corp extensible without touching Python code whenever you adopt a new consulting methodology, Task 5 harvests the operationalized `csa-*` skills (`csa-init`, `csa-ask`, `csa-context`, `csa-catalog`, `csa-deliverables`, `csa-engagement-scoping`, `csa-export`, `csa-import-bundle`, `make-diagram`, `slalom-html-slide-decks`, `skill-adapter`) into `backend/inventory/skills/` and the capability-bundle packs (`bvf/`, `raid/`, `baits/`) into `backend/inventory/packs/`.

We wire these into runtime execution through a blocking `pre_subagent_dispatch` lifecycle hook (`SkillAndPackInjectorHook` in `backend/inventory/hooks/skill_pack_injector.py`). Whenever `WorkflowDAGRunner` is about to dispatch a subagent (such as `business-architect` or `csa-synthesize`) inside an engagement whose `.csa/config.yaml` enables `packs: ["bvf", "raid"]`, the hook automatically reads the matching `pack.yaml` rubrics and `SKILL.md` instructions and attaches them to the subagent's `SubagentDispatchEnvelope`. We also test Review Focus #5 here: if someone drops a malformed `pack.yaml` with an unknown BAITS pillar or a missing template file into `inventory/packs/`, `PackLoader` catches it immediately with `InvalidPackManifestError`.

**Files:**
- Create: `backend/inventory/skills/csa_skills_manifest.yaml`
- Create: `backend/inventory/packs/bvf/pack.yaml`
- Create: `backend/inventory/packs/bvf/rubric.md`
- Create: `backend/inventory/packs/raid/pack.yaml`
- Create: `backend/inventory/packs/raid/rubric.md`
- Create: `backend/inventory/packs/baits/pack.yaml`
- Create: `backend/inventory/packs/baits/rubric.md`
- Create: `backend/inventory/packs/loader.py`
- Create: `backend/inventory/hooks/hooks.yaml`
- Create: `backend/inventory/hooks/skill_pack_injector.py`
- Test: `backend/tests/inventory/test_skills_packs_and_injector_hook.py`

**Interfaces:**
- Consumes:
  - `SubagentDispatchEnvelope(agent_uri: str, engagement_slug: str,baits_pillars: list[str], active_packs: list[str], injected_skills: list[str], injected_rubrics: dict[str, str])`
- Produces:
  - `CapabilityPackManifest(uri: str, name: str, version: str, target_pillars: list[Literal["business", "application", "information", "technology", "security"]], target_agents: list[str], rubric_file: str, ontology_types: list[str])`
  - `InvalidPackManifestError(ValueError)`
  - `PackLoader.load_pack(pack_dir: Path) -> CapabilityPackManifest`
  - `SkillAndPackInjectorHook.pre_subagent_dispatch(envelope: SubagentDispatchEnvelope) -> SubagentDispatchEnvelope`

- [ ] **Step 1: Write the failing test**

Create `backend/tests/inventory/test_skills_packs_and_injector_hook.py` testing: (a) registration of all 11 operationalized `csa-*` skills and the `bvf`, `raid`, and `baits` packs, (b) `SkillAndPackInjectorHook` auto-attaching `bvf` and `raid` rubrics to `business-architect` on `pre_subagent_dispatch`, and (c) Review Focus #5 (`InvalidPackManifestError` when `pack.yaml` specifies an invalid pillar or missing rubric file):

```python
from __future__ import annotations

from pathlib import Path
import pytest

from backend.inventory.hooks.skill_pack_injector import (
    SkillAndPackInjectorHook,
    SubagentDispatchEnvelope,
)
from backend.inventory.packs.loader import InvalidPackManifestError, PackLoader


def test_pack_loader_loads_bvf_raid_baits_and_injects_on_pre_subagent_dispatch(
    inventory_root: Path,
) -> None:
    loader = PackLoader(packs_root=inventory_root / "packs", skills_root=inventory_root / "skills")
    packs = loader.load_all_packs()
    assert {"bvf", "raid", "baits"}.issubset(packs.keys())

    skills = loader.load_all_skills()
    expected_skills = {
        "csa-init",
        "csa-engagement-scoping",
        "csa-context",
        "csa-ask",
        "csa-catalog",
        "csa-deliverables",
        "csa-export",
        "csa-import-bundle",
        "make-diagram",
        "slalom-html-slide-decks",
        "skill-adapter",
    }
    assert expected_skills.issubset(skills.keys())

    hook = SkillAndPackInjectorHook(pack_loader=loader)
    envelope = SubagentDispatchEnvelope(
        agent_uri="chotu://agent/architects/business-architect@1.0.0",
        engagement_slug="acme-cloud",
        baits_pillars=["business"],
        active_packs=["bvf", "raid"],
    )
    enriched = hook.pre_subagent_dispatch(envelope)
    assert "bvf" in enriched.injected_rubrics
    assert "Business Value Driver" in enriched.injected_rubrics["bvf"]
    assert "raid" in enriched.injected_rubrics


def test_pack_loader_rejects_invalid_pack_yaml_schema(tmp_path: Path) -> None:
    bad_pack_dir = tmp_path / "broken_pack"
    bad_pack_dir.mkdir(parents=True)
    (bad_pack_dir / "pack.yaml").write_text(
        "uri: chotu://pack/frameworks/broken@1.0.0\n"
        "name: broken\n"
        "version: 1.0.0\n"
        "target_pillars:\n"
        "  - astrology\n"
        "target_agents:\n"
        "  - chotu://agent/architects/business-architect@1.0.0\n"
        "rubric_file: missing_rubric.md\n"
        "ontology_types: []\n",
        encoding="utf-8",
    )

    loader = PackLoader(packs_root=tmp_path, skills_root=tmp_path)
    with pytest.raises(InvalidPackManifestError):
        loader.load_pack(bad_pack_dir)
```

- [ ] **Step 2: Run test to verify it fails**

Run: `uv run pytest backend/tests/inventory/test_skills_packs_and_injector_hook.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'backend.inventory.packs.loader'`

- [ ] **Step 3: Implement `PackLoader`, the `bvf`/`raid`/`baits` packs, the 11 `csa-*` skill manifests, and `SkillAndPackInjectorHook`**

1. Create `pack.yaml` and `rubric.md` inside `backend/inventory/packs/bvf/`, `backend/inventory/packs/raid/`, and `backend/inventory/packs/baits/`, plus `backend/inventory/skills/csa_skills_manifest.yaml` defining the 11 `csa-*` skills and their I/O schemas.
2. In `backend/inventory/packs/loader.py`, implement `PackLoader.load_pack(pack_dir: Path) -> CapabilityPackManifest` using Pydantic validation so any pillar outside `{"business", "application", "information", "technology", "security"}` or any non-existent `rubric_file` raises `InvalidPackManifestError`.
3. In `backend/inventory/hooks/skill_pack_injector.py`, implement `SkillAndPackInjectorHook.pre_subagent_dispatch(envelope: SubagentDispatchEnvelope) -> SubagentDispatchEnvelope` to match `envelope.active_packs` and `envelope.baits_pillars` against loaded packs and populate `envelope.injected_rubrics`.

- [ ] **Step 4: Run test to verify it passes**

Run: `uv run pytest backend/tests/inventory/test_skills_packs_and_injector_hook.py -v`
Expected: PASS (2 passed)

- [ ] **Step 5: Commit**

```bash
git add backend/inventory/skills/ \
        backend/inventory/packs/ \
        backend/inventory/hooks/ \
        backend/tests/inventory/test_skills_packs_and_injector_hook.py
git commit -m "feat(inventory): add operationalized csa-* skills, bvf/raid/baits capability packs, and pre_subagent_dispatch injector hook"
```

---

### Task 6: 3-Tier Subconscious Background Daemon (`SubconsciousScheduler` & Daily Executive Briefing)

While the Conscious Mind handles your live interactive requests on the `P0` and `P1` lanes, Chotu's Subconscious Mind works continuously in the background across three cadences:
- **Tier 1 (Continuous Event-Driven Outbox on `P2`)**: Picks up raw file drops (`on_raw_file_dropped`) and newly persisted facts (`on_facts_persisted`) to run `csa-ingest` $\rightarrow$ `narrative-writer` $\rightarrow$ `csa-extract` $\rightarrow$ `csa-concept-writer` $\rightarrow$ `okf-indexer` $\rightarrow$ `truth_finder_agent`.
- **Tier 2 (Periodic Consolidation Heartbeat on `P3`)**: Refreshes stale BAITS synthesis views flagged in `.csa/synthesis-status.md`, promotes `CANDIDATE` dynamic workflows to `ESTABLISHED` once they hit 3 consecutive verified runs, and runs `vault_hygiene`.
- **Tier 3 (Scheduled Morning Executive Briefing on `P3`)**: Compiles overnight ingestion summaries, active multi-perspective Truth Finder Make/Break impact diffs, and tactical prep for your ~10 daily meetings.

Two safeguards keep this background daemon from ever degrading your interactive experience: first, `SubconsciousScheduler.tick_p3()` checks `TaskQueuePort.count_pending(lanes=["P0", "P1"])` before every `P3` job and immediately yields if any interactive or active workflow node is waiting (Review Focus #5); second, every background workflow is stamped with `origin="SUBCONSCIOUS"` and `hop_depth=1` so observer hooks never trigger an infinite recursion loop.

**Files:**
- Create: `backend/platform/harness/domain/subconscious.py`
- Create: `backend/platform/harness/adapters/subconscious_scheduler.py`
- Create: `backend/inventory/workflows/daily_executive_briefing.yaml`
- Test: `backend/tests/harness/test_subconscious_scheduler_and_briefing.py`

**Interfaces:**
- Consumes:
  - `TaskQueuePort.count_pending(lanes: list[Literal["P0", "P1", "P2", "P3"]]) -> int`
  - `TaskQueuePort.dequeue(lane: Literal["P0", "P1", "P2", "P3"]) -> QueuedTask | None`
  - `WorkflowRegistryPort.promote_verified_candidates(min_consecutive_successes: int = 3) -> list[str]`
- Produces:
  - `DailyExecutiveBriefing(date: str, engagements_summarized: list[str], ingested_sources_count: int, truth_finder_alerts: list[dict[str, Any]], meeting_prep_items: list[dict[str, str]], promoted_workflows: list[str], markdown_briefing: str)`
  - `SubconsciousScheduler.tick_p2_outbox() -> int`
  - `SubconsciousScheduler.tick_p3_consolidation() -> SubconsciousConsolidationReport`
  - `SubconsciousScheduler.generate_daily_executive_briefing(date_str: str) -> DailyExecutiveBriefing`

- [ ] **Step 1: Write the failing test**

Create `backend/tests/harness/test_subconscious_scheduler_and_briefing.py` testing: (a) Review Focus #5 (`tick_p3_consolidation` immediately yielding when `P0` interactive tasks are pending), (b) probationary workflow graduation after 3 consecutive verified runs and origin-tagged recursion fencing (`origin="SUBCONSCIOUS"`, `hop_depth=1`), and (c) Tier-3 `generate_daily_executive_briefing` surfacing Truth Finder Make/Break impact diffs and meeting prep items:

```python
from __future__ import annotations

import pytest

from backend.platform.harness.adapters.subconscious_scheduler import SubconsciousScheduler


@pytest.mark.asyncio
async def test_subconscious_p3_yields_when_p0_interactive_queue_is_non_empty(
    fake_four_lane_queue,
    fake_workflow_registry,
    fake_briefing_data_source,
) -> None:
    fake_four_lane_queue.enqueue(lane="P0", task_id="interactive-turn-1", payload={"text": "urgent"})
    fake_four_lane_queue.enqueue(lane="P3", task_id="bulk-synth-1", payload={"engagement": "acme-cloud"})

    scheduler = SubconsciousScheduler(
        queue=fake_four_lane_queue,
        workflow_registry=fake_workflow_registry,
        briefing_source=fake_briefing_data_source,
    )

    report = await scheduler.tick_p3_consolidation()
    assert report.yielded_to_foreground is True
    assert report.p3_tasks_processed == 0
    assert fake_four_lane_queue.count_pending(lanes=["P3"]) == 1


@pytest.mark.asyncio
async def test_subconscious_graduates_candidate_workflows_and_enforces_origin_fence(
    fake_four_lane_queue,
    fake_workflow_registry,
    fake_briefing_data_source,
) -> None:
    fake_workflow_registry.seed_candidate("wf-baits-dual-review", consecutive_verified_runs=3)
    fake_workflow_registry.seed_candidate("wf-flaky-draft", consecutive_verified_runs=2)
    fake_four_lane_queue.enqueue(
        lane="P3",
        task_id="subconscious-synth-1",
        payload={"engagement": "acme-cloud", "origin": "SUBCONSCIOUS", "hop_depth": 1},
    )

    scheduler = SubconsciousScheduler(
        queue=fake_four_lane_queue,
        workflow_registry=fake_workflow_registry,
        briefing_source=fake_briefing_data_source,
    )

    report = await scheduler.tick_p3_consolidation()
    assert report.yielded_to_foreground is False
    assert report.promoted_workflows == ["wf-baits-dual-review"]
    assert report.spawned_followup_workflows == 0  # Blocked by origin == SUBCONSCIOUS fence


@pytest.mark.asyncio
async def test_subconscious_generates_morning_executive_briefing(
    fake_four_lane_queue,
    fake_workflow_registry,
    fake_briefing_data_source,
) -> None:
    fake_briefing_data_source.seed_overnight_state(
        engagement_slug="acme-cloud",
        ingested_count=6,
        conflict_alert={
            "concept": "legacy-billing",
            "client_stated": "Q2 2027",
            "internal_team": "Q4 2027",
            "breaks_sow_milestones": 3,
            "plausibility_winner": "Q4 2027 (0.82)",
        },
        meeting_prep={"meeting": "09:30 Architecture Board", "question": "Confirm Q2 vs Q4 mainframe exit penalty"},
    )

    scheduler = SubconsciousScheduler(
        queue=fake_four_lane_queue,
        workflow_registry=fake_workflow_registry,
        briefing_source=fake_briefing_data_source,
    )

    briefing = await scheduler.generate_daily_executive_briefing("2026-10-28")
    assert briefing.ingested_sources_count == 6
    assert len(briefing.truth_finder_alerts) == 1
    assert "legacy-billing" in briefing.markdown_briefing
    assert "09:30 Architecture Board" in briefing.markdown_briefing
```

- [ ] **Step 2: Run test to verify it fails**

Run: `uv run pytest backend/tests/harness/test_subconscious_scheduler_and_briefing.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'backend.platform.harness.adapters.subconscious_scheduler'`

- [ ] **Step 3: Implement `SubconsciousScheduler` and `daily_executive_briefing.yaml`**

1. In `backend/platform/harness/domain/subconscious.py`, define `SubconsciousConsolidationReport` and `DailyExecutiveBriefing`.
2. In `backend/platform/harness/adapters/subconscious_scheduler.py`, implement `SubconsciousScheduler`:
   - `tick_p2_outbox()`: drains `P2` outbox events for near-real-time `okf-indexer` projection and `truth_finder_agent` Make/Break weighing.
   - `tick_p3_consolidation()`: checks `if self._queue.count_pending(lanes=["P0", "P1"]) > 0:` first and returns `SubconsciousConsolidationReport(yielded_to_foreground=True, p3_tasks_processed=0)` immediately. Otherwise processes `P3` tasks with `origin="SUBCONSCIOUS"`, `hop_depth=1` (suppressing any recursive child workflow spawning when `origin in {"HOOK", "SUBCONSCIOUS"}`), and calls `self._workflow_registry.promote_verified_candidates(min_consecutive_successes=3)`.
   - `generate_daily_executive_briefing(date_str: str)`: aggregates overnight ingestion metrics, Truth Finder Make/Break impact diffs, and daily meeting prep into a structured `DailyExecutiveBriefing`.
3. Create `backend/inventory/workflows/daily_executive_briefing.yaml` defining the declarative workflow template.

- [ ] **Step 4: Run test to verify it passes**

Run: `uv run pytest backend/tests/harness/test_subconscious_scheduler_and_briefing.py -v`
Expected: PASS (3 passed)

- [ ] **Step 5: Commit**

```bash
git add backend/platform/harness/domain/subconscious.py \
        backend/platform/harness/adapters/subconscious_scheduler.py \
        backend/inventory/workflows/daily_executive_briefing.yaml \
        backend/tests/harness/test_subconscious_scheduler_and_briefing.py
git commit -m "feat(harness): add 3-tier SubconsciousScheduler with P0/P1 priority yielding, DAG graduation, and Daily Executive Briefing"
```

---

### Task 7: Multi-Channel Comms Adapters & `ExternalAppAdapter` (`dsapps-journal` & `dsapps-checkin`)

Our final task in Slice Group 3 connects Chotu Corp to the outside world across two boundaries:
1. **Multi-Channel Operator Comms (`backend/platform/harness/adapters/comms/`)**: Implements `ChotuCLIAdapter`, `FastAPIRestSSEAdapter`, `ObsidianWorkspaceWatcherAdapter`, and `TelegramBotAdapter` against `ChannelAdapterPort` so whether you drop a file into `workspaces/<slug>/raw/`, edit an Obsidian note in `okf-wiki/`, send a Telegram voice note between meetings, or run a command in the Web Portal, it flows through `WorkClassifierPort` $\rightarrow$ `LeadOrchestrator` (or `P2` outbox for file watchers).
2. **Independent External Apps (`ExternalAppAdapter` in `backend/platform/tools/adapters/external_app_adapter.py`)**: Onboards standalone external applications—starting with [`dsapps-journal`](file:///home/nox/Developer/ds2/dsapps-journal/CLAUDE.md) (`chotu://app/external/dsapps-journal@1.0.0`) and [`dsapps-checkin`](file:///home/nox/Developer/ds2/dsapps-checkin/README.md) (`chotu://app/external/dsapps-checkin@1.0.0`)—over HTTP/MCP/CLI without merging their repositories into `dsa-agent`. Because external apps run as separate processes that can restart or hang, `ExternalAppAdapter` implements a per-app circuit breaker (`CLOSED` $\rightarrow$ `OPEN` after 3 consecutive failures/timeouts) so an unresponsive external app degrades gracefully instead of stalling a workflow (Review Focus #5).

**Files:**
- Create: `backend/platform/harness/ports/channel_port.py`
- Create: `backend/platform/harness/adapters/comms/cli_adapter.py`
- Create: `backend/platform/harness/adapters/comms/http_sse_adapter.py`
- Create: `backend/platform/harness/adapters/comms/workspace_watcher_adapter.py`
- Create: `backend/platform/harness/adapters/comms/telegram_adapter.py`
- Create: `backend/platform/tools/adapters/external_app_adapter.py`
- Create: `backend/inventory/tools/external_apps/dsapps_manifests.yaml`
- Test: `backend/tests/harness/test_comms_and_external_app_circuit_breaker.py`

**Interfaces:**
- Consumes:
  - `LeadOrchestrator.handle_turn(request: InboundWorkRequest) -> WorkflowRun`
  - `TaskQueuePort.enqueue(lane: Literal["P0", "P1", "P2", "P3"], task_id: str, payload: dict[str, Any]) -> None`
- Produces:
  - `ChannelAdapterPort.receive_and_route(raw_event: dict[str, Any]) -> str`
  - `ExternalAppInvocationResult(app_uri: str, ok: bool, circuit_state: Literal["CLOSED", "OPEN", "HALF_OPEN"], data: dict[str, Any] | None, fallback_reason: str | None)`
  - `ExternalAppAdapter.invoke_app(app_uri: str, action: str, payload: dict[str, Any]) -> ExternalAppInvocationResult`

- [ ] **Step 1: Write the failing test**

Create `backend/tests/harness/test_comms_and_external_app_circuit_breaker.py` testing: (a) `ObsidianWorkspaceWatcherAdapter` routing `raw/` file drops and `okf-wiki/` Markdown edits to the `P2` outbox lane with `origin="SUBCONSCIOUS"`, (b) `TelegramBotAdapter` and `ChotuCLIAdapter` routing interactive messages and inline `ApprovalBus` callbacks on `P0`, and (c) Review Focus #5 (`ExternalAppAdapter` invoking `dsapps-journal` / `dsapps-checkin` and tripping its circuit breaker to `OPEN` after 3 consecutive timeouts):

```python
from __future__ import annotations

import httpx
import pytest

from backend.platform.harness.adapters.comms.telegram_adapter import TelegramBotAdapter
from backend.platform.harness.adapters.comms.workspace_watcher_adapter import (
    ObsidianWorkspaceWatcherAdapter,
)
from backend.platform.tools.adapters.external_app_adapter import ExternalAppAdapter


@pytest.mark.asyncio
async def test_workspace_watcher_enqueues_raw_drops_and_wiki_edits_to_p2_outbox(
    fake_four_lane_queue,
) -> None:
    watcher = ObsidianWorkspaceWatcherAdapter(queue=fake_four_lane_queue)

    await watcher.on_filesystem_event(
        event_type="created",
        path_str="/workspaces/acme-cloud/raw/2026-10-28-call.vtt",
    )
    await watcher.on_filesystem_event(
        event_type="modified",
        path_str="/workspaces/acme-cloud/okf-wiki/legacy-billing.md",
    )

    assert fake_four_lane_queue.count_pending(lanes=["P2"]) == 2
    first_task = fake_four_lane_queue.dequeue(lane="P2")
    assert first_task is not None
    assert first_task.payload["engagement_slug"] == "acme-cloud"
    assert first_task.payload["origin"] == "SUBCONSCIOUS"


@pytest.mark.asyncio
async def test_telegram_adapter_routes_voice_memo_and_approval_callback(
    fake_orchestrator,
    fake_approval_bus,
) -> None:
    ticket = fake_approval_bus.create_pending_ticket("chotu://tool/custom/profiler@1.0.0")
    tg = TelegramBotAdapter(orchestrator=fake_orchestrator, approval_bus=fake_approval_bus)

    await tg.handle_webhook_update(
        {
            "callback_query": {
                "data": f"approve:{ticket.ticket_id}",
                "from": {"username": "nox_ceo"},
            }
        }
    )
    assert fake_approval_bus.is_approved(ticket.ticket_id) is True


@pytest.mark.asyncio
async def test_external_app_adapter_trips_circuit_breaker_on_timeout() -> None:
    call_count = 0

    async def timeout_transport(url: str, payload: dict[str, object]) -> dict[str, object]:
        nonlocal call_count
        call_count += 1
        raise httpx.ReadTimeout("dsapps-journal HTTP endpoint timed out")

    adapter = ExternalAppAdapter(
        http_transport=timeout_transport,
        failure_threshold=3,
    )
    adapter.register_external_app(
        uri="chotu://app/external/dsapps-journal@1.0.0",
        transport="http",
        endpoint="http://localhost:4100/api/entries",
    )

    for _ in range(3):
        res = await adapter.invoke_app(
            app_uri="chotu://app/external/dsapps-journal@1.0.0",
            action="create_entry",
            payload={"title": "EOD Reflection"},
        )
        assert res.ok is False

    # 4th call must short-circuit immediately with circuit_state == "OPEN" without hitting transport
    fourth = await adapter.invoke_app(
        app_uri="chotu://app/external/dsapps-journal@1.0.0",
        action="create_entry",
        payload={"title": "EOD Reflection 2"},
    )
    assert fourth.ok is False
    assert fourth.circuit_state == "OPEN"
    assert call_count == 3
```

- [ ] **Step 2: Run test to verify it fails**

Run: `uv run pytest backend/tests/harness/test_comms_and_external_app_circuit_breaker.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'backend.platform.harness.adapters.comms.telegram_adapter'`

- [ ] **Step 3: Implement the 4 Comms Adapters, `ExternalAppAdapter` with circuit breaker, and `dsapps_manifests.yaml`**

1. Create `backend/platform/harness/ports/channel_port.py` defining `ChannelAdapterPort`.
2. Implement `ChotuCLIAdapter`, `FastAPIRestSSEAdapter`, `ObsidianWorkspaceWatcherAdapter`, and `TelegramBotAdapter` under `backend/platform/harness/adapters/comms/`. Ensure `ObsidianWorkspaceWatcherAdapter` parses `/workspaces/<slug>/raw/...` and `/workspaces/<slug>/okf-wiki/...` paths and enqueues tasks to the `P2` lane with `origin="SUBCONSCIOUS"` and `hop_depth=1`.
3. In `backend/platform/tools/adapters/external_app_adapter.py`, implement `ExternalAppAdapter` supporting `http`, `mcp`, and `cli` transports with per-URI failure counters: when `consecutive_failures >= failure_threshold` (default `3`), transition the app's circuit state to `"OPEN"` and immediately return `ExternalAppInvocationResult(app_uri=app_uri, ok=False, circuit_state="OPEN", data=None, fallback_reason="Circuit breaker OPEN after 3 consecutive failures")` without invoking the underlying network transport until a health heartbeat resets it.
4. Create `backend/inventory/tools/external_apps/dsapps_manifests.yaml` registering `chotu://app/external/dsapps-journal@1.0.0` and `chotu://app/external/dsapps-checkin@1.0.0`.

- [ ] **Step 4: Run test to verify it passes**

Run: `uv run pytest backend/tests/harness/test_comms_and_external_app_circuit_breaker.py -v`
Expected: PASS (3 passed)

- [ ] **Step 5: Commit**

```bash
git add backend/platform/harness/ports/channel_port.py \
        backend/platform/harness/adapters/comms/ \
        backend/platform/tools/adapters/external_app_adapter.py \
        backend/inventory/tools/external_apps/dsapps_manifests.yaml \
        backend/tests/harness/test_comms_and_external_app_circuit_breaker.py
git commit -m "feat(comms,tools): add multi-channel comms adapters and ExternalAppAdapter with circuit breaker for dsapps-*"
```
