# Slice Group 1: Platform Microkernel Chassis, 4-Lane Master Queue, Service Locator & Sovereign Harness Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the domain-agnostic 9-layer `backend/platform/` microkernel chassis, the `core.*` PostgreSQL 16+ schema, the 4-Lane Priority Master Task Queue + Transactional Outbox, the Per-Node Durable DAG State Machine, the 3-Stage Runtime Service Locator & Discovery engine, the `EngagementWorkspaceJail`, the Dual-Mode Lifecycle Hook Registry, and the two sovereign execution chokepoints (`AgentRuntime.invoke()` and `ToolRegistry.invoke()`).

**Architecture:** Every subsystem under `backend/platform/` (`brain`, `tools`, `orchestration`, `agentic_loop`, `harness`, `memory`, `knowledge`, `environment`, `multiagent`) follows strict Fractal Hexagonal (`domain/ | ports/ | adapters/`) isolation where `domain/` contains pure Python 3.12+ OOP and Pydantic v2 models with zero vendor SDK, FastAPI, or database driver imports. Core platform infrastructure ports are wired via Constructor Dependency Injection in `backend/platform/harness/bootstrap.py`, while runtime capabilities (`chotu://<kind>/<namespace>/<name>@<version>`) resolve dynamically through `ServiceLocatorPort` and `ServiceDiscoveryPort` behind the two invocation chokepoints.

**Tech Stack:** Python 3.12+, `uv`, Pydantic v2, `mypy --strict`, `ruff`, `pytest` + `pytest-asyncio`, PostgreSQL 16+ (`psycopg[binary,pool]` v3 async + `pgvector`), Alembic, FastAPI, OpenTelemetry SDK (`opentelemetry-api`, `opentelemetry-sdk`).

**Spec:** [`docs/planning/slice-01-platform-microkernel-queues-and-discovery.md`](file:///home/nox/Developer/ds2/dsa-agent/docs/planning/slice-01-platform-microkernel-queues-and-discovery.md)

## Global Constraints

- **Two-Tree Boundary (`platform/` vs. `inventory/`)**: `backend/platform/` is the domain-agnostic chassis and must **never** import `backend/inventory/`.
- **Zero Vendor SDKs in `domain/`**: No file matching `backend/platform/*/domain/**/*.py` may import `fastapi`, `sqlalchemy`, `psycopg`, `asyncpg`, `litellm`, `openai`, `anthropic`, or `google.genai`. Enforced by automated AST/grep test gate.
- **Port-Contract Testing Mandate**: Every port (`<Thing>Port`) in `backend/platform/*/ports/` must ship with a shared contract test class or parametrized fixture that runs against both its `InMemory*Adapter` and its `Postgres*Adapter`.
- **Canonical URI Format**: All dynamic capabilities use `chotu://<kind>/<namespace>/<name>@<version>` where `<kind> in {"agent", "skill", "tool", "pack", "sidecar", "app"}` and `<version>` is strict SemVer (`MAJOR.MINOR.PATCH`).
- **4-Lane Priority Queue Rules**: `P0 interactive` (`priority=0`), `P1 workflow_dag` (`priority=1`), `P2 outbox_hooks` (`priority=2`), `P3 subconscious_bulk` (`priority=3`). Reserved `P0` worker claims only `P0`; general workers claim `P0 -> P1 -> P2 -> P3` using `SELECT ... FOR UPDATE SKIP LOCKED`, and `P3` yields whenever `P0` or `P1` has queued rows.
- **Subagents-Only Invariant**: The Lead Orchestrator (`chotu://agent/lead/chotu@...`) is structurally forbidden from invoking any `chotu://tool/...` capability through `ToolRegistry.invoke()`.
- **Origin-Tagged Hook Recursion Fence**: Every `WorkflowRun` and hook event carries `origin: WorkflowOrigin` (`USER | HOOK | SUBCONSCIOUS`) and `hop_depth: int`. Async observer hooks may only spawn follow-up workflows when `origin == WorkflowOrigin.USER` and `hop_depth == 0`.
- **Probationary DAG Crystallization**: Newly composed dynamic `WorkflowPlan` DAGs start at `WorkflowPlanStatus.CANDIDATE` (`verified_success_count = 0`) and only graduate to `WorkflowPlanStatus.ESTABLISHED` after `3` consecutive verified executions without policy, citation, or node failures (any failure resets the streak to `0`), or via explicit operator promotion.

## Review Focus

1. **P0 Interactive Starvation Under P3 Bulk Flood**: When 50+ bulk ingestion tasks (`P3 subconscious_bulk`) are queued and general workers are busy, a newly enqueued live operator message (`P0 interactive`) must be claimed by the reserved interactive worker in `<50ms` and general workers must preempt `P3` in favor of `P0`/`P1` (`test_reserved_p0_worker_never_starves_under_p3_bulk_flood` in **Task 2**).
2. **Concurrent Parallel DAG Fan-In Lost Update / Double Enqueue**: When two parallel subagent branches (`business_architect` and `security_architect`) complete at the exact same time and both share a downstream join node (`csa_reviewer`), the join node must transition from `PENDING` to `READY` exactly once without lost updates or duplicate `P1` task rows (`test_concurrent_parallel_branches_fan_in_without_race_or_double_enqueue` in **Task 4**).
3. **Infinite Hook Storm via `on_workflow_complete` Recursion**: When a background workflow spawned by an observer hook (`origin=WorkflowOrigin.HOOK, hop_depth=1`) finishes and emits `on_workflow_complete`, the hook engine must record telemetry only and refuse to enqueue a third-generation workflow (`test_observer_hook_recursion_fence_blocks_non_user_or_nonzero_hop` in **Task 6**).
4. **Symlink & Relative Path Escape Out of `EngagementWorkspaceJail`**: When a subagent running inside `workspaces/acme-cloud/` invokes a tool with `../../globex-corp/secret.md`, `$CHOTU_VAULT_PATH/index.md`, or a symlink inside `workspaces/acme-cloud/` that resolves outside the engagement root, `EngagementWorkspaceJail` must block the call before execution (`test_workspace_jail_blocks_dotdot_and_symlink_escape_to_sibling_or_vault` in **Task 6**).
5. **Lead Orchestrator Direct Tool Invocation Bypass**: When ` caller_agent_uri="chotu://agent/lead/chotu@1.0.0"` attempts to call `ToolRegistry.invoke("chotu://tool/core/bash@1.0.0", ...)`, the chokepoint must immediately raise `SubagentsOnlyViolationError` before running any adapter (`test_lead_orchestrator_direct_tool_call_raises_subagents_only_violation` in **Task 7**).

---

### Task 1: Core Domain Value Objects, `CanonicalUri` Parser & Zero-Vendor Architecture Guard

Before we touch queues, DAG runners, or service registries, we need the shared vocabulary of pure domain types that every layer of `backend/platform/` speaks. If each layer invents its own string formats for `chotu://` URIs, priority lanes, scope tags, or perspective labels, adapters will spend half their code translating strings and silently dropping fields.

This first task creates the foundational immutable value objects (`CanonicalUri`, `CapabilityDescriptor`, `TaskLane`, `ScopeKind`, `PerspectiveKind`, `BaitsPillar`, `DepartmentKind`, `LifecyclePolicy`, and `ExecutionContext`) across the `domain/` packages of `multiagent`, `harness`, and `orchestration`, alongside the automated AST/import check that enforces our "Zero Vendor SDKs in `domain/`" rule on every test run.

**Files:**
- Create: `backend/platform/multiagent/domain/uris.py`
- Create: `backend/platform/multiagent/domain/capability.py`
- Create: `backend/platform/harness/domain/lanes.py`
- Create: `backend/platform/orchestration/domain/taxonomy.py`
- Create: `backend/tests/platform/test_domain_purity_and_uris.py`

**Interfaces:**
- Consumes: Python 3.12 standard library (` dataclasses`, `enum`, `re`, `ast`, `pathlib`, `datetime`, `uuid`) and `pydantic.BaseModel` / `ConfigDict`.
- Produces:
  - `CanonicalUri.parse(raw: str) -> CanonicalUri` with fields `kind: CapabilityKind`, `namespace: str`, `name: str`, `version: str`, property `unversioned_key: str` (`"chotu://<kind>/<namespace>/<name>"`), and `__str__() -> str`.
  - `CapabilityKind(StrEnum)`: `AGENT = "agent"`, `SKILL = "skill"`, `TOOL = "tool"`, `PACK = "pack"`, `SIDECAR = "sidecar"`, `APP = "app"`.
  - `LifecyclePolicy(StrEnum)`: `ALWAYS_ON = "always_on"`, `LAZY_START = "lazy_start"`, `EXTERNAL_MANAGED = "external_managed"`.
  - `HealthStatus(StrEnum)`: `HEALTHY = "HEALTHY"`, `SLEEPING = "SLEEPING"`, `UNHEALTHY = "UNHEALTHY"`, `DISABLED = "DISABLED"`.
  - `CapabilityDescriptor(BaseModel, frozen=True)`: `uri: CanonicalUri`, `description: str`, `department: DepartmentKind`, `baits_pillar: BaitsPillar | None`, `trigger_phrases: tuple[str, ...]`, `input_schema: dict[str, object]`, `output_schema: dict[str, object]`, `applies_to_kinds: tuple[str, ...]`, `permissions: tuple[str, ...]`, `risk_level: RiskLevel`, `lifecycle_policy: LifecyclePolicy`, `fallback_uri: CanonicalUri | None = None`, `health_status: HealthStatus = HealthStatus.HEALTHY`, `disabled_reason: str | None = None`.
  - `TaskLane( IntEnum)`: `P0_INTERACTIVE = 0`, `P1_WORKFLOW_DAG = 1`, `P2_OUTBOX_HOOKS = 2`, `P3_SUBCONSCIOUS_BULK = 3`.
  - `ScopeKind(StrEnum)`: `ENGAGEMENT = "engagement"`, `CLIENT = "client"`, `INTERNAL = "internal"`, `REFERENCE = "reference"`, `PRIOR_PROJECT = "prior-project"`, `DELIVERABLE = "deliverable"`.
  - `PerspectiveKind(StrEnum)`: `CLIENT_STATED = "client_stated"`, `INTERNAL_TEAM = "internal_team"`, `PERSONAL_TAKE = "personal_take"`, `WORK_RESEARCH = "work_research"`, `PERSONAL_RD = "personal_rd"`.
  - `OperatingMode(StrEnum)`: `WORK_ENGAGEMENT = "work_engagement"`, `PERSONAL_RD = "personal_rd"`, `UNIVERSAL_RESEARCH = "universal_research"`.
  - `DepartmentKind(StrEnum)`: `EXECUTIVE = "executive"`, `KNOWLEDGE = "knowledge"`, `DELIVERY = "delivery"`, `ENGINEERING = "engineering"`, `INTELLIGENCE = "intelligence"`.
  - `BaitsPillar(StrEnum)`: `BUSINESS = "business"`, `APPLICATION = "application"`, `INFORMATION = "information"`, `TECHNOLOGY = "technology"`, `SECURITY = "security"`.
  - `RiskLevel(StrEnum)`: `LOW = "low"`, `MEDIUM = "medium"`, `HIGH = "high"`, `CRITICAL = "critical"`.
  - `WorkflowOrigin(StrEnum)`: `USER = "USER"`, `HOOK = "HOOK"`, `SUBCONSCIOUS = "SUBCONSCIOUS"`.

- [ ] **Step 1: Write the failing test**

Create `backend/tests/platform/test_domain_purity_and_uris.py`:

```python
import ast
from pathlib import Path
import pytest

from backend.platform.multiagent.domain.uris import CanonicalUri, CapabilityKind, InvalidCapabilityUriError
from backend.platform.multiagent.domain.capability import (
    CapabilityDescriptor,
    HealthStatus,
    LifecyclePolicy,
)
from backend.platform.harness.domain.lanes import TaskLane
from backend.platform.orchestration.domain.taxonomy import (
    BaitsPillar,
    DepartmentKind,
    OperatingMode,
    PerspectiveKind,
    RiskLevel,
    ScopeKind,
    WorkflowOrigin,
)

FORBIDDEN_DOMAIN_IMPORTS = {
    "fastapi",
    "sqlalchemy",
    "psycopg",
    "asyncpg",
    "litellm",
    "openai",
    "anthropic",
    "google",
    "httpx",
}


def test_canonical_uri_parses_valid_uri_and_rejects_malformed() -> None:
    uri = CanonicalUri.parse("chotu://agent/architects/business@1.2.0")
    assert uri.kind == CapabilityKind.AGENT
    assert uri.namespace == "architects"
    assert uri.name == "business"
    assert uri.version == "1.2.0"
    assert uri.unversioned_key == "chotu://agent/architects/business"
    assert str(uri) == "chotu://agent/architects/business@1.2.0"

    for bad in [
        "http://agent/architects/business@1.0.0",
        "chotu://unknown_kind/core/grep@1.0.0",
        "chotu://tool/core/grep",  # missing @version
        "chotu://tool/core/grep@v1.0",  # non-semver
        "chotu://tool//grep@1.0.0",
    ]:
        with pytest.raises(InvalidCapabilityUriError):
            CanonicalUri.parse(bad)


def test_capability_descriptor_and_taxonomy_enums() -> None:
    primary = CanonicalUri.parse("chotu://sidecar/ingestion/docling@1.0.0")
    fallback = CanonicalUri.parse("chotu://tool/extractors/parse-pdf-local@1.0.0")
    desc = CapabilityDescriptor(
        uri=primary,
        description="Docling layout-aware document parser sidecar",
        department=DepartmentKind.KNOWLEDGE,
        baits_pillar=BaitsPillar.INFORMATION,
        trigger_phrases=("parse pdf", "docling extract"),
        input_schema={"type": "object", "required": ["file_path"]},
        output_schema={"type": "object", "required": ["markdown"]},
        applies_to_kinds=("pdf", "docx", "pptx"),
        permissions=("workspace:read",),
        risk_level=RiskLevel.LOW,
        lifecycle_policy=LifecyclePolicy.LAZY_START,
        fallback_uri=fallback,
    )
    assert desc.health_status == HealthStatus.HEALTHY
    assert desc.fallback_uri == fallback
    assert TaskLane.P0_INTERACTIVE < TaskLane.P1_WORKFLOW_DAG < TaskLane.P2_OUTBOX_HOOKS < TaskLane.P3_SUBCONSCIOUS_BULK
    assert ScopeKind.PRIOR_PROJECT.value == "prior-project"
    assert PerspectiveKind.CLIENT_STATED.value == "client_stated"
    assert OperatingMode.WORK_ENGAGEMENT.value == "work_engagement"
    assert WorkflowOrigin.USER.value == "USER"


def test_zero_vendor_sdks_or_inventory_imports_in_platform_domain() -> None:
    platform_root = Path(__file__).resolve().parents[2] / "platform"
    domain_files = list(platform_root.glob("*/domain/**/*.py"))
    assert len(domain_files) >= 4

    for py_file in domain_files:
        tree = ast.parse(py_file.read_text(encoding="utf-8"), filename=str(py_file))
        for node in ast.walk(tree):
            if isinstance(node, ast.Import):
                for alias in node.names:
                    top = alias.name.split(".")[0]
                    assert top not in FORBIDDEN_DOMAIN_IMPORTS, f"Forbidden import {alias.name} in {py_file}"
                    assert "inventory" not in alias.name.split("."), f"Platform imported inventory in {py_file}"
            elif isinstance(node, ast.ImportFrom) and node.module:
                top = node.module.split(".")[0]
                assert top not in FORBIDDEN_DOMAIN_IMPORTS, f"Forbidden import from {node.module} in {py_file}"
                assert "inventory" not in node.module.split("."), f"Platform imported inventory in {py_file}"
```

- [ ] **Step 2: Run test to verify it fails**

Run: `uv run pytest backend/tests/platform/test_domain_purity_and_uris.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'backend.platform.multiagent.domain.uris'`

- [ ] **Step 3: Implement `CanonicalUri`, `CapabilityDescriptor`, `TaskLane`, and taxonomy enums**

Implement the four domain modules with strict regex validation (`^chotu://(?P<kind>agent|skill|tool|pack|sidecar|app)/(?P<namespace>[a-z0-9_-]+)/(?P<name>[a-z0-9_-]+)@(?P<version>\d+\.\d+\.\d+)$`) in `CanonicalUri.parse`, plus custom Pydantic serialization so `CanonicalUri` accepts either a `CanonicalUri` instance or a valid URI string when constructing `CapabilityDescriptor`.

- [ ] **Step 4: Run test to verify it passes**

Run: `uv run pytest backend/tests/platform/test_domain_purity_and_uris.py -v && uv run mypy --strict backend/platform`
Expected: PASS (3 passed, 0 mypy issues)

- [ ] **Step 5: Commit**

```bash
git add backend/platform/multiagent/domain/ backend/platform/harness/domain/lanes.py backend/platform/orchestration/domain/taxonomy.py backend/tests/platform/test_domain_purity_and_uris.py
git commit -m "feat(platform): add core domain value objects, CanonicalUri parser, and zero-vendor AST gate"
```

---

### Task 2: 4-Lane Priority Master Task Queue (`TaskQueuePort` + In-Memory & Postgres `SKIP LOCKED` Adapters)

Now we tackle the operational heartbeat of Chotu Corp. When twenty dense documents and ten multi-perspective transcripts are being parsed and embedded on `P3 subconscious_bulk`, and you send a live question from a client meeting on `P0 interactive`, Chotu cannot make you wait behind bulk jobs. At the same time, if a worker process dies mid-task, that task cannot vanish into thin air.

This task defines `TaskQueuePort` (`backend/platform/harness/ports/task_queue_port.py`), builds both `InMemoryTaskQueueAdapter` and `PostgresTaskQueueAdapter` (plus the initial `core.*` Alembic migration `0001_core_platform_schema.py`), and verifies them against a shared Port-Contract Test Suite. Specifically, `claim_next(worker_id, reserved_for_p0, lease_seconds)` implements two crucial guarantees: (1) a reserved `P0` worker claims only `P0_INTERACTIVE` tasks so a live slot is always open, and (2) when a general worker looks for work, `P3_SUBCONSCIOUS_BULK` tasks are automatically skipped (`yielded`) whenever any `P0_INTERACTIVE` or `P1_WORKFLOW_DAG` task is currently `QUEUED` or `RUNNING` above the P3 concurrency budget.

**Files:**
- Create: `backend/migrations/versions/0001_core_platform_schema.py`
- Create: `backend/platform/harness/domain/task.py`
- Create: `backend/platform/harness/ports/task_queue_port.py`
- Create: `backend/platform/harness/adapters/in_memory_task_queue.py`
- Create: `backend/platform/harness/adapters/postgres_task_queue.py`
- Test: `backend/tests/platform/harness/test_task_queue_contract.py`

**Interfaces:**
- Consumes: `TaskLane` from `backend.platform.harness.domain.lanes`.
- Produces:
  - `TaskStatus(StrEnum)`: `QUEUED = "QUEUED"`, `RUNNING = "RUNNING"`, `COMPLETED = "COMPLETED"`, `FAILED = "FAILED"`, `DEAD_LETTER = "DEAD_LETTER"`, `CANCELLED = "CANCELLED"`.
  - `QueueTask(BaseModel)`: `task_id: UUID`, `lane: TaskLane`, `task_type: str`, `payload: dict[str, object]`, `idempotency_key: str`, `status: TaskStatus`, `attempt_count: int = 0`, `max_attempts: int = 3`, `worker_id: str | None = None`, `leased_until: datetime | None = None`, `next_run_at: datetime`, `last_error: str | None = None`, `created_at: datetime`.
  - `TaskQueuePort(ABC)`:
    - `async def enqueue(self, *, lane: TaskLane, task_type: str, payload: dict[str, object], idempotency_key: str, max_attempts: int = 3) -> QueueTask` (idempotent on `idempotency_key`: returns existing task if already present).
    - `async def claim_next(self, *, worker_id: str, lease_seconds: int = 30, reserved_for_p0: bool = False, now: datetime | None = None) -> QueueTask | None`
    - `async def heartbeat(self, *, task_id: UUID, worker_id: str, extend_seconds: int = 30, now: datetime | None = None) -> bool`
    - `async def complete(self, *, task_id: UUID, worker_id: str) -> QueueTask`
    - `async def fail(self, *, task_id: UUID, worker_id: str, error: str, base_backoff_seconds: int = 5, now: datetime | None = None) -> QueueTask` (increments `attempt_count`; if `attempt_count >= max_attempts`, transitions to `DEAD_LETTER`, else transitions to `QUEUED` with `next_run_at = now + base_backoff_seconds * (2 ** (attempt_count - 1))`).
    - `async def reclaim_expired_leases(self, *, now: datetime | None = None) -> int`

- [ ] **Step 1: Write the failing test**

Create `backend/tests/platform/harness/test_task_queue_contract.py` (including Review Focus #1: `test_reserved_p0_worker_never_starves_under_p3_bulk_flood`):

```python
from datetime import datetime, timedelta, timezone
import time
import pytest

from backend.platform.harness.domain.lanes import TaskLane
from backend.platform.harness.domain.task import TaskStatus
from backend.platform.harness.ports.task_queue_port import TaskQueuePort
from backend.platform.harness.adapters.in_memory_task_queue import InMemoryTaskQueueAdapter


@pytest.fixture
def queue() -> TaskQueuePort:
    return InMemoryTaskQueueAdapter()


@pytest.mark.asyncio
async def test_idempotent_enqueue_and_strict_lane_priority_order(queue: TaskQueuePort) -> None:
    t_p3 = await queue.enqueue(
        lane=TaskLane.P3_SUBCONSCIOUS_BULK,
        task_type="parse_pdf",
        payload={"doc": "deck.pdf"},
        idempotency_key="pdf-1",
    )
    t_p3_dup = await queue.enqueue(
        lane=TaskLane.P3_SUBCONSCIOUS_BULK,
        task_type="parse_pdf",
        payload={"doc": "deck.pdf"},
        idempotency_key="pdf-1",
    )
    assert t_p3.task_id == t_p3_dup.task_id

    t_p2 = await queue.enqueue(
        lane=TaskLane.P2_OUTBOX_HOOKS,
        task_type="project_okf",
        payload={"concept": "iam"},
        idempotency_key="hook-1",
    )
    t_p1 = await queue.enqueue(
        lane=TaskLane.P1_WORKFLOW_DAG,
        task_type="run_node",
        payload={"node": "sec-arch"},
        idempotency_key="dag-1",
    )

    claimed_1 = await queue.claim_next(worker_id="worker-general-1")
    assert claimed_1 is not None and claimed_1.task_id == t_p1.task_id
    await queue.complete(task_id=claimed_1.task_id, worker_id="worker-general-1")

    claimed_2 = await queue.claim_next(worker_id="worker-general-1")
    assert claimed_2 is not None and claimed_2.task_id == t_p2.task_id
    await queue.complete(task_id=claimed_2.task_id, worker_id="worker-general-1")

    claimed_3 = await queue.claim_next(worker_id="worker-general-1")
    assert claimed_3 is not None and claimed_3.task_id == t_p3.task_id


@pytest.mark.asyncio
async def test_reserved_p0_worker_never_starves_under_p3_bulk_flood(queue: TaskQueuePort) -> None:
    # Flood queue with 50 P3 bulk document ingestion tasks
    for idx in range(50):
        await queue.enqueue(
            lane=TaskLane.P3_SUBCONSCIOUS_BULK,
            task_type="ingest_doc",
            payload={"index": idx},
            idempotency_key=f"bulk-{idx}",
        )

    # Reserved P0 worker ignores P3 completely and keeps slot open for live user turn
    assert await queue.claim_next(worker_id="worker-p0-reserved", reserved_for_p0=True) is None

    # Operator sends a live interactive request
    enqueued_p0 = await queue.enqueue(
        lane=TaskLane.P0_INTERACTIVE,
        task_type="user_turn",
        payload={"text": "What did the client say about SSO latency?"},
        idempotency_key="turn-live-1",
    )

    start = time.perf_counter()
    claimed_p0 = await queue.claim_next(worker_id="worker-p0-reserved", reserved_for_p0=True)
    elapsed_ms = (time.perf_counter() - start) * 1000.0

    assert claimed_p0 is not None
    assert claimed_p0.task_id == enqueued_p0.task_id
    assert claimed_p0.lane == TaskLane.P0_INTERACTIVE
    assert elapsed_ms < 50.0


@pytest.mark.asyncio
async def test_lease_expiry_reclamation_backoff_and_dead_letter(queue: TaskQueuePort) -> None:
    t0 = datetime(2026, 10, 28, 12, 0, 0, tzinfo=timezone.utc)
    task = await queue.enqueue(
        lane=TaskLane.P1_WORKFLOW_DAG,
        task_type="run_node",
        payload={"node": "bvf"},
        idempotency_key="lease-test-1",
        max_attempts=2,
    )

    claimed = await queue.claim_next(worker_id="w-1", lease_seconds=30, now=t0)
    assert claimed is not None and claimed.attempt_count == 1

    # Simulate worker crash: 31 seconds pass without heartbeat
    reclaimed_count = await queue.reclaim_expired_leases(now=t0 + timedelta(seconds=31))
    assert reclaimed_count == 1

    # Re-claim on attempt 2 and fail -> hits max_attempts=2 -> transitions to DEAD_LETTER
    claimed_2 = await queue.claim_next(worker_id="w-2", lease_seconds=30, now=t0 + timedelta(seconds=40))
    assert claimed_2 is not None and claimed_2.attempt_count == 2

    dead = await queue.fail(
        task_id=claimed_2.task_id,
        worker_id="w-2",
        error="Sidecar connection refused",
        now=t0 + timedelta(seconds=45),
    )
    assert dead.status == TaskStatus.DEAD_LETTER
    assert dead.last_error == "Sidecar connection refused"
```

- [ ] **Step 2: Run test to verify it fails**

Run: `uv run pytest backend/tests/platform/harness/test_task_queue_contract.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'backend.platform.harness.domain.task'`

- [ ] **Step 3: Implement `TaskQueuePort`, `InMemoryTaskQueueAdapter`, `PostgresTaskQueueAdapter`, and migration `0001`**

In `backend/platform/harness/adapters/postgres_task_queue.py`, implement `claim_next` using an atomic CTE with `FOR UPDATE SKIP LOCKED`:

```sql
WITH next_task AS (
    SELECT task_id
    FROM core.master_task_queue
    WHERE status = 'QUEUED'
      AND next_run_at <= %(now)s
      AND (
          (%(reserved_for_p0)s = TRUE AND lane = 0)
          OR (
              %(reserved_for_p0)s = FALSE
              AND (
                  lane < 3
                  OR NOT EXISTS (
                      SELECT 1 FROM core.master_task_queue
                      WHERE lane IN (0, 1) AND status = 'QUEUED' AND next_run_at <= %(now)s
                  )
              )
          )
      )
    ORDER BY lane ASC, next_run_at ASC, created_at ASC
    LIMIT 1
    FOR UPDATE SKIP LOCKED
)
UPDATE core.master_task_queue q
SET status = 'RUNNING',
    worker_id = %(worker_id)s,
    leased_until = %(leased_until)s,
    attempt_count = q.attempt_count + 1,
    updated_at = %(now)s
FROM next_task
WHERE q.task_id = next_task.task_id
RETURNING q.*;
```

- [ ] **Step 4: Run test to verify it passes**

Run: `uv run pytest backend/tests/platform/harness/test_task_queue_contract.py -v && uv run mypy --strict backend/platform`
Expected: PASS (all 3 contract tests pass)

- [ ] **Step 5: Commit**

```bash
git add backend/migrations/versions/0001_core_platform_schema.py backend/platform/harness/domain/task.py backend/platform/harness/ports/task_queue_port.py backend/platform/harness/adapters/in_memory_task_queue.py backend/platform/harness/adapters/postgres_task_queue.py backend/tests/platform/harness/test_task_queue_contract.py
git commit -m "feat(harness): implement 4-lane priority TaskQueuePort with reserved P0 slot and SKIP LOCKED"
```

---

### Task 3: Transactional Outbox (`OutboxPort` + `OutboxRelayWorker` with Idempotent Delivery & Dead-Lettering)

Whenever a file lands in an engagement's `raw/` folder, a subagent persists new facts to `okf-wiki/`, or a workflow completes, Chotu needs to trigger downstream observer hooks on the `P2 outbox_hooks` lane. If the code writes to the database first and then tries to enqueue a task in a separate network call, any crash or timeout between those two lines causes silent data drift—facts exist on disk, but `pgvector` never indexes them.

This task builds `OutboxPort` and `OutboxRelayWorker` (`backend/platform/harness/`). Domain operations stage `OutboxEvent` records inside the same unit of work, and `OutboxRelayWorker.drain_batch()` claims pending outbox events, converts them into idempotent `TaskQueuePort.enqueue()` calls on the target lane (`P2_OUTBOX_HOOKS` by default), and marks them `PUBLISHED`—or backs off and transitions them to `DEAD_LETTER` if a downstream handler fails repeatedly.

**Files:**
- Create: `backend/platform/harness/domain/outbox.py`
- Create: `backend/platform/harness/ports/outbox_port.py`
- Create: `backend/platform/harness/adapters/in_memory_outbox.py`
- Create: `backend/platform/harness/adapters/postgres_outbox.py`
- Create: `backend/platform/harness/relay.py`
- Test: `backend/tests/platform/harness/test_outbox_contract.py`

**Interfaces:**
- Consumes: `TaskLane` from `backend.platform.harness.domain.lanes`, `TaskQueuePort` from `backend.platform.harness.ports.task_queue_port`, `WorkflowOrigin` from `backend.platform.orchestration.domain.taxonomy`.
- Produces:
  - `OutboxStatus(StrEnum)`: `PENDING = "PENDING"`, `PUBLISHED = "PUBLISHED"`, `DEAD_LETTER = "DEAD_LETTER"`.
  - `OutboxEvent(BaseModel)`: `event_id: UUID`, `event_type: str`, `aggregate_id: str`, `payload: dict[str, object]`, `idempotency_key: str`, `target_lane: TaskLane = TaskLane.P2_OUTBOX_HOOKS`, `origin: WorkflowOrigin = WorkflowOrigin.USER`, `hop_depth: int = 0`, `status: OutboxStatus = OutboxStatus.PENDING`, `attempt_count: int = 0`, `max_attempts: int = 5`, `next_attempt_at: datetime`, `last_error: str | None = None`.
  - `OutboxPort(ABC)`:
    - `async def stage(self, *, event_type: str, aggregate_id: str, payload: dict[str, object], idempotency_key: str, target_lane: TaskLane = TaskLane.P2_OUTBOX_HOOKS, origin: WorkflowOrigin = WorkflowOrigin.USER, hop_depth: int = 0) -> OutboxEvent`
    - `async def claim_pending_batch(self, *, batch_size: int = 25, now: datetime | None = None) -> list[OutboxEvent]`
    - `async def mark_published(self, *, event_id: UUID) -> OutboxEvent`
    - `async def mark_failed(self, *, event_id: UUID, error: str, base_backoff_seconds: int = 5, now: datetime | None = None) -> OutboxEvent`
  - `OutboxRelayWorker`:
    - `def __init__(self, outbox: OutboxPort, task_queue: TaskQueuePort) -> None`
    - `async def drain_batch(self, *, batch_size: int = 25, now: datetime | None = None) -> int`

- [ ] **Step 1: Write the failing test**

Create `backend/tests/platform/harness/test_outbox_contract.py`:

```python
import pytest

from backend.platform.harness.domain.lanes import TaskLane
from backend.platform.harness.domain.outbox import OutboxStatus
from backend.platform.harness.adapters.in_memory_outbox import InMemoryOutboxAdapter
from backend.platform.harness.adapters.in_memory_task_queue import InMemoryTaskQueueAdapter
from backend.platform.harness.relay import OutboxRelayWorker
from backend.platform.orchestration.domain.taxonomy import WorkflowOrigin


@pytest.mark.asyncio
async def test_outbox_stages_idempotently_and_relays_to_p2_task_queue() -> None:
    outbox = InMemoryOutboxAdapter()
    queue = InMemoryTaskQueueAdapter()
    relay = OutboxRelayWorker(outbox=outbox, task_queue=queue)

    ev1 = await outbox.stage(
        event_type="on_facts_persisted",
        aggregate_id="eng_acme-cloud/concepts/sso.md",
        payload={"sha256": "abc1234"},
        idempotency_key="outbox:sso:abc1234",
        origin=WorkflowOrigin.USER,
        hop_depth=0,
    )
    ev1_dup = await outbox.stage(
        event_type="on_facts_persisted",
        aggregate_id="eng_acme-cloud/concepts/sso.md",
        payload={"sha256": "abc1234"},
        idempotency_key="outbox:sso:abc1234",
    )
    assert ev1.event_id == ev1_dup.event_id

    relayed_count = await relay.drain_batch(batch_size=10)
    assert relayed_count == 1

    # Verify task arrived in P2_OUTBOX_HOOKS lane with origin and hop_depth preserved
    claimed = await queue.claim_next(worker_id="w-p2")
    assert claimed is not None
    assert claimed.lane == TaskLane.P2_OUTBOX_HOOKS
    assert claimed.task_type == "on_facts_persisted"
    assert claimed.payload["origin"] == "USER"
    assert claimed.payload["hop_depth"] == 0

    # Draining a second time is a no-op because event is PUBLISHED
    assert await relay.drain_batch(batch_size=10) == 0


@pytest.mark.asyncio
async def test_outbox_exponential_backoff_and_dead_letter_on_repeated_failure() -> None:
    outbox = InMemoryOutboxAdapter()
    ev = await outbox.stage(
        event_type="on_workflow_complete",
        aggregate_id="run-99",
        payload={},
        idempotency_key="outbox:run-99",
    )
    for attempt in range(1, 5):
        failed = await outbox.mark_failed(event_id=ev.event_id, error=f"timeout {attempt}")
        assert failed.status == OutboxStatus.PENDING
        assert failed.attempt_count == attempt

    dead = await outbox.mark_failed(event_id=ev.event_id, error="timeout 5")
    assert dead.status == OutboxStatus.DEAD_LETTER
    assert dead.attempt_count == 5
```

- [ ] **Step 2: Run test to verify it fails**

Run: `uv run pytest backend/tests/platform/harness/test_outbox_contract.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'backend.platform.harness.domain.outbox'`

- [ ] **Step 3: Implement `OutboxPort`, `InMemoryOutboxAdapter`, `PostgresOutboxAdapter`, and `OutboxRelayWorker`**

In `OutboxRelayWorker.drain_batch`, claim up to `batch_size` pending events, inject `"origin": event.origin.value` and `"hop_depth": event.hop_depth` into the task payload, call `task_queue.enqueue(lane=event.target_lane, task_type=event.event_type, payload=..., idempotency_key=event.idempotency_key)`, and call `outbox.mark_published(event_id=event.event_id)`. Catch any exception from `task_queue.enqueue` and call `outbox.mark_failed(event_id=event.event_id, error=str(exc))`.

- [ ] **Step 4: Run test to verify it passes**

Run: `uv run pytest backend/tests/platform/harness/test_outbox_contract.py -v && uv run mypy --strict backend/platform`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add backend/platform/harness/domain/outbox.py backend/platform/harness/ports/outbox_port.py backend/platform/harness/adapters/in_memory_outbox.py backend/platform/harness/adapters/postgres_outbox.py backend/platform/harness/relay.py backend/tests/platform/harness/test_outbox_contract.py
git commit -m "feat(harness): add Transactional OutboxPort and OutboxRelayWorker with dead-letter protection"
```

---

### Task 4: Per-Node Durable DAG State Machine (`WorkflowStorePort` + `WorkflowRun`, `WorkflowNodeRun` & Immutable `NodeOutput`)

In our retrospectives, we saw why `dsa-prj`'s workflow engine (`current_step: int` + mutable `run.state: dict`) breaks as soon as Chotu runs parallel BAITS subagents (`business_architect` $\parallel$ `security_architect`): concurrent completions overwrite each other's dictionary keys, and a single approval gate freezes the entire workflow instead of just its branch.

This task implements the Per-Node Durable DAG State Machine in `backend/platform/orchestration/`. Each `WorkflowNodeRun` tracks its own parent dependencies (`depends_on_node_ids`) and transitions atomically (`PENDING -> READY -> QUEUED -> RUNNING -> AWAITING_APPROVAL -> COMPLETED | FAILED | CANCELLED`). When a node finishes, it writes an immutable `NodeOutput` record (`output_payload` + `artifact_refs`). In the same atomic operation, `complete_node` inspects all downstream nodes in that `WorkflowRun`: any node still in `PENDING` whose `depends_on_node_ids` are now all `COMPLETED` is transitioned to `READY` and returned to the caller so it can be enqueued once—even if two parallel parent branches finish concurrently.

**Files:**
- Create: `backend/platform/orchestration/domain/workflow.py`
- Create: `backend/platform/orchestration/ports/workflow_store_port.py`
- Create: `backend/platform/orchestration/adapters/in_memory_workflow_store.py`
- Create: `backend/platform/orchestration/adapters/postgres_workflow_store.py`
- Test: `backend/tests/platform/orchestration/test_workflow_dag_state_machine.py`

**Interfaces:**
- Consumes: `CanonicalUri` from `backend.platform.multiagent.domain.uris`, `WorkflowOrigin` from `backend.platform.orchestration.domain.taxonomy`.
- Produces:
  - `NodeRunStatus(StrEnum)`: `PENDING = "PENDING"`, `READY = "READY"`, `QUEUED = "QUEUED"`, `RUNNING = "RUNNING"`, `AWAITING_APPROVAL = "AWAITING_APPROVAL"`, `COMPLETED = "COMPLETED"`, `FAILED = "FAILED"`, `CANCELLED = "CANCELLED"`.
  - `WorkflowRunStatus(StrEnum)`: `RUNNING = "RUNNING"`, `AWAITING_APPROVAL = "AWAITING_APPROVAL"`, `COMPLETED = "COMPLETED"`, `FAILED = "FAILED"`, `CANCELLED = "CANCELLED"`.
  - `WorkflowPlanStatus(StrEnum)`: `CANDIDATE = "CANDIDATE"`, `ESTABLISHED = "ESTABLISHED"`, `DISABLED = "DISABLED"`.
  - `ArtifactRef(BaseModel, frozen=True)`: `relative_path: str`, `sha256: str`, `media_type: str = "text/markdown"`.
  - `NodeOutput(BaseModel, frozen=True)`: `run_id: UUID`, `node_id: str`, `agent_uri: CanonicalUri`, `output_payload: dict[str, object]`, `artifact_refs: tuple[ArtifactRef, ...] = ()`, `citations: tuple[str, ...] = ()`, `completed_at: datetime`.
  - `WorkflowNodeSpec(BaseModel, frozen=True)`: `node_id: str`, `agent_uri: CanonicalUri`, `depends_on_node_ids: tuple[str, ...] = ()`, `requires_approval: bool = False`, `input_mapping: dict[str, str] = {}`.
  - `WorkflowPlan(BaseModel)`: `plan_id: str`, `signature_key: str`, `nodes: tuple[WorkflowNodeSpec, ...]`, `status: WorkflowPlanStatus = WorkflowPlanStatus.CANDIDATE`, `verified_success_count: int = 0`.
  - `WorkflowNodeRun(BaseModel)`: `run_id: UUID`, `node_id: str`, `agent_uri: CanonicalUri`, `depends_on_node_ids: tuple[str, ...]`, `status: NodeRunStatus`, `requires_approval: bool = False`, `error: str | None = None`.
  - `WorkflowRun(BaseModel)`: `run_id: UUID`, `plan_id: str`, `engagement_slug: str | None`, `origin: WorkflowOrigin`, `hop_depth: int`, `status: WorkflowRunStatus`.
  - `NodeCompletionResult(BaseModel, frozen=True)`: `completed_node: WorkflowNodeRun`, `output: NodeOutput`, `newly_ready_nodes: tuple[WorkflowNodeRun, ...]`, `workflow_run: WorkflowRun`.
  - `WorkflowStorePort(ABC)`:
    - `async def create_run(self, *, plan: WorkflowPlan, engagement_slug: str | None, origin: WorkflowOrigin = WorkflowOrigin.USER, hop_depth: int = 0) -> tuple[WorkflowRun, list[WorkflowNodeRun]]` (root nodes with empty `depends_on_node_ids` start in `READY`; dependent nodes start in `PENDING`).
    - `async def transition_node(self, *, run_id: UUID, node_id: str, expected_from: NodeRunStatus, to_status: NodeRunStatus, error: str | None = None) -> WorkflowNodeRun`
    - `async def complete_node(self, *, run_id: UUID, node_id: str, output_payload: dict[str, object], artifact_refs: tuple[ArtifactRef, ...] = (), citations: tuple[str, ...] = ()) -> NodeCompletionResult`
    - `async def get_upstream_outputs(self, *, run_id: UUID, node_id: str) -> dict[str, NodeOutput]`
    - `async def splice_nodes(self, *, run_id: UUID, new_nodes: list[WorkflowNodeSpec], rewire_downstream_node_id: str, additional_dependency_ids: list[str]) -> list[WorkflowNodeRun]`

- [ ] **Step 1: Write the failing test**

Create `backend/tests/platform/orchestration/test_workflow_dag_state_machine.py` (including Review Focus #2: `test_concurrent_parallel_branches_fan_in_without_race_or_double_enqueue`):

```python
import asyncio
import pytest

from backend.platform.multiagent.domain.uris import CanonicalUri
from backend.platform.orchestration.domain.taxonomy import WorkflowOrigin
from backend.platform.orchestration.domain.workflow import (
    ArtifactRef,
    NodeRunStatus,
    WorkflowNodeSpec,
    WorkflowPlan,
    WorkflowRunStatus,
)
from backend.platform.orchestration.adapters.in_memory_workflow_store import InMemoryWorkflowStoreAdapter


def _build_fanout_fanin_plan() -> WorkflowPlan:
    return WorkflowPlan(
        plan_id="plan-baits-review",
        signature_key="ingest_and_synthesize:work_engagement:business,security",
        nodes=(
            WorkflowNodeSpec(
                node_id="extract",
                agent_uri=CanonicalUri.parse("chotu://agent/ingestion/csa-extract@1.0.0"),
            ),
            WorkflowNodeSpec(
                node_id="business_arch",
                agent_uri=CanonicalUri.parse("chotu://agent/architects/business@1.0.0"),
                depends_on_node_ids=("extract",),
            ),
            WorkflowNodeSpec(
                node_id="security_arch",
                agent_uri=CanonicalUri.parse("chotu://agent/architects/security@1.0.0"),
                depends_on_node_ids=("extract",),
                requires_approval=True,
            ),
            WorkflowNodeSpec(
                node_id="reviewer",
                agent_uri=CanonicalUri.parse("chotu://agent/architects/reviewer@1.0.0"),
                depends_on_node_ids=("business_arch", "security_arch"),
            ),
        ),
    )


@pytest.mark.asyncio
async def test_branch_approval_pause_does_not_block_sibling_branch() -> None:
    store = InMemoryWorkflowStoreAdapter()
    plan = _build_fanout_fanin_plan()
    run, nodes = await store.create_run(plan=plan, engagement_slug="acme-cloud", origin=WorkflowOrigin.USER)

    by_id = {n.node_id: n for n in nodes}
    assert by_id["extract"].status == NodeRunStatus.READY
    assert by_id["business_arch"].status == NodeRunStatus.PENDING
    assert by_id["security_arch"].status == NodeRunStatus.PENDING
    assert by_id["reviewer"].status == NodeRunStatus.PENDING

    # Run and complete root 'extract' node
    await store.transition_node(
        run_id=run.run_id, node_id="extract", expected_from=NodeRunStatus.READY, to_status=NodeRunStatus.QUEUED
    )
    await store.transition_node(
        run_id=run.run_id, node_id="extract", expected_from=NodeRunStatus.QUEUED, to_status=NodeRunStatus.RUNNING
    )
    res_extract = await store.complete_node(
        run_id=run.run_id,
        node_id="extract",
        output_payload={"facts_count": 14},
        artifact_refs=(ArtifactRef(relative_path="okf-wiki/concepts/sso.md", sha256="111aaa"),),
    )
    ready_ids = {n.node_id for n in res_extract.newly_ready_nodes}
    assert ready_ids == {"business_arch", "security_arch"}

    # Start both branches; pause security_arch on AWAITING_APPROVAL while business_arch completes
    for nid in ("business_arch", "security_arch"):
        await store.transition_node(
            run_id=run.run_id, node_id=nid, expected_from=NodeRunStatus.READY, to_status=NodeRunStatus.QUEUED
        )
        await store.transition_node(
            run_id=run.run_id, node_id=nid, expected_from=NodeRunStatus.QUEUED, to_status=NodeRunStatus.RUNNING
        )

    await store.transition_node(
        run_id=run.run_id,
        node_id="security_arch",
        expected_from=NodeRunStatus.RUNNING,
        to_status=NodeRunStatus.AWAITING_APPROVAL,
    )
    res_biz = await store.complete_node(
        run_id=run.run_id,
        node_id="business_arch",
        output_payload={"bvf_score": 0.91},
        citations=("[[sso#^f-1]]",),
    )
    # Reviewer must NOT be ready yet because security_arch is AWAITING_APPROVAL
    assert res_biz.newly_ready_nodes == ()
    assert res_biz.workflow_run.status == WorkflowRunStatus.AWAITING_APPROVAL

    # Approve and complete security_arch -> reviewer becomes READY and sees both immutable outputs
    await store.transition_node(
        run_id=run.run_id,
        node_id="security_arch",
        expected_from=NodeRunStatus.AWAITING_APPROVAL,
        to_status=NodeRunStatus.QUEUED,
    )
    await store.transition_node(
        run_id=run.run_id,
        node_id="security_arch",
        expected_from=NodeRunStatus.QUEUED,
        to_status=NodeRunStatus.RUNNING,
    )
    res_sec = await store.complete_node(
        run_id=run.run_id,
        node_id="security_arch",
        output_payload={"zero_trust_gaps": 2},
        citations=("[[sso#^f-2]]",),
    )
    assert [n.node_id for n in res_sec.newly_ready_nodes] == ["reviewer"]

    upstream = await store.get_upstream_outputs(run_id=run.run_id, node_id="reviewer")
    assert upstream["business_arch"].output_payload == {"bvf_score": 0.91}
    assert upstream["security_arch"].output_payload == {"zero_trust_gaps": 2}


@pytest.mark.asyncio
async def test_concurrent_parallel_branches_fan_in_without_race_or_double_enqueue() -> None:
    store = InMemoryWorkflowStoreAdapter()
    plan = _build_fanout_fanin_plan()
    run, _ = await store.create_run(plan=plan, engagement_slug="acme-cloud")

    await store.transition_node(
        run_id=run.run_id, node_id="extract", expected_from=NodeRunStatus.READY, to_status=NodeRunStatus.QUEUED
    )
    await store.transition_node(
        run_id=run.run_id, node_id="extract", expected_from=NodeRunStatus.QUEUED, to_status=NodeRunStatus.RUNNING
    )
    await store.complete_node(run_id=run.run_id, node_id="extract", output_payload={"ok": True})

    for nid in ("business_arch", "security_arch"):
        await store.transition_node(
            run_id=run.run_id, node_id=nid, expected_from=NodeRunStatus.READY, to_status=NodeRunStatus.QUEUED
        )
        await store.transition_node(
            run_id=run.run_id, node_id=nid, expected_from=NodeRunStatus.QUEUED, to_status=NodeRunStatus.RUNNING
        )

    # Complete both parallel branches concurrently via asyncio.gather
    r1, r2 = await asyncio.gather(
        store.complete_node(run_id=run.run_id, node_id="business_arch", output_payload={"branch": "biz"}),
        store.complete_node(run_id=run.run_id, node_id="security_arch", output_payload={"branch": "sec"}),
    )

    # Exactly ONE of the two completion calls may report 'reviewer' in newly_ready_nodes
    combined_ready = list(r1.newly_ready_nodes) + list(r2.newly_ready_nodes)
    assert len(combined_ready) == 1
    assert combined_ready[0].node_id == "reviewer"
```

- [ ] **Step 2: Run test to verify it fails**

Run: `uv run pytest backend/tests/platform/orchestration/test_workflow_dag_state_machine.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'backend.platform.orchestration.domain.workflow'`

- [ ] **Step 3: Implement `WorkflowStorePort`, `InMemoryWorkflowStoreAdapter`, and `PostgresWorkflowStoreAdapter`**

Validate DAG acyclicity on `WorkflowPlan` creation (raising `CyclicWorkflowGraphError` if a cycle exists). Use an async per-run lock in `InMemoryWorkflowStoreAdapter` and `SELECT ... FOR UPDATE` on `core.workflow_runs` in `PostgresWorkflowStoreAdapter` inside `complete_node` so concurrent sibling branch completions serialize their downstream readiness check and transition `reviewer` from `PENDING` to `READY` exactly once.

- [ ] **Step 4: Run test to verify it passes**

Run: `uv run pytest backend/tests/platform/orchestration/test_workflow_dag_state_machine.py -v && uv run mypy --strict backend/platform`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add backend/platform/orchestration/domain/workflow.py backend/platform/orchestration/ports/workflow_store_port.py backend/platform/orchestration/adapters/in_memory_workflow_store.py backend/platform/orchestration/adapters/postgres_workflow_store.py backend/tests/platform/orchestration/test_workflow_dag_state_machine.py
git commit -m "feat(orchestration): implement Per-Node Durable DAG State Machine and immutable NodeOutput store"
```

---

### Task 5: Runtime Service Discovery & Service Locator (`ServiceDiscoveryPort`, `ServiceLocatorPort`, 3-Stage Resolution & Lifecycle Fallback)

Chotu Corp grows continuously: shipped capabilities live in `backend/inventory/`, runtime-grown tools and skills live in `~/.chotu/`, and external apps (`dsapps-journal`, `dsapps-checkin`) or sidecars (`docling`) announce themselves over HTTP. Stuffing that entire catalog into an LLM prompt on every turn is slow and expensive.

This task implements `ServiceLocatorPort` and `ServiceDiscoveryPort` (`backend/platform/multiagent/`). It supports Hybrid Pull (manifest directory scan with fail-loud `DISABLED` marking for invalid schemas) + Push registration, the **3-Stage Zero-LLM Resolution Pipeline** (Stage 1: $O(1)$ exact URI or trigger phrase lookup $\rightarrow$ Stage 2: Health/LazyStart + Allowlist + Department/BAITS Pillar + `applies_to_kinds` filter $\rightarrow$ Stage 3: Hybrid lexical + embedding cosine + historical reliability score ranking, logging to `CapabilityMiss` if below threshold), plus lifecycle-aware activation (`always_on`, `lazy_start` wakeup callback, and circuit-breaker failover to `fallback_uri`).

**Files:**
- Create: `backend/platform/multiagent/domain/discovery.py`
- Create: `backend/platform/multiagent/ports/service_discovery_port.py`
- Create: `backend/platform/multiagent/adapters/capability_registry.py`
- Test: `backend/tests/platform/multiagent/test_service_discovery_and_locator.py`

**Interfaces:**
- Consumes: `CanonicalUri`, `CapabilityKind`, `CapabilityDescriptor`, `HealthStatus`, `LifecyclePolicy` from `backend.platform.multiagent.domain.*`; `BaitsPillar`, `DepartmentKind` from `backend.platform.orchestration.domain.taxonomy`.
- Produces:
  - `DiscoveryQuery(BaseModel, frozen=True)`: `intent_text: str`, `kind: CapabilityKind | None = None`, `department: DepartmentKind | None = None`, `baits_pillar: BaitsPillar | None = None`, `applies_to_kind: str | None = None`, `caller_allowlist: tuple[str, ...] | None = None`, `min_score_threshold: float = 0.45`, `limit: int = 5`.
  - `ScoredCapability(BaseModel, frozen=True)`: `descriptor: CapabilityDescriptor`, `score: float`, `reliability_rate: float`.
  - `CapabilityMiss(BaseModel, frozen=True)`: `miss_id: UUID`, `query: DiscoveryQuery`, `best_score: float`, `recorded_at: datetime`.
  - `ResolvedCapability(BaseModel, frozen=True)`: `requested_uri: CanonicalUri`, `effective_descriptor: CapabilityDescriptor`, `used_fallback: bool`, `woke_from_sleep: bool`.
  - `ServiceLocatorPort(ABC)`:
    - `async def locate(self, uri_or_trigger: str) -> CapabilityDescriptor | None`
    - `async def resolve_for_invocation(self, uri: CanonicalUri) -> ResolvedCapability` (handles `lazy_start` wakeup hook and circuit-breaker fallback to `fallback_uri` when primary is `UNHEALTHY` or circuit is open).
    - `async def record_invocation_outcome(self, uri: CanonicalUri, *, succeeded: bool) -> None` (updates historical reliability stats and trips circuit breaker to `UNHEALTHY` after 3 consecutive failures).
  - `ServiceDiscoveryPort(ABC)`:
    - `async def register(self, descriptor: CapabilityDescriptor) -> CapabilityDescriptor`
    - `async def reconcile_manifest_dict(self, raw_manifest: dict[str, object]) -> CapabilityDescriptor` (Fail-Loud Reconciliation: if validation fails on a capability fields check, registers with `health_status=HealthStatus.DISABLED` and `disabled_reason` populated rather than crashing startup).
    - `async def discover(self, query: DiscoveryQuery) -> list[ScoredCapability]`
    - `async def list_capability_misses(self) -> list[CapabilityMiss]`

- [ ] **Step 1: Write the failing test**

Create `backend/tests/platform/multiagent/test_service_discovery_and_locator.py`:

```python
import pytest

from backend.platform.multiagent.domain.uris import CanonicalUri, CapabilityKind
from backend.platform.multiagent.domain.capability import (
    CapabilityDescriptor,
    HealthStatus,
    LifecyclePolicy,
)
from backend.platform.multiagent.domain.discovery import DiscoveryQuery
from backend.platform.multiagent.adapters.capability_registry import InMemoryCapabilityRegistry
from backend.platform.orchestration.domain.taxonomy import BaitsPillar, DepartmentKind, RiskLevel


@pytest.mark.asyncio
async def test_three_stage_resolution_and_capability_miss_ledger() -> None:
    registry = InMemoryCapabilityRegistry()

    raid_skill = CapabilityDescriptor(
        uri=CanonicalUri.parse("chotu://skill/baits/make-raid@1.2.0"),
        description="Build RAID register of risks, assumptions, issues, and dependencies from facts",
        department=DepartmentKind.DELIVERY,
        baits_pillar=BaitsPillar.BUSINESS,
        trigger_phrases=("make raid", "generate raid log"),
        input_schema={"type": "object"},
        output_schema={"type": "object"},
        applies_to_kinds=("transcript", "md"),
        permissions=("workspace:read", "workspace:write"),
        risk_level=RiskLevel.LOW,
        lifecycle_policy=LifecyclePolicy.ALWAYS_ON,
    )
    sec_agent = CapabilityDescriptor(
        uri=CanonicalUri.parse("chotu://agent/architects/security@1.0.0"),
        description="Evaluate zero trust architecture, IAM federation, and encryption controls",
        department=DepartmentKind.DELIVERY,
        baits_pillar=BaitsPillar.SECURITY,
        trigger_phrases=("security review",),
        input_schema={"type": "object"},
        output_schema={"type": "object"},
        applies_to_kinds=("transcript", "pdf"),
        permissions=("workspace:read",),
        risk_level=RiskLevel.LOW,
        lifecycle_policy=LifecyclePolicy.ALWAYS_ON,
    )
    await registry.register(raid_skill)
    await registry.register(sec_agent)

    # Stage 1: O(1) exact URI and trigger phrase location
    located_by_uri = await registry.locate("chotu://skill/baits/make-raid@1.2.0")
    located_by_trigger = await registry.locate("make raid")
    assert located_by_uri == raid_skill
    assert located_by_trigger == raid_skill

    # Stage 2 & 3: Faceted filter + hybrid ranking
    hits = await registry.discover(
        DiscoveryQuery(
            intent_text="zero trust IAM federation controls",
            kind=CapabilityKind.AGENT,
            baits_pillar=BaitsPillar.SECURITY,
            applies_to_kind="pdf",
        )
    )
    assert len(hits) == 1
    assert hits[0].descriptor.uri == sec_agent.uri

    # Below threshold -> records to Capability-Miss Ledger
    misses_result = await registry.discover(
        DiscoveryQuery(
            intent_text="compile solidity smart contract bytecode",
            min_score_threshold=0.75,
        )
    )
    assert misses_result == []
    misses = await registry.list_capability_misses()
    assert len(misses) == 1
    assert "solidity" in misses[0].query.intent_text


@pytest.mark.asyncio
async def test_lazy_start_wakeup_circuit_breaker_and_fallback_uri() -> None:
    woken_uris: list[str] = []

    async def fake_starter(desc: CapabilityDescriptor) -> bool:
        woken_uris.append(str(desc.uri))
        return True

    registry = InMemoryCapabilityRegistry(lazy_starter=fake_starter)

    local_pdf = CapabilityDescriptor(
        uri=CanonicalUri.parse("chotu://tool/extractors/parse-pdf-local@1.0.0"),
        description="Fallback local Python PDF parser",
        department=DepartmentKind.KNOWLEDGE,
        baits_pillar=BaitsPillar.INFORMATION,
        trigger_phrases=(),
        input_schema={"type": "object"},
        output_schema={"type": "object"},
        applies_to_kinds=("pdf",),
        permissions=("workspace:read",),
        risk_level=RiskLevel.LOW,
        lifecycle_policy=LifecyclePolicy.ALWAYS_ON,
    )
    docling_sidecar = CapabilityDescriptor(
        uri=CanonicalUri.parse("chotu://sidecar/ingestion/docling@1.0.0"),
        description="Docling container sidecar",
        department=DepartmentKind.KNOWLEDGE,
        baits_pillar=BaitsPillar.INFORMATION,
        trigger_phrases=(),
        input_schema={"type": "object"},
        output_schema={"type": "object"},
        applies_to_kinds=("pdf",),
        permissions=("workspace:read",),
        risk_level=RiskLevel.LOW,
        lifecycle_policy=LifecyclePolicy.LAZY_START,
        fallback_uri=local_pdf.uri,
        health_status=HealthStatus.SLEEPING,
    )
    await registry.register(local_pdf)
    await registry.register(docling_sidecar)

    # 1. Resolving a SLEEPING lazy_start capability wakes it on demand
    res_wake = await registry.resolve_for_invocation(docling_sidecar.uri)
    assert res_wake.woke_from_sleep is True
    assert res_wake.used_fallback is False
    assert res_wake.effective_descriptor.uri == docling_sidecar.uri
    assert woken_uris == ["chotu://sidecar/ingestion/docling@1.0.0"]

    # 2. 3 consecutive failures trip the circuit breaker -> routes to fallback_uri
    for _ in range(3):
        await registry.record_invocation_outcome(docling_sidecar.uri, succeeded=False)

    res_fallback = await registry.resolve_for_invocation(docling_sidecar.uri)
    assert res_fallback.used_fallback is True
    assert res_fallback.effective_descriptor.uri == local_pdf.uri


@pytest.mark.asyncio
async def test_fail_loud_reconciliation_marks_broken_manifest_disabled() -> None:
    registry = InMemoryCapabilityRegistry()
    broken_manifest: dict[str, object] = {
        "uri": "chotu://tool/custom/broken-tool@1.0.0",
        "description": "Missing required input_schema and invalid department",
        "department": "not_a_real_department",
    }
    reconciled = await registry.reconcile_manifest_dict(broken_manifest)
    assert reconciled.health_status == HealthStatus.DISABLED
    assert reconciled.disabled_reason is not None
```

- [ ] **Step 2: Run test to verify it fails**

Run: `uv run pytest backend/tests/platform/multiagent/test_service_discovery_and_locator.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'backend.platform.multiagent.domain.discovery'`

- [ ] **Step 3: Implement `ServiceDiscoveryPort`, `ServiceLocatorPort`, and `InMemoryCapabilityRegistry`**

Implement Stage 1 hash lookup (`_by_uri`, `_by_unversioned`, `_by_trigger`), Stage 2 structural & allowlist filtering (matching `caller_allowlist` wildcards like `"chotu://tool/core/*"`), Stage 3 hybrid token overlap + historical reliability scoring (`0.75 * lexical_cosine + 0.25 * reliability_rate`), and the 3-strike circuit breaker + `fallback_uri` resolution in `resolve_for_invocation`.

- [ ] **Step 4: Run test to verify it passes**

Run: `uv run pytest backend/tests/platform/multiagent/test_service_discovery_and_locator.py -v && uv run mypy --strict backend/platform`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add backend/platform/multiagent/domain/discovery.py backend/platform/multiagent/ports/service_discovery_port.py backend/platform/multiagent/adapters/capability_registry.py backend/tests/platform/multiagent/test_service_discovery_and_locator.py
git commit -m "feat(multiagent): implement 3-Stage Service Discovery, Service Locator, lazy-start, and fallback chains"
```

---

### Task 6: `EngagementWorkspaceJail` (`platform/environment/`) & Dual-Mode `HookRegistryPort` (`platform/harness/`)

This task locks down two critical safety mechanisms before we wire up the runtime chokepoints:
1. **`EngagementWorkspaceJail` (`backend/platform/environment/`)**: When a subagent runs inside an engagement (`workspaces/acme-cloud/`), it must never be able to read or write another client's workspace (`workspaces/globex-corp/`) or bypass the Knowledge Librarian to touch `CHOTU_VAULT_PATH` via `..` traversal, absolute paths, or symlinks pointing outside the engagement root. Only custodial knowledge capabilities explicitly granted `vault:custodian` permission may resolve paths inside `CHOTU_VAULT_PATH`.
2. **Dual-Mode `HookRegistryPort` (`backend/platform/harness/`)**: Runs synchronous **Guard/Mutator hooks** (`pre_subagent_dispatch`, `post_subagent_dispatch`, `pre_tool_invoke`) that can enrich envelopes or block execution on citation/scope violations, and async **Observer hooks** (`on_facts_persisted`, `on_workflow_complete`, `on_capability_miss`) enqueued via `OutboxPort`. Crucially, `HookRegistryPort` enforces the **Origin-Tagged Recursion Fence**: observer hooks can only spawn follow-up workflows when `origin == WorkflowOrigin.USER` and `hop_depth == 0`.

**Files:**
- Create: `backend/platform/environment/domain/jail.py`
- Create: `backend/platform/environment/ports/workspace_jail_port.py`
- Create: `backend/platform/environment/adapters/filesystem_jail.py`
- Create: `backend/platform/harness/domain/hooks.py`
- Create: `backend/platform/harness/ports/hook_registry_port.py`
- Create: `backend/platform/harness/adapters/hook_registry.py`
- Test: `backend/tests/platform/test_workspace_jail_and_hooks.py`

**Interfaces:**
- Consumes: `OutboxPort` from `backend.platform.harness.ports.outbox_port`, `WorkflowOrigin`, `ScopeKind` from `backend.platform.orchestration.domain.taxonomy`.
- Produces:
  - `WorkspaceEscapeError(PermissionError)`: raised when a path or command escapes the active jail.
  - `EngagementWorkspaceJail`:
    - `def __init__(self, *, workspaces_root: Path, engagement_slug: str, vault_root: Path | None = None, allow_vault_custodian: bool = False) -> None`
    - `def resolve_safe_path(self, candidate: str | Path) -> Path` (canonicalizes via `Path.resolve()`, rejects relative traversal `..`, absolute paths outside `workspaces_root / engagement_slug`, and symlinks whose target resolves outside the allowed root).
    - `def sanitized_subprocess_env(self, base_env: dict[str, str] | None = None) -> dict[str, str]` (strips secret keys like `DATABASE_URL`, `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, `CHOTU_VAULT_PATH` and sets `PWD` to the engagement root).
  - `HookEvent(StrEnum)`: `PRE_CLASSIFY = "pre_classify"`, `POST_CLASSIFY = "post_classify"`, `PRE_SUBAGENT_DISPATCH = "pre_subagent_dispatch"`, `POST_SUBAGENT_DISPATCH = "post_subagent_dispatch"`, `PRE_TOOL_INVOKE = "pre_tool_invoke"`, `POST_TOOL_INVOKE = "post_tool_invoke"`, `ON_FACTS_PERSISTED = "on_facts_persisted"`, `ON_CAPABILITY_MISS = "on_capability_miss"`, `ON_WORKFLOW_COMPLETE = "on_workflow_complete"`.
  - `HookContext(BaseModel)`: `event: HookEvent`, `run_id: UUID | None = None`, `engagement_slug: str | None = None`, `origin: WorkflowOrigin = WorkflowOrigin.USER`, `hop_depth: int = 0`, `payload: dict[str, object]`.
  - `HookResult(BaseModel, frozen=True)`: `allowed: bool = True`, `mutated_payload: dict[str, object]`, `violation_reason: str | None = None`, `spawned_workflow_intents: tuple[str, ...] = ()`.
  - `HookRegistryPort(ABC)`:
    - `def register_interceptor(self, event: HookEvent, handler: Callable[[HookContext], Awaitable[HookResult]]) -> None`
    - `def register_observer(self, event: HookEvent, handler: Callable[[HookContext], Awaitable[list[str]]]) -> None` (handler returns any follow-up workflow intents it wants to spawn).
    - `async def run_interceptors(self, context: HookContext) -> HookResult`
    - `async def emit_observer_event(self, context: HookContext, *, idempotency_key: str) -> list[str]` (stages event to `OutboxPort` and executes observer handlers, enforcing the `origin == WorkflowOrigin.USER and hop_depth == 0` fence on any spawned follow-up workflow intents).

- [ ] **Step 1: Write the failing test**

Create `backend/tests/platform/test_workspace_jail_and_hooks.py` (including Review Focus #3 and Review Focus #4):

```python
from pathlib import Path
from uuid import uuid4
import pytest

from backend.platform.environment.domain.jail import WorkspaceEscapeError
from backend.platform.environment.adapters.filesystem_jail import EngagementWorkspaceJail
from backend.platform.harness.adapters.in_memory_outbox import InMemoryOutboxAdapter
from backend.platform.harness.domain.hooks import HookContext, HookEvent, HookResult
from backend.platform.harness.adapters.hook_registry import DefaultHookRegistryAdapter
from backend.platform.orchestration.domain.taxonomy import WorkflowOrigin


def test_workspace_jail_blocks_dotdot_and_symlink_escape_to_sibling_or_vault(tmp_path: Path) -> None:
    workspaces_root = tmp_path / "workspaces"
    acme_dir = workspaces_root / "acme-cloud"
    globex_dir = workspaces_root / "globex-corp"
    vault_dir = tmp_path / "universal-vault"
    acme_dir.mkdir(parents=True)
    globex_dir.mkdir(parents=True)
    vault_dir.mkdir(parents=True)

    (acme_dir / "notes.md").write_text("Acme client notes", encoding="utf-8")
    secret_file = globex_dir / "secret.md"
    secret_file.write_text("Globex confidential", encoding="utf-8")
    vault_file = vault_dir / "pattern.md"
    vault_file.write_text("Universal pattern", encoding="utf-8")

    # Create a sneaky symlink inside acme-cloud pointing to globex-corp/secret.md
    escape_symlink = acme_dir / "sneaky_link.md"
    escape_symlink.symlink_to(secret_file)

    jail = EngagementWorkspaceJail(
        workspaces_root=workspaces_root,
        engagement_slug="acme-cloud",
        vault_root=vault_dir,
        allow_vault_custodian=False,
    )

    # Valid path inside acme-cloud succeeds
    resolved = jail.resolve_safe_path("notes.md")
    assert resolved == (acme_dir / "notes.md").resolve()

    # Relative ../ escape to sibling engagement is blocked
    with pytest.raises(WorkspaceEscapeError):
        jail.resolve_safe_path("../globex-corp/secret.md")

    # Direct absolute path to vault is blocked for non-custodial subagents
    with pytest.raises(WorkspaceEscapeError):
        jail.resolve_safe_path(vault_file)

    # Symlink escape pointing outside acme-cloud is blocked
    with pytest.raises(WorkspaceEscapeError):
        jail.resolve_safe_path("sneaky_link.md")

    # Subprocess environment strips secrets and CHOTU_VAULT_PATH
    clean_env = jail.sanitized_subprocess_env(
        {"PATH": "/usr/bin", "DATABASE_URL": "postgresql://secret", "CHOTU_VAULT_PATH": str(vault_dir)}
    )
    assert "DATABASE_URL" not in clean_env
    assert "CHOTU_VAULT_PATH" not in clean_env
    assert clean_env["PWD"] == str(acme_dir.resolve())


@pytest.mark.asyncio
async def test_interceptor_mutator_and_citation_guard_hooks() -> None:
    outbox = InMemoryOutboxAdapter()
    hooks = DefaultHookRegistryAdapter(outbox=outbox)

    async def inject_context_mutator(ctx: HookContext) -> HookResult:
        updated = dict(ctx.payload)
        updated["context_packet"] = ["[[sso#^f-1]]"]
        return HookResult(allowed=True, mutated_payload=updated)

    async def verify_citation_guard(ctx: HookContext) -> HookResult:
        text = str(ctx.payload.get("synthesis_markdown", ""))
        if "[[" not in text or "#^" not in text:
            return HookResult(
                allowed=False,
                mutated_payload=ctx.payload,
                violation_reason="Missing verbatim [[concept#^fact]] citation anchor",
            )
        return HookResult(allowed=True, mutated_payload=ctx.payload)

    hooks.register_interceptor(HookEvent.PRE_SUBAGENT_DISPATCH, inject_context_mutator)
    hooks.register_interceptor(HookEvent.POST_SUBAGENT_DISPATCH, verify_citation_guard)

    pre_res = await hooks.run_interceptors(
        HookContext(event=HookEvent.PRE_SUBAGENT_DISPATCH, payload={"task": "synthesize"})
    )
    assert pre_res.allowed is True
    assert pre_res.mutated_payload["context_packet"] == ["[[sso#^f-1]]"]

    bad_post = await hooks.run_interceptors(
        HookContext(event=HookEvent.POST_SUBAGENT_DISPATCH, payload={"synthesis_markdown": "Uncited claim"})
    )
    assert bad_post.allowed is False
    assert "citation anchor" in (bad_post.violation_reason or "")


@pytest.mark.asyncio
async def test_observer_hook_recursion_fence_blocks_non_user_or_nonzero_hop() -> None:
    outbox = InMemoryOutboxAdapter()
    hooks = DefaultHookRegistryAdapter(outbox=outbox)

    async def spawn_resynthesis_on_complete(ctx: HookContext) -> list[str]:
        return ["resynthesize_baits_pillars"]

    hooks.register_observer(HookEvent.ON_WORKFLOW_COMPLETE, spawn_resynthesis_on_complete)

    # 1. Top-level USER workflow (origin=USER, hop_depth=0) IS allowed to spawn follow-up workflow
    spawned_user = await hooks.emit_observer_event(
        HookContext(
            event=HookEvent.ON_WORKFLOW_COMPLETE,
            run_id=uuid4(),
            origin=WorkflowOrigin.USER,
            hop_depth=0,
            payload={"status": "COMPLETED"},
        ),
        idempotency_key="wf-complete-user-0",
    )
    assert spawned_user == ["resynthesize_baits_pillars"]

    # 2. Follow-up HOOK workflow (origin=HOOK, hop_depth=1) is BLOCKED from spawning recursive workflows
    spawned_hook = await hooks.emit_observer_event(
        HookContext(
            event=HookEvent.ON_WORKFLOW_COMPLETE,
            run_id=uuid4(),
            origin=WorkflowOrigin.HOOK,
            hop_depth=1,
            payload={"status": "COMPLETED"},
        ),
        idempotency_key="wf-complete-hook-1",
    )
    assert spawned_hook == []

    # 3. SUBCONSCIOUS background workflow is also BLOCKED from recursive hook spawning
    spawned_sub = await hooks.emit_observer_event(
        HookContext(
            event=HookEvent.ON_WORKFLOW_COMPLETE,
            run_id=uuid4(),
            origin=WorkflowOrigin.SUBCONSCIOUS,
            hop_depth=0,
            payload={"status": "COMPLETED"},
        ),
        idempotency_key="wf-complete-sub-0",
    )
    assert spawned_sub == []
```

- [ ] **Step 2: Run test to verify it fails**

Run: `uv run pytest backend/tests/platform/test_workspace_jail_and_hooks.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'backend.platform.environment.domain.jail'`

- [ ] **Step 3: Implement `EngagementWorkspaceJail` and `DefaultHookRegistryAdapter`**

In `EngagementWorkspaceJail.resolve_safe_path`, resolve `allowed_root = (self.workspaces_root / self.engagement_slug).resolve()`. If `candidate` is relative, join it to `allowed_root` before calling `.resolve(strict=False)`. Verify `resolved.is_relative_to(allowed_root)` (or `resolved.is_relative_to(self.vault_root.resolve())` only when `self.allow_vault_custodian is True`). Raise `WorkspaceEscapeError` on any violation. In `DefaultHookRegistryAdapter.emit_observer_event`, stage the event to `self._outbox.stage(...)` and only keep returned follow-up workflow intents when `context.origin == WorkflowOrigin.USER and context.hop_depth == 0`.

- [ ] **Step 4: Run test to verify it passes**

Run: `uv run pytest backend/tests/platform/test_workspace_jail_and_hooks.py -v && uv run mypy --strict backend/platform`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add backend/platform/environment/ backend/platform/harness/domain/hooks.py backend/platform/harness/ports/hook_registry_port.py backend/platform/harness/adapters/hook_registry.py backend/tests/platform/test_workspace_jail_and_hooks.py
git commit -m "feat(environment,harness): add symlink-proof EngagementWorkspaceJail and HookRegistryPort with recursion fence"
```

---

### Task 7: Sovereign Harness Chokepoints (`ToolRegistry.invoke()` & `AgentRuntime.invoke()` with Policy, ApprovalBus, OTel Spans & Subagents-Only Enforcement)

Every single action in Chotu Corp flows through two sovereign in-process execution seams: `AgentRuntime.invoke()` for subagent turns and `ToolRegistry.invoke()` for tool calls. Centralizing execution at these two chokepoints means no subagent can bypass `EngagementWorkspaceJail`, `PolicyEngine`, `ApprovalBus`, `HookRegistryPort`, or OpenTelemetry span emission—and it is where we structurally enforce the **Subagents-Only Invariant** that forbids `lead/chotu` from calling raw tools directly.

This task implements `PolicyEngine`, `ApprovalBus`, `ObservabilityPort` (recording parent-child trace spans `turn -> workflow_node -> subagent.invoke -> tool.invoke`), `ToolRegistry.invoke()`, and `AgentRuntime.invoke()`.

**Files:**
- Create: `backend/platform/orchestration/domain/policy.py`
- Create: `backend/platform/harness/domain/observability.py`
- Create: `backend/platform/harness/ports/observability_port.py`
- Create: `backend/platform/harness/adapters/in_memory_observability.py`
- Create: `backend/platform/tools/domain/invocation.py`
- Create: `backend/platform/tools/ports/tool_registry_port.py`
- Create: `backend/platform/tools/adapters/tool_registry.py`
- Create: `backend/platform/multiagent/domain/execution.py`
- Create: `backend/platform/multiagent/ports/agent_runtime_port.py`
- Create: `backend/platform/multiagent/adapters/agent_runtime.py`
- Test: `backend/tests/platform/test_sovereign_chokepoints.py`

**Interfaces:**
- Consumes: `CanonicalUri`, `CapabilityKind` from `backend.platform.multiagent.domain.uris`, `ServiceLocatorPort` from `backend.platform.multiagent.ports.service_discovery_port`, `EngagementWorkspaceJail` from `backend.platform.environment.adapters.filesystem_jail`, `HookRegistryPort`, `HookEvent`, `HookContext` from `backend.platform.harness.*`.
- Produces:
  - `SubagentsOnlyViolationError(PermissionError)`: raised when `caller_agent_uri.unversioned_key == "chotu://agent/lead/chotu"` invokes `ToolRegistry.invoke()`.
  - `PolicyViolationError(PermissionError)` and `ApprovalRequiredPause(Exception)`.
  - `SpanRecord(BaseModel, frozen=True)`: `span_id: str`, `parent_span_id: str | None`, `name: str`, `attributes: dict[str, str]`.
  - `ToolInvocationContext(BaseModel, arbitrary_types_allowed=True)`: `caller_agent_uri: CanonicalUri`, `run_id: UUID | None = None`, `node_id: str | None = None`, `engagement_slug: str | None = None`, `parent_span_id: str | None = None`, `workspace_jail: EngagementWorkspaceJail | None = None`, `approved_tool_calls: frozenset[str] = frozenset()`.
  - `ToolRegistryPort(ABC)`:
    - `def register_handler(self, tool_uri: CanonicalUri, handler: Callable[[dict[str, object], ToolInvocationContext], Awaitable[dict[str, object]]]) -> None`
    - `async def invoke(self, tool_uri: CanonicalUri | str, args: dict[str, object], context: ToolInvocationContext) -> dict[str, object]`
  - `SubagentEnvelope(BaseModel, arbitrary_types_allowed=True)`: `run_id: UUID`, `node_id: str`, `engagement_slug: str | None`, `payload: dict[str, object]`, `parent_span_id: str | None = None`, `workspace_jail: EngagementWorkspaceJail | None = None`.
  - `AgentRuntimePort(ABC)`:
    - `def register_agent_handler(self, agent_uri: CanonicalUri, handler: Callable[[SubagentEnvelope, str], Awaitable[dict[str, object]]]) -> None`
    - `async def invoke(self, agent_uri: CanonicalUri | str, envelope: SubagentEnvelope) -> dict[str, object]`

- [ ] **Step 1: Write the failing test**

Create `backend/tests/platform/test_sovereign_chokepoints.py` (including Review Focus #5: `test_lead_orchestrator_direct_tool_call_raises_subagents_only_violation`):

```python
from pathlib import Path
from uuid import uuid4
import pytest

from backend.platform.environment.adapters.filesystem_jail import EngagementWorkspaceJail
from backend.platform.environment.domain.jail import WorkspaceEscapeError
from backend.platform.harness.adapters.hook_registry import DefaultHookRegistryAdapter
from backend.platform.harness.adapters.in_memory_observability import InMemoryObservabilityAdapter
from backend.platform.harness.adapters.in_memory_outbox import InMemoryOutboxAdapter
from backend.platform.multiagent.adapters.agent_runtime import SovereignAgentRuntime
from backend.platform.multiagent.adapters.capability_registry import InMemoryCapabilityRegistry
from backend.platform.multiagent.domain.capability import CapabilityDescriptor, LifecyclePolicy
from backend.platform.multiagent.domain.execution import SubagentEnvelope
from backend.platform.multiagent.domain.uris import CanonicalUri
from backend.platform.orchestration.domain.policy import SubagentsOnlyViolationError
from backend.platform.orchestration.domain.taxonomy import BaitsPillar, DepartmentKind, RiskLevel
from backend.platform.tools.adapters.tool_registry import SovereignToolRegistry
from backend.platform.tools.domain.invocation import ToolInvocationContext


@pytest.mark.asyncio
async def test_lead_orchestrator_direct_tool_call_raises_subagents_only_violation(tmp_path: Path) -> None:
    registry = InMemoryCapabilityRegistry()
    outbox = InMemoryOutboxAdapter()
    hooks = DefaultHookRegistryAdapter(outbox=outbox)
    obs = InMemoryObservabilityAdapter()
    tool_registry = SovereignToolRegistry(locator=registry, hooks=hooks, observability=obs)

    grep_uri = CanonicalUri.parse("chotu://tool/core/grep@1.0.0")
    await registry.register(
        CapabilityDescriptor(
            uri=grep_uri,
            description="Jailed workspace grep",
            department=DepartmentKind.ENGINEERING,
            baits_pillar=None,
            trigger_phrases=(),
            input_schema={"type": "object", "required": ["path"]},
            output_schema={"type": "object"},
            applies_to_kinds=("md",),
            permissions=("workspace:read",),
            risk_level=RiskLevel.LOW,
            lifecycle_policy=LifecyclePolicy.ALWAYS_ON,
        )
    )

    async def grep_handler(args: dict[str, object], ctx: ToolInvocationContext) -> dict[str, object]:
        return {"matches": 3}

    tool_registry.register_handler(grep_uri, grep_handler)

    # Lead Orchestrator ('chotu://agent/lead/chotu@1.0.0') MUST be blocked from calling tools directly
    lead_ctx = ToolInvocationContext(
        caller_agent_uri=CanonicalUri.parse("chotu://agent/lead/chotu@1.0.0"),
        run_id=uuid4(),
        node_id="lead_turn",
        engagement_slug="acme-cloud",
    )
    with pytest.raises(SubagentsOnlyViolationError):
        await tool_registry.invoke(grep_uri, {"path": "notes.md"}, lead_ctx)


@pytest.mark.asyncio
async def test_agent_runtime_and_tool_registry_emit_nested_otel_spans_and_enforce_jail(tmp_path: Path) -> None:
    workspaces_root = tmp_path / "workspaces"
    (workspaces_root / "acme-cloud").mkdir(parents=True)
    (workspaces_root / "acme-cloud" / "sow.md").write_text("SOW milestone 1", encoding="utf-8")

    registry = InMemoryCapabilityRegistry()
    outbox = InMemoryOutboxAdapter()
    hooks = DefaultHookRegistryAdapter(outbox=outbox)
    obs = InMemoryObservabilityAdapter()
    tool_registry = SovereignToolRegistry(locator=registry, hooks=hooks, observability=obs)
    agent_runtime = SovereignAgentRuntime(locator=registry, hooks=hooks, observability=obs)

    grep_uri = CanonicalUri.parse("chotu://tool/core/grep@1.0.0")
    biz_agent_uri = CanonicalUri.parse("chotu://agent/architects/business@1.0.0")

    await registry.register(
        CapabilityDescriptor(
            uri=grep_uri,
            description="Jailed workspace grep",
            department=DepartmentKind.DELIVERY,
            baits_pillar=BaitsPillar.BUSINESS,
            trigger_phrases=(),
            input_schema={"type": "object", "required": ["path"]},
            output_schema={"type": "object"},
            applies_to_kinds=("md",),
            permissions=("workspace:read",),
            risk_level=RiskLevel.LOW,
            lifecycle_policy=LifecyclePolicy.ALWAYS_ON,
        )
    )
    await registry.register(
        CapabilityDescriptor(
            uri=biz_agent_uri,
            description="Business Architect subagent",
            department=DepartmentKind.DELIVERY,
            baits_pillar=BaitsPillar.BUSINESS,
            trigger_phrases=(),
            input_schema={"type": "object"},
            output_schema={"type": "object"},
            applies_to_kinds=("md",),
            permissions=("workspace:read",),
            risk_level=RiskLevel.LOW,
            lifecycle_policy=LifecyclePolicy.ALWAYS_ON,
        )
    )

    async def grep_handler(args: dict[str, object], ctx: ToolInvocationContext) -> dict[str, object]:
        assert ctx.workspace_jail is not None
        safe_path = ctx.workspace_jail.resolve_safe_path(str(args["path"]))
        return {"content": safe_path.read_text(encoding="utf-8")}

    tool_registry.register_handler(grep_uri, grep_handler)

    async def biz_agent_handler(env: SubagentEnvelope, subagent_span_id: str) -> dict[str, object]:
        tool_out = await tool_registry.invoke(
            grep_uri,
            {"path": str(env.payload["target_file"])},
            ToolInvocationContext(
                caller_agent_uri=biz_agent_uri,
                run_id=env.run_id,
                node_id=env.node_id,
                engagement_slug=env.engagement_slug,
                parent_span_id=subagent_span_id,
                workspace_jail=env.workspace_jail,
            ),
        )
        return {"summary": tool_out["content"], "citations": ["[[sow#^f-1]]"]}

    agent_runtime.register_agent_handler(biz_agent_uri, biz_agent_handler)

    jail = EngagementWorkspaceJail(workspaces_root=workspaces_root, engagement_slug="acme-cloud")
    run_id = uuid4()

    # 1. Valid call inside engagement jail succeeds and records parent->child spans
    out = await agent_runtime.invoke(
        biz_agent_uri,
        SubagentEnvelope(
            run_id=run_id,
            node_id="biz_node",
            engagement_slug="acme-cloud",
            payload={"target_file": "sow.md"},
            parent_span_id="span-workflow-node-1",
            workspace_jail=jail,
        ),
    )
    assert out["summary"] == "SOW milestone 1"
    assert len(obs.spans) == 2
    subagent_span, tool_span = obs.spans[0], obs.spans[1]
    assert subagent_span.name == "subagent.invoke"
    assert subagent_span.parent_span_id == "span-workflow-node-1"
    assert tool_span.name == "tool.invoke"
    assert tool_span.parent_span_id == subagent_span.span_id

    # 2. Attempting to read outside acme-cloud inside tool call is blocked by EngagementWorkspaceJail
    with pytest.raises(WorkspaceEscapeError):
        await agent_runtime.invoke(
            biz_agent_uri,
            SubagentEnvelope(
                run_id=run_id,
                node_id="biz_node",
                engagement_slug="acme-cloud",
                payload={"target_file": "../globex-corp/secret.md"},
                workspace_jail=jail,
            ),
        )
```

- [ ] **Step 2: Run test to verify it fails**

Run: `uv run pytest backend/tests/platform/test_sovereign_chokepoints.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'backend.platform.orchestration.domain.policy'`

- [ ] **Step 3: Implement `SovereignToolRegistry` and `SovereignAgentRuntime`**

In `SovereignToolRegistry.invoke`:
1. Check `if context.caller_agent_uri.unversioned_key == "chotu://agent/lead/chotu": raise SubagentsOnlyViolationError(...)`.
2. Resolve capability via `await self._locator.resolve_for_invocation(parsed_uri)`.
3. If ` context.workspace_jail is not None` and `"path"` in `args`, pre-validate `context.workspace_jail.resolve_safe_path(str(args["path"]))`.
4. Run `HookEvent.PRE_TOOL_INVOKE` interceptors, open OTel span `"tool.invoke"` with `parent_span_id=context.parent_span_id`, execute handler, record outcome on `self._locator`, and run `HookEvent.POST_TOOL_INVOKE`.

In `SovereignAgentRuntime.invoke`:
1. Resolve agent capability via `self._locator.resolve_for_invocation(parsed_uri)`.
2. Run `HookEvent.PRE_SUBAGENT_DISPATCH` interceptors (which may mutate `envelope.payload`).
3. Open OTel span `"subagent.invoke"`, invoke agent handler passing `span.span_id`, and run `HookEvent.POST_SUBAGENT_DISPATCH` guard hooks before returning.

- [ ] **Step 4: Run test to verify it passes**

Run: `uv run pytest backend/tests/platform/test_sovereign_chokepoints.py -v && uv run mypy --strict backend/platform`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add backend/platform/orchestration/domain/policy.py backend/platform/harness/domain/observability.py backend/platform/harness/ports/observability_port.py backend/platform/harness/adapters/in_memory_observability.py backend/platform/tools/ backend/platform/multiagent/domain/execution.py backend/platform/multiagent/ports/agent_runtime_port.py backend/platform/multiagent/adapters/agent_runtime.py backend/tests/platform/test_sovereign_chokepoints.py
git commit -m "feat(tools,multiagent): implement AgentRuntime.invoke and ToolRegistry.invoke chokepoints with Subagents-Only guard"
```

---

### Task 8: `WorkClassifierPort`, `DynamicWorkflowComposer` (Probationary Crystallization & Mid-Flight Splicing) & `bootstrap.py` Composition Root

Finally, we connect the classification gate, dynamic DAG composition, probationary workflow crystallization, and the composition root (`backend/platform/harness/bootstrap.py`). Every inbound piece of work enters through `WorkClassifierPort.classify()` to produce a `WorkClassification`. Then `DynamicWorkflowComposer.resolve_or_compose()` checks if an `ESTABLISHED` DAG already matches that classification signature (System-1 fast path with zero LLM planning overhead). If not, it queries `ServiceDiscoveryPort` and composes a `CANDIDATE` DAG of subagents.

To prevent a lucky one-off dynamic plan from polluting System-1 habits, `record_workflow_verification()` requires **3 consecutive verified successes** before a `CANDIDATE` plan graduates to `ESTABLISHED` (and resets the counter to `0` if any run fails verification). Finally, `bootstrap.py` wires all infrastructure ports via Constructor Dependency Injection so the entire Slice 1 chassis can be booted and tested end-to-end.

**Files:**
- Create: `backend/platform/orchestration/domain/classification.py`
- Create: `backend/platform/orchestration/ports/work_classifier_port.py`
- Create: `backend/platform/orchestration/adapters/rule_and_checkpoint_classifier.py`
- Create: `backend/platform/orchestration/composer.py`
- Create: `backend/platform/harness/bootstrap.py`
- Test: `backend/tests/platform/orchestration/test_classifier_composer_and_bootstrap.py`

**Interfaces:**
- Consumes: All ports and domain types from Tasks 1–7.
- Produces:
  - `ComplexityKind(StrEnum)`: `FAST_PATH_DAG = "fast_path_dag"`, `DYNAMIC_DAG = "dynamic_dag"`, `NOVEL_ESCALATION = "novel_escalation"`.
  - `WorkClassification(BaseModel, frozen=True)`: `intent: str`, `mode: OperatingMode`, `engagement_slug: str | None`, `departments: tuple[DepartmentKind, ...]`, `baits_pillars: tuple[BaitsPillar, ...]`, `scope: ScopeKind`, `perspective: PerspectiveKind`, `complexity: ComplexityKind`, `risk_level: RiskLevel`, `confidence: float`, property `signature_key: str`.
  - `WorkClassifierPort(ABC)`:
    - `async def classify(self, *, raw_input: str, engagement_slug: str | None = None, file_kind: str | None = None) -> WorkClassification`
  - `DynamicWorkflowComposer`:
    - `def __init__(self, discovery: ServiceDiscoveryPort, locator: ServiceLocatorPort) -> None`
    - `async def resolve_or_compose(self, classification: WorkClassification) -> tuple[WorkflowPlan, bool]` (returns `(plan, used_established_fast_path)`).
    - `def record_workflow_verification(self, signature_key: str, *, verified_ok: bool, operator_promoted: bool = False) -> WorkflowPlan` (graduates `CANDIDATE -> ESTABLISHED` on 3rd consecutive `verified_ok=True` or `operator_promoted=True`; resets `verified_success_count = 0` on `verified_ok=False`).
  - `PlatformContainer` and `build_platform_container(...) -> PlatformContainer` in `backend/platform/harness/bootstrap.py`.

- [ ] **Step 1: Write the failing test**

Create `backend/tests/platform/orchestration/test_classifier_composer_and_bootstrap.py`:

```python
from pathlib import Path
import pytest

from backend.platform.harness.bootstrap import build_in_memory_platform_container
from backend.platform.multiagent.domain.capability import CapabilityDescriptor, LifecyclePolicy
from backend.platform.multiagent.domain.uris import CanonicalUri
from backend.platform.orchestration.domain.taxonomy import (
    BaitsPillar,
    DepartmentKind,
    OperatingMode,
    PerspectiveKind,
    RiskLevel,
    ScopeKind,
)
from backend.platform.orchestration.domain.workflow import WorkflowPlanStatus


@pytest.mark.asyncio
async def test_probationary_dag_crystallization_requires_three_consecutive_successes(tmp_path: Path) -> None:
    container = build_in_memory_platform_container(workspaces_root=tmp_path / "workspaces")

    # Register BAITS Business & Security Architect subagents in ServiceDiscoveryPort
    for pillar_name, pillar_enum in [("business", BaitsPillar.BUSINESS), ("security", BaitsPillar.SECURITY)]:
        await container.capability_registry.register(
            CapabilityDescriptor(
                uri=CanonicalUri.parse(f"chotu://agent/architects/{pillar_name}@1.0.0"),
                description=f"Synthesize {pillar_name} architecture from transcript facts",
                department=DepartmentKind.DELIVERY,
                baits_pillar=pillar_enum,
                trigger_phrases=(f"{pillar_name} architect",),
                input_schema={"type": "object"},
                output_schema={"type": "object"},
                applies_to_kinds=("transcript",),
                permissions=("workspace:read",),
                risk_level=RiskLevel.LOW,
                lifecycle_policy=LifecyclePolicy.ALWAYS_ON,
            )
        )

    classification = await container.classifier.classify(
        raw_input="Client stated in the SSO meeting that IAM latency must be under 50ms and business value is high",
        engagement_slug="acme-cloud",
        file_kind="transcript",
    )
    assert classification.mode == OperatingMode.WORK_ENGAGEMENT
    assert classification.scope == ScopeKind.CLIENT
    assert classification.perspective == PerspectiveKind.CLIENT_STATED
    assert BaitsPillar.BUSINESS in classification.baits_pillars
    assert BaitsPillar.SECURITY in classification.baits_pillars

    # First resolution composes a new dynamic DAG with status == CANDIDATE
    plan, used_fast_path = await container.composer.resolve_or_compose(classification)
    assert used_fast_path is False
    assert plan.status == WorkflowPlanStatus.CANDIDATE
    assert plan.verified_success_count == 0
    assert len(plan.nodes) == 2

    # 2 consecutive successes -> still CANDIDATE; a failure resets streak to 0!
    container.composer.record_workflow_verification(classification.signature_key, verified_ok=True)
    p_after_2 = container.composer.record_workflow_verification(classification.signature_key, verified_ok=True)
    assert p_after_2.status == WorkflowPlanStatus.CANDIDATE
    assert p_after_2.verified_success_count == 2

    p_failed = container.composer.record_workflow_verification(classification.signature_key, verified_ok=False)
    assert p_failed.status == WorkflowPlanStatus.CANDIDATE
    assert p_failed.verified_success_count == 0

    # 3 consecutive verified successes -> graduates to ESTABLISHED and routes via System-1 fast path
    container.composer.record_workflow_verification(classification.signature_key, verified_ok=True)
    container.composer.record_workflow_verification(classification.signature_key, verified_ok=True)
    p_graduated = container.composer.record_workflow_verification(classification.signature_key, verified_ok=True)
    assert p_graduated.status == WorkflowPlanStatus.ESTABLISHED
    assert p_graduated.verified_success_count == 3

    fast_plan, used_fast_path_now = await container.composer.resolve_or_compose(classification)
    assert used_fast_path_now is True
    assert fast_plan.status == WorkflowPlanStatus.ESTABLISHED
```

- [ ] **Step 2: Run test to verify it fails**

Run: `uv run pytest backend/tests/platform/orchestration/test_classifier_composer_and_bootstrap.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'backend.platform.harness.bootstrap'`

- [ ] **Step 3: Implement `RuleAndCheckpointClassifierAdapter`, `DynamicWorkflowComposer`, and `bootstrap.py`**

Implement `RuleAndCheckpointClassifierAdapter` to classify operating mode, 6-value `scope`, 5-value `perspective`, and `baits_pillars` from calibrated keyword/taxonomy rules. Implement `DynamicWorkflowComposer` with an internal plan registry indexed by `classification.signature_key`, and wire all adapters via Constructor Dependency Injection in `backend/platform/harness/bootstrap.py`.

- [ ] **Step 4: Run full Slice Group 1 test suite and strict typecheck**

Run: `uv run pytest backend/tests/platform/ -v && uv run mypy --strict backend/platform`
Expected: PASS (all Slice Group 1 tests pass with 0 mypy errors)

- [ ] **Step 5: Commit**

```bash
git add backend/platform/orchestration/domain/classification.py backend/platform/orchestration/ports/work_classifier_port.py backend/platform/orchestration/adapters/rule_and_checkpoint_classifier.py backend/platform/orchestration/composer.py backend/platform/harness/bootstrap.py backend/tests/platform/orchestration/test_classifier_composer_and_bootstrap.py
git commit -m "feat(orchestration,harness): add WorkClassifierPort, DynamicWorkflowComposer probationary crystallization, and bootstrap DI"
```
