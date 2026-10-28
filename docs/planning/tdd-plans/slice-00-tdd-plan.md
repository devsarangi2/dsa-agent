# Slice Group 0: Foundation, Standards, Doc-Chain & Hexagonal Skeleton Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Establish the repository's root documentation chain, harvested engineering standards, 9-layer Fractal Hexagonal (`platform/` vs. `inventory/`) Python package skeleton, AST-based hexagonal boundary enforcement, and Tier-0 hardened PostgreSQL 16 + `pgvector` multi-schema (`core`, `universal`, `personal`, `eng_<slug>`) bootstrap infrastructure.

**Architecture:** Slice Group 0 locks in the structural chassis before any runtime agent or workflow code is written. It separates `backend/platform/` (9 domain-agnostic mechanism layers, each internally divided into `domain/`, `ports/`, and `adapters/`) from `backend/inventory/` ("Chotu Corp" capabilities), enforces import isolation via a zero-dependency Python AST verifier, and provisions schema-isolated PostgreSQL 16 storage via an idempotent async schema bootstrap adapter behind a hexagonal port.

**Tech Stack:** Python 3.12+, `uv`, `pytest`, `pytest-asyncio`, `ruff`, `mypy --strict`, `psycopg[binary,pool]` (v3 async/sync driver) / `asyncpg`, Docker Compose (`pgvector/pgvector:pg16`), standard-library `ast` & `pathlib`.

**Spec:** [`docs/planning/slice-00-foundation-standards-and-skeleton.md`](file:///home/nox/Developer/ds2/dsa-agent/docs/planning/slice-00-foundation-standards-and-skeleton.md) · [`docs/planning/chotu_architecture_plan.md`](file:///home/nox/Developer/ds2/dsa-agent/docs/planning/chotu_architecture_plan.md)

## Global Constraints

- **Conversational Prose Standard (`AGENTS.md` Hard Mandate #1):** Every human-facing markdown file (`PRD.md`, `ARCHITECTURE.md`, `HARVEST.md`, `RULES.md`, `BACKLOG.md`, `docs/standards/*.md`, `docs/adr/*.md`) must be written in natural, explanatory, plain-English prose without placeholder markers (`TODO`, `TBD`, `FIXME`) or sterile bullet dumps.
- **Two-Tree Microkernel Split (`AGENTS.md` Hard Mandate #4.1):** `backend/platform/` contains exactly the 9 mechanism layers (`brain`, `tools`, `orchestration`, `agentic_loop`, `harness`, `memory`, `knowledge`, `environment`, `multiagent`), and every layer repeats `domain/`, `ports/`, and `adapters/`. `backend/platform/` never imports `backend/inventory/`.
- **Zero Vendor SDKs in `domain/` (`docs/standards/hexagonal.md` Rule #1):** No file under `backend/platform/**/domain/**` may import vendor or infrastructure packages (`fastapi`, `sqlalchemy`, `asyncpg`, `psycopg`, `pgvector`, `litellm`, `openai`, `anthropic`, `httpx`, `redis`, `celery`, `boto3`).
- **Subagents-Only Structural Rule (`AGENTS.md` Hard Mandate #4.3):** `backend/inventory/agents/lead/chotu/**` may never import tool execution adapters from `backend/inventory/tools/**` or `backend/platform/tools/adapters/**`.
- **Read-Only Sibling Repositories (`AGENTS.md` Hard Mandate #4.6):** Sibling repos in `/home/nox/Developer/ds2/` (`dsa-prj`, `dsa-companion-config`, `okf-kb-manager`, `dsappz-dsrag-prj`, `classifier-training`, `substrate`) are strictly read-only.
- **Multi-Schema Postgres Isolation (`AGENTS.md` Hard Mandate #3):** PostgreSQL 16 + `pgvector` runs as Tier-0 infrastructure with extensions `vector`, `pg_trgm`, `btree_gin`, and `uuid-ossp`, isolating schemas `core`, `universal`, `personal`, and dynamically provisioned `eng_<slug>`.

## Review Focus

1. **Multi-line parenthesized relative imports (`from ...adapters import (...)`) bypassing regex/line-grep boundary checks:** A line-by-line regex grep misses multi-line or relative imports (`level > 0` in Python's AST) that resolve from `domain/` into `adapters/` or from `platform/` into `inventory/`. Pinned by `test_detects_relative_import_from_domain_to_adapters` in Task 3.
2. **Submodule vendor imports (`import sqlalchemy.ext.asyncio` or `from litellm.main import completion`) slipping past exact top-level package name checks:** Checking `node.module in BANNED` instead of `node.module.split(".")[0] in BANNED` allows submodule imports of banned SDKs inside `domain/`. Pinned by `test_detects_submodule_vendor_sdk_imports_in_domain` in Task 3.
3. **SQL injection or invalid identifier characters in dynamic engagement slugs (`eng_<slug>`):** Passing an untrusted slug like `"acme; DROP SCHEMA core CASCADE; --"` or `"Acme-Corp!"` into `CREATE SCHEMA IF NOT EXISTS eng_<slug>` without strict validation (`^[a-z0-9][a-z0-9_]{0,48}$`) risks SQL injection or broken unquoted Postgres identifiers. Pinned by `test_rejects_malicious_or_malformed_engagement_slugs` in Task 4.
4. **Placeholder markers (`TODO`, `TBD`, `FIXME`) or broken relative/file links in the root doc-chain:** A hastily scaffolded doc-chain file that exists on disk but contains stub markers or dead markdown links breaks the `doc-chain.md` review contract. Pinned by `test_doc_chain_files_have_substantive_prose_and_valid_links` in Task 1.
5. **Non-idempotent schema & extension bootstrap on repeated startup:** Running `bootstrap_core_schemas()` or `provision_engagement_schema("acme")` twice on an already-initialized database must succeed cleanly without raising `DuplicateSchema` or `DuplicateObject` errors. Pinned by `test_schema_bootstrap_and_engagement_provisioning_are_idempotent` in Task 4.

---

## File Map Overview

Before diving into the tasks, here is how the files created in Slice Group 0 map to their responsibilities:

- **Governance, Standards & Doc-Chain (Task 1)**:
  - `docs/standards/{hexagonal.md,code-structure.md,doc-chain.md,slice-delivery.md,engineering-discipline.md}` — Harvested and adapted engineering rules from `dsa-prj/docs/standards/`.
  - `PRD.md`, `ARCHITECTURE.md`, `HARVEST.md`, `RULES.md`, `BACKLOG.md` — The root documentation chain anchoring product scope, system topology, sibling-repo harvest ledger, invariants, and slice backlog.
  - `docs/adr/0001-*.md` through `0008-*.md` — Concise Architecture Decision Records linking directly to the 7 topic-level retrospectives in `brain-logs/conversation-retrospectives/`.
  - `backend/tests/governance/test_doc_chain_integrity.py` — Automated verification of the entire doc-chain, ADR set, and link integrity.
- **Python Toolchain & 9-Layer Fractal Hexagonal Skeleton (Task 2)**:
  - `pyproject.toml` & `Makefile` — Root workspace configuration (`uv`, `pytest`, `ruff`, `mypy --strict`) and developer targets (`make lint`, `make typecheck`, `make check-boundaries`, `make test`, `make up`, `make down`).
  - `backend/platform/<9_layers>/{domain,ports,adapters}/__init__.py` — The 9 mechanism layers (`brain`, `tools`, `orchestration`, `agentic_loop`, `harness`, `memory`, `knowledge`, `environment`, `multiagent`).
  - `backend/inventory/{agents,workflows,hooks,tools,skills,packs}/...` — The shipped "Chotu Corp" capability structure.
  - `backend/tests/architecture/test_directory_topology.py` — Automated check proving the physical directory tree matches the Microkernel + Fractal Hexagonal blueprint.
- **AST Hexagonal Boundary Enforcer (Task 3)**:
  - `backend/platform/harness/domain/boundary_rules.py` — Pure domain rules and AST checker inspecting Python modules for hexagonal and Subagents-Only import violations.
  - `backend/scripts/check_hexagonal_boundaries.py` — CLI entry point wired into `make check-boundaries`.
  - `backend/tests/architecture/test_hexagonal_boundaries.py` — Unit and live-tree tests covering clean imports, submodule vendor SDK imports, relative imports, `platform -> inventory` leaks, and `lead/chotu -> tools` leaks.
- **Tier-0 Hardened Postgres 16 + `pgvector` & Multi-Schema Bootstrap (Task 4)**:
  - `infrastructure/docker-compose.yml` & `infrastructure/postgres/init/01-extensions-and-schemas.sql` — Hardened Postgres 16 + `pgvector` container definition and startup initialization.
  - `backend/platform/harness/domain/schema_names.py` — Pure domain value object (`EngagementSlug`) and canonical schema constants (`CORE_SCHEMA`, `UNIVERSAL_SCHEMA`, `PERSONAL_SCHEMA`).
  - `backend/platform/harness/ports/schema_bootstrap_port.py` — Abstract `SchemaBootstrapPort` contract.
  - `backend/platform/harness/adapters/postgres_schema_bootstrap.py` — Concrete `PostgresSchemaBootstrapAdapter` and `InMemorySchemaBootstrapAdapter` (for fast contract testing alongside live Postgres testing).
  - `backend/tests/platform/harness/test_postgres_schema_bootstrap.py` — Port-contract test suite running against both the in-memory fake and the live Docker Postgres 16 + `pgvector` instance.

---

### Task 1: Harvest Engineering Standards (`docs/standards/*`) & Author Root Doc-Chain (`PRD.md`, `ARCHITECTURE.md`, `HARVEST.md`, `RULES.md`, `BACKLOG.md`, ADRs `0001`–`0008`)

Why does our first TDD task test markdown files? In prior repositories, documentation drift started when root files were either left half-finished with `TODO` placeholders or linked to files that were later renamed. By writing an automated doc-chain integrity test first (`backend/tests/governance/test_doc_chain_integrity.py`), we turn [`doc-chain.md`](file:///home/nox/Developer/ds2/dsa-prj/docs/standards/doc-chain.md) from a manual reminder into a permanent test gate that runs on every `pytest` invocation.

This task harvests the five engineering standard documents from `/home/nox/Developer/ds2/dsa-prj/docs/standards/`, authors the five root documentation chain files (`PRD.md`, `ARCHITECTURE.md`, `HARVEST.md`, `RULES.md`, `BACKLOG.md`), and writes ADRs `0001` through `0008` in `docs/adr/`, ensuring every file is written in clear conversational prose (`narrative-writer` standard) and every internal markdown link resolves cleanly on disk.

**Files:**
- Create: `docs/standards/hexagonal.md`
- Create: `docs/standards/code-structure.md`
- Create: `docs/standards/doc-chain.md`
- Create: `docs/standards/slice-delivery.md`
- Create: `docs/standards/engineering-discipline.md`
- Create: `PRD.md`
- Create: `ARCHITECTURE.md`
- Create: `HARVEST.md`
- Create: `RULES.md`
- Create: `BACKLOG.md`
- Create: `docs/adr/0001-chotu-corp-conscious-and-subconscious-operating-model.md`
- Create: `docs/adr/0002-microkernel-fractal-hexagonal-and-sovereign-in-process-harness.md`
- Create: `docs/adr/0003-constructor-di-vs-runtime-service-locator-and-discovery.md`
- Create: `docs/adr/0004-classifier-first-routing-subagents-only-and-probationary-dags.md`
- Create: `docs/adr/0005-dual-mode-lifecycle-hooks-and-origin-tagged-recursion-fence.md`
- Create: `docs/adr/0006-disjoint-universal-knowledge-engagement-datasets-and-elevation.md`
- Create: `docs/adr/0007-bitemporal-multi-perspective-claims-and-truth-finder-hits-weighing.md`
- Create: `docs/adr/0008-tier-0-hardened-postgres-pgvector-and-four-lane-master-task-queue.md`
- Test: `backend/tests/governance/test_doc_chain_integrity.py`

**Interfaces:**
- Consumes: Read-only reference standards in `/home/nox/Developer/ds2/dsa-prj/docs/standards/` and retrospectives in `brain-logs/conversation-retrospectives/`.
- Produces: Verified repository documentation chain and `validate_doc_chain(repo_root: Path) -> list[str]` in `backend/tests/governance/test_doc_chain_integrity.py`.

- [ ] **Step 1: Write the failing test**

Create `backend/tests/governance/test_doc_chain_integrity.py` to assert that all required root doc-chain files, harvested standards, and ADRs `0001`–`0008` exist, contain substantive prose (`>= 400` characters), have zero standalone `TODO` / `TBD` / `FIXME` placeholders, and contain no broken local markdown links:

```python
from pathlib import Path
import re

REPO_ROOT = Path(__file__).resolve().parents[3]

REQUIRED_ROOT_DOCS = [
    "AGENTS.md",
    "PRD.md",
    "ARCHITECTURE.md",
    "HARVEST.md",
    "RULES.md",
    "BACKLOG.md",
]

REQUIRED_STANDARDS = [
    "docs/standards/hexagonal.md",
    "docs/standards/code-structure.md",
    "docs/standards/doc-chain.md",
    "docs/standards/slice-delivery.md",
    "docs/standards/engineering-discipline.md",
]

REQUIRED_ADRS = [
    "docs/adr/0001-chotu-corp-conscious-and-subconscious-operating-model.md",
    "docs/adr/0002-microkernel-fractal-hexagonal-and-sovereign-in-process-harness.md",
    "docs/adr/0003-constructor-di-vs-runtime-service-locator-and-discovery.md",
    "docs/adr/0004-classifier-first-routing-subagents-only-and-probationary-dags.md",
    "docs/adr/0005-dual-mode-lifecycle-hooks-and-origin-tagged-recursion-fence.md",
    "docs/adr/0006-disjoint-universal-knowledge-engagement-datasets-and-elevation.md",
    "docs/adr/0007-bitemporal-multi-perspective-claims-and-truth-finder-hits-weighing.md",
    "docs/adr/0008-tier-0-hardened-postgres-pgvector-and-four-lane-master-task-queue.md",
]

PLACEHOLDER_PATTERN = re.compile(r"\b(TODO|TBD|FIXME)\b")
MARKDOWN_LINK_PATTERN = re.compile(r"\[[^\]]+\]\(([^)]+)\)")


def test_all_required_doc_chain_files_exist() -> None:
    missing = [
        rel_path
        for rel_path in (REQUIRED_ROOT_DOCS + REQUIRED_STANDARDS + REQUIRED_ADRS)
        if not (REPO_ROOT / rel_path).is_file()
    ]
    assert missing == [], f"Missing required doc-chain files: {missing}"


def test_doc_chain_files_have_substantive_prose_and_valid_links() -> None:
    for rel_path in REQUIRED_ROOT_DOCS + REQUIRED_STANDARDS + REQUIRED_ADRS:
        doc_path = REPO_ROOT / rel_path
        content = doc_path.read_text(encoding="utf-8")
        assert len(content.strip()) >= 400, f"{rel_path} is too short (<400 chars)"

        # Strip fenced code blocks before checking for placeholder tokens
        prose_only = re.sub(r"```.*?```", "", content, flags=re.DOTALL)
        prose_only = re.sub(r"`[^`]+`", "", prose_only)
        assert not PLACEHOLDER_PATTERN.search(prose_only), (
            f"{rel_path} contains unresolved TODO/TBD/FIXME placeholder in prose"
        )

        for match in MARKDOWN_LINK_PATTERN.finditer(prose_only):
            target = match.group(1).strip()
            if target.startswith(("http://", "https://", "mailto:", "#")):
                continue
            clean_target = target.split("#")[0]
            if clean_target.startswith("file://"):
                resolved = Path(clean_target.removeprefix("file://"))
            else:
                resolved = (doc_path.parent / clean_target).resolve()
            assert resolved.exists(), (
                f"Broken link in {rel_path}: {target} -> {resolved} does not exist"
            )
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest backend/tests/governance/test_doc_chain_integrity.py -v`
Expected: FAIL with `AssertionError: Missing required doc-chain files: ['PRD.md', 'ARCHITECTURE.md', ...]`

- [ ] **Step 3: Author `docs/standards/*`, root doc-chain files, and `docs/adr/0001`–`0008` (via `narrative-writer`)**

Author each file in natural conversational prose (delegating to `narrative-writer` per `AGENTS.md` Hard Mandate #1):
- Adapt `docs/standards/{hexagonal,code-structure,doc-chain,slice-delivery,engineering-discipline}.md` from `/home/nox/Developer/ds2/dsa-prj/docs/standards/` for `dsa-agent`'s unified repo layout (`backend/platform/` + `backend/inventory/` + `frontend/`).
- Write `PRD.md` (Chotu Corp Conscious/Subconscious loops, ~10 meetings/day + ~20 docs/day scale reality, CSA/BAITS model, Work vs. Personal R&D modes, Internal Tools vs. External `dsapps-*`).
- Write `ARCHITECTURE.md` (C1–C4 system topology, 9-layer `platform/` chassis, `inventory/` departments, multi-schema Postgres topology, and request/outbox lifecycles).
- Write `HARVEST.md` (complete source-to-target harvest ledger across `dsa-prj`, `dsa-companion-config`, `okf-kb-manager`, `dsappz-dsrag-prj`, `classifier-training`, `substrate`, `dsapps-journal`, and `dsapps-checkin`).
- Write `RULES.md` (non-negotiable hexagonal, Subagents-Only, WorkspaceJail, 6-value `scope:`, 5-value `perspective:`, Truth Finder no-auto-overwrite, and hook recursion-fence rules).
- Write `BACKLOG.md` (tracking Slice Groups 0 through 5 linked to `docs/planning/slice-00-*.md` through `slice-05-*.md`).
- Write ADRs `0001`–`0008` in `docs/adr/`, each containing `Status: Accepted`, `Context`, `Decision`, `Alternatives Considered & Why Rejected`, `Consequences`, and a clickable link to its parent retrospective in `brain-logs/conversation-retrospectives/`.

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest backend/tests/governance/test_doc_chain_integrity.py -v`
Expected: PASS (2 passed)

- [ ] **Step 5: Commit**

```bash
git add docs/standards/ docs/adr/ PRD.md ARCHITECTURE.md HARVEST.md RULES.md BACKLOG.md backend/tests/governance/test_doc_chain_integrity.py
git commit -m "docs(slice-00): harvest engineering standards and author root doc-chain and ADRs 0001-0008"
```

---

### Task 2: Python Project Configuration (`pyproject.toml`, `Makefile`) & 9-Layer Fractal Hexagonal Skeleton (`backend/platform/` + `backend/inventory/`)

Once the documentation chain is locked in, we need the physical Python package tree to mirror our architecture 1:1. As [`code-structure.md`](file:///home/nox/Developer/ds2/dsa-prj/docs/standards/code-structure.md) states, the codebase must *scream the architecture*: opening `backend/platform/` immediately reveals the 9 domain-agnostic mechanism layers, and opening any layer immediately reveals `domain/`, `ports/`, and `adapters/`. Opening `backend/inventory/` reveals the shipped "Chotu Corp" departments, workflows, hooks, tools, skills, and capability-bundle packs.

In this task, we configure the Python project (`pyproject.toml` with `ruff`, `mypy --strict`, and `pytest` settings) and `Makefile`, scaffold the 9-layer `backend/platform/` and `backend/inventory/` package trees alongside their 1:1 `backend/tests/` mirror directories, and pin the entire directory topology with an automated test (`backend/tests/architecture/test_directory_topology.py`).

**Files:**
- Create: `pyproject.toml`
- Create: `Makefile`
- Create: `backend/__init__.py`
- Create: `backend/platform/__init__.py`
- Create: `backend/platform/{brain,tools,orchestration,agentic_loop,harness,memory,knowledge,environment,multiagent}/{__init__.py,domain/__init__.py,ports/__init__.py,adapters/__init__.py}`
- Create: `backend/inventory/__init__.py`
- Create: `backend/inventory/agents/{__init__.py,lead/__init__.py,lead/chotu/__init__.py,domain/__init__.py,domain/architects/__init__.py,domain/builders/__init__.py,domain/ingestion/__init__.py,domain/kb/__init__.py,internal/__init__.py,internal/extractors/__init__.py,internal/classifiers/__init__.py}`
- Create: `backend/inventory/{workflows,hooks,tools,skills,packs}/__init__.py`
- Create: `backend/inventory/packs/{bvf,raid,baits}/README.md`
- Create: `backend/migrations/README.md`
- Test: `backend/tests/architecture/test_directory_topology.py`

**Interfaces:**
- Consumes: Directory layout specification from `ARCHITECTURE.md` and `docs/standards/code-structure.md`.
- Produces: Importable `backend.platform.<layer>.{domain,ports,adapters}` and `backend.inventory.<section>` packages, plus `PLATFORM_LAYERS: tuple[str, ...]` and `INVENTORY_SECTIONS: tuple[str, ...]` constants in `backend/platform/__init__.py`.

- [ ] **Step 1: Write the failing test**

Create `backend/tests/architecture/test_directory_topology.py` to verify that all 9 platform layers exist with `domain/`, `ports/`, and `adapters/` Python packages, that all `inventory/` departments and packs exist, and that `backend/tests/` mirrors `platform/` and `inventory/`:

```python
from pathlib import Path

REPO_ROOT = Path(__file__).resolve().parents[3]
BACKEND_ROOT = REPO_ROOT / "backend"

EXPECTED_PLATFORM_LAYERS = (
    "brain",
    "tools",
    "orchestration",
    "agentic_loop",
    "harness",
    "memory",
    "knowledge",
    "environment",
    "multiagent",
)

EXPECTED_HEXAGONAL_SUBPACKAGES = ("domain", "ports", "adapters")

EXPECTED_INVENTORY_PATHS = (
    "agents/lead/chotu",
    "agents/domain/architects",
    "agents/domain/builders",
    "agents/domain/ingestion",
    "agents/domain/kb",
    "agents/internal/extractors",
    "agents/internal/classifiers",
    "workflows",
    "hooks",
    "tools",
    "skills",
    "packs/bvf",
    "packs/raid",
    "packs/baits",
)


def test_nine_layer_fractal_hexagonal_platform_tree_exists() -> None:
    platform_root = BACKEND_ROOT / "platform"
    assert (platform_root / "__init__.py").is_file()

    for layer in EXPECTED_PLATFORM_LAYERS:
        layer_dir = platform_root / layer
        assert (layer_dir / "__init__.py").is_file(), f"Missing platform layer: {layer}"
        for subpkg in EXPECTED_HEXAGONAL_SUBPACKAGES:
            subpkg_init = layer_dir / subpkg / "__init__.py"
            assert subpkg_init.is_file(), f"Missing {layer}/{subpkg}/__init__.py"


def test_inventory_tree_and_test_mirror_exist() -> None:
    inventory_root = BACKEND_ROOT / "inventory"
    assert (inventory_root / "__init__.py").is_file()

    for rel_path in EXPECTED_INVENTORY_PATHS:
        target_dir = inventory_root / rel_path
        assert target_dir.is_dir(), f"Missing inventory directory: {rel_path}"

    tests_root = BACKEND_ROOT / "tests"
    for layer in EXPECTED_PLATFORM_LAYERS:
        assert (tests_root / "platform" / layer).is_dir(), (
            f"Missing mirrored test directory: tests/platform/{layer}"
        )
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest backend/tests/architecture/test_directory_topology.py -v`
Expected: FAIL with `AssertionError: assert PosixPath('.../backend/platform/__init__.py').is_file()`

- [ ] **Step 3: Create `pyproject.toml`, `Makefile`, and the 9-layer `backend/platform/` + `backend/inventory/` package skeleton**

Create root `pyproject.toml` configuring `pytest`, `ruff`, and `mypy --strict`:

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "dsa-agent"
version = "0.1.0"
description = "Chotu (dsa-agent) — Sovereign Hexagonal Autonomous Enterprise & Consulting Engine"
readme = "PRD.md"
requires-python = ">=3.12"
dependencies = [
    "pydantic>=2.9.0",
    "psycopg[binary,pool]>=3.2.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.3.0",
    "pytest-asyncio>=0.24.0",
    "ruff>=0.7.0",
    "mypy>=1.13.0",
]

[tool.hatch.build.targets.wheel]
packages = ["backend"]

[tool.pytest.ini_options]
testpaths = ["backend/tests"]
pythonpath = ["."]
asyncio_mode = "auto"

[tool.ruff]
line-length = 100
target-version = "py312"
src = ["backend"]

[tool.ruff.lint]
select = ["E", "F", "I", "UP", "B", "SIM"]

[tool.mypy]
python_version = "3.12"
strict = true
packages = ["backend"]
```

Scaffold all `__init__.py` files across `backend/platform/<layer>/{domain,ports,adapters}`, `backend/inventory/...`, and `backend/tests/platform/<layer>`, exporting `PLATFORM_LAYERS: tuple[str, ...]` from `backend/platform/__init__.py`, and create `Makefile` with targets `up`, `down`, `lint`, `typecheck`, `check-boundaries`, and `test`.

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest backend/tests/architecture/test_directory_topology.py -v`
Expected: PASS (2 passed)

- [ ] **Step 5: Commit**

```bash
git add pyproject.toml Makefile backend/
git commit -m "build(slice-00): scaffold 9-layer fractal hexagonal platform and inventory skeleton"
```

---

### Task 3: AST-Based Hexagonal Boundary Enforcer (`backend/platform/harness/domain/boundary_rules.py` & `backend/scripts/check_hexagonal_boundaries.py`)

Why use Python's `ast` module instead of a shell `grep` one-liner to enforce hexagonal boundaries? Because real Python code uses multi-line parenthesized imports (`from ...adapters import (\n PostgresAdapter,\n)`), relative imports (`level >= 1`), and submodule imports (`import sqlalchemy.ext.asyncio`). A naive regex grep misses relative paths or breaks on multi-line formatting, whereas an AST walker resolves the exact module target, line number, and source layer with 100% precision and zero external dependencies.

In this task, we implement `HexagonalBoundaryChecker` inside `backend/platform/harness/domain/boundary_rules.py` and expose a CLI runner at `backend/scripts/check_hexagonal_boundaries.py`. It enforces four hard architectural laws across the repository:
1. **Domain Purity (`NO_VENDOR_IN_DOMAIN` & `NO_ADAPTER_IN_DOMAIN`)**: Files under `backend/platform/**/domain/**` cannot import banned vendor/infrastructure SDKs (`fastapi`, `sqlalchemy`, `asyncpg`, `psycopg`, `pgvector`, `litellm`, `openai`, `anthropic`, `httpx`, `redis`, `celery`, `boto3`, `starlette`, `pydantic_settings`) or any `adapters` module (whether via absolute or relative import).
2. **Port Purity (`NO_ADAPTER_IN_PORTS`)**: Files under `backend/platform/**/ports/**` cannot import any `adapters` module.
3. **Chassis Independence (`NO_INVENTORY_IN_PLATFORM`)**: Files under `backend/platform/**` cannot import anything from `backend/inventory/**`.
4. **Subagents-Only Lead Orchestrator (`NO_RAW_TOOLS_IN_LEAD_ORCHESTRATOR`)**: Files under `backend/inventory/agents/lead/chotu/**` cannot import concrete tool implementations from `backend/inventory/tools/**` or `backend/platform/tools/adapters/**`.

**Files:**
- Create: `backend/platform/harness/domain/boundary_rules.py`
- Create: `backend/scripts/check_hexagonal_boundaries.py`
- Test: `backend/tests/architecture/test_hexagonal_boundaries.py`

**Interfaces:**
- Consumes: `backend/` package tree created in Task 2.
- Produces:
  - `BoundaryViolation(file_path: Path, line_number: int, rule_id: str, imported_target: str, message: str)` (frozen dataclass in `backend/platform/harness/domain/boundary_rules.py`)
  - `BANNED_DOMAIN_VENDOR_ROOTS: frozenset[str]`
  - `check_source_boundaries(source_code: str, relative_posix_path: str) -> list[BoundaryViolation]`
  - `scan_repository_boundaries(backend_root: Path) -> list[BoundaryViolation]`

- [ ] **Step 1: Write the failing test**

Create `backend/tests/architecture/test_hexagonal_boundaries.py` exercising all four rules, including submodule vendor SDK imports (`Review Focus #2`), multi-line parenthesized relative imports (`Review Focus #1`), `platform -> inventory` leaks, `lead/chotu -> tools` leaks, and a full scan of the live `backend/` directory:

```python
from pathlib import Path
from backend.platform.harness.domain.boundary_rules import (
    BoundaryViolation,
    check_source_boundaries,
    scan_repository_boundaries,
)

REPO_ROOT = Path(__file__).resolve().parents[3]


def test_allows_clean_domain_and_port_imports() -> None:
    source = (
        "from dataclasses import dataclass\n"
        "from typing import Protocol\n"
        "from backend.platform.brain.domain.models import PromptRequest\n"
    )
    violations = check_source_boundaries(
        source, "backend/platform/brain/ports/llm_port.py"
    )
    assert violations == []


def test_detects_submodule_vendor_sdk_imports_in_domain() -> None:
    source = (
        "import sqlalchemy.ext.asyncio\n"
        "from litellm.main import completion\n"
        "from fastapi import APIRouter\n"
    )
    violations = check_source_boundaries(
        source, "backend/platform/brain/domain/router.py"
    )
    rule_ids = [v.rule_id for v in violations]
    assert rule_ids == [
        "NO_VENDOR_IN_DOMAIN",
        "NO_VENDOR_IN_DOMAIN",
        "NO_VENDOR_IN_DOMAIN",
    ]
    assert violations[0].imported_target == "sqlalchemy.ext.asyncio"
    assert violations[1].imported_target == "litellm.main"


def test_detects_relative_import_from_domain_to_adapters() -> None:
    source = (
        "from ..adapters.postgres_store import (\n"
        "    PostgresWorkflowStoreAdapter,\n"
        ")\n"
    )
    violations = check_source_boundaries(
        source, "backend/platform/orchestration/domain/workflow.py"
    )
    assert len(violations) == 1
    assert violations[0].rule_id == "NO_ADAPTER_IN_DOMAIN"
    assert "adapters" in violations[0].imported_target


def test_detects_platform_importing_inventory() -> None:
    source = "from backend.inventory.agents.lead.chotu import LeadAgent\n"
    violations = check_source_boundaries(
        source, "backend/platform/multiagent/adapters/dispatcher.py"
    )
    assert len(violations) == 1
    assert violations[0].rule_id == "NO_INVENTORY_IN_PLATFORM"


def test_detects_lead_orchestrator_importing_raw_tools() -> None:
    source = "from backend.inventory.tools.bash_tool import run_bash\n"
    violations = check_source_boundaries(
        source, "backend/inventory/agents/lead/chotu/orchestrator.py"
    )
    assert len(violations) == 1
    assert violations[0].rule_id == "NO_RAW_TOOLS_IN_LEAD_ORCHESTRATOR"


def test_live_backend_tree_has_zero_boundary_violations() -> None:
    violations = scan_repository_boundaries(REPO_ROOT / "backend")
    assert violations == []
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest backend/tests/architecture/test_hexagonal_boundaries.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'backend.platform.harness.domain.boundary_rules'`

- [ ] **Step 3: Implement `check_source_boundaries` and `scan_repository_boundaries` in `backend/platform/harness/domain/boundary_rules.py` and CLI runner in `backend/scripts/check_hexagonal_boundaries.py`**

Implement `backend/platform/harness/domain/boundary_rules.py` using `ast.parse` and `ast.walk`. Resolve relative imports (`isinstance(node, ast.ImportFrom)` with `node.level > 0`) against the file's package parts (`PurePosixPath(relative_posix_path).parent.parts`) before evaluating rules:

```python
from __future__ import annotations

import ast
from dataclasses import dataclass
from pathlib import Path, PurePosixPath

BANNED_DOMAIN_VENDOR_ROOTS: frozenset[str] = frozenset(
    {
        "fastapi",
        "starlette",
        "sqlalchemy",
        "asyncpg",
        "psycopg",
        "psycopg2",
        "pgvector",
        "litellm",
        "openai",
        "anthropic",
        "httpx",
        "requests",
        "aiohttp",
        "redis",
        "celery",
        "boto3",
        "botocore",
        "pydantic_settings",
    }
)


@dataclass(frozen=True)
class BoundaryViolation:
    file_path: Path
    line_number: int
    rule_id: str
    imported_target: str
    message: str


def _resolve_import_target(
    pkg_parts: tuple[str, ...], module: str | None, level: int
) -> str:
    if level == 0:
        return module or ""
    anchor_len = max(0, len(pkg_parts) - (level - 1))
    base = ".".join(pkg_parts[:anchor_len])
    if module:
        return f"{base}.{module}" if base else module
    return base
```

Complete `check_source_boundaries(source_code: str, relative_posix_path: str) -> list[BoundaryViolation]` and `scan_repository_boundaries(backend_root: Path) -> list[BoundaryViolation]`, and create `backend/scripts/check_hexagonal_boundaries.py` which calls `scan_repository_boundaries`, prints any violations to `stderr`, and exits with code `1` if non-empty or `0` if clean.

- [ ] **Step 4: Run test and CLI script to verify they pass**

Run: `pytest backend/tests/architecture/test_hexagonal_boundaries.py -v && python -m backend.scripts.check_hexagonal_boundaries`
Expected: PASS (6 passed) and CLI prints `Hexagonal boundary check passed: 0 violations found.`

- [ ] **Step 5: Commit**

```bash
git add backend/platform/harness/domain/boundary_rules.py backend/scripts/check_hexagonal_boundaries.py backend/tests/architecture/test_hexagonal_boundaries.py
git commit -m "feat(slice-00): implement AST-based hexagonal boundary enforcer and test suite"
```

---

### Task 4: Tier-0 Hardened PostgreSQL 16 + `pgvector` Docker Compose & Multi-Schema Bootstrap (`core`, `universal`, `personal`, `eng_<slug>`)

Why do we build the multi-schema PostgreSQL bootstrap behind a hexagonal port (`SchemaBootstrapPort`) in Slice Group 0 instead of scattering raw SQL strings across migrations later? Because Chotu's epistemic model depends on physical schema isolation:
- **`core`** holds the 4-Lane Priority Master Task Queue, `WorkflowRun` / `WorkflowNodeRun` state, and capability registries.
- **`universal`** holds the disjoint Universal Knowledge Fabric (`pgvector` HNSW embeddings and elevated engagement learnings).
- **`personal`** holds the operator's Personal R&D Mode studies and experiments.
- **`eng_<slug>`** is provisioned dynamically every time a new client engagement starts (for example, `eng_acme_cloud_2026`), giving that engagement its own closed-world tables and vector indexes.

Because `eng_<slug>` is constructed from an engagement slug, our domain layer (`backend/platform/harness/domain/schema_names.py`) must strictly validate and normalize the slug (`EngagementSlug`) before any SQL adapter ever touches it — preventing SQL injection or malformed schema names (`Review Focus #3`). And following `docs/standards/hexagonal.md` Rule #2 (*"New port method $\Rightarrow$ contract test in the same PR"*), we ship `SchemaBootstrapPort` with a shared contract test suite that runs against both `InMemorySchemaBootstrapAdapter` and `PostgresSchemaBootstrapAdapter` (against the live Docker Compose `pgvector/pgvector:pg16` container).

**Files:**
- Create: `infrastructure/docker-compose.yml`
- Create: `infrastructure/postgres/init/01-extensions-and-schemas.sql`
- Create: `backend/platform/harness/domain/schema_names.py`
- Create: `backend/platform/harness/ports/schema_bootstrap_port.py`
- Create: `backend/platform/harness/adapters/postgres_schema_bootstrap.py`
- Test: `backend/tests/platform/harness/test_postgres_schema_bootstrap.py`

**Interfaces:**
- Consumes: Hexagonal boundary rules from Task 3 (`schema_names.py` in `domain/` imports zero vendor SDKs).
- Produces:
  - `CORE_SCHEMA = "core"`, `UNIVERSAL_SCHEMA = "universal"`, `PERSONAL_SCHEMA = "personal"`, `REQUIRED_EXTENSIONS = ("vector", "pg_trgm", "btree_gin", "uuid-ossp")` in `backend/platform/harness/domain/schema_names.py`
  - `InvalidEngagementSlugError(ValueError)` in `backend/platform/harness/domain/schema_names.py`
  - `EngagementSlug(slug: str)` value object with `.schema_name -> str` (returns `"eng_<slug>"`) in `backend/platform/harness/domain/schema_names.py`
  - `BootstrapStatus(installed_extensions: frozenset[str], existing_schemas: frozenset[str])` in `backend/platform/harness/domain/schema_names.py`
  - `SchemaBootstrapPort` ABC in `backend/platform/harness/ports/schema_bootstrap_port.py` with methods:
    - `bootstrap_core_schemas(self) -> BootstrapStatus`
    - `provision_engagement_schema(self, slug: EngagementSlug) -> str`
    - `inspect_status(self) -> BootstrapStatus`
  - `InMemorySchemaBootstrapAdapter(SchemaBootstrapPort)` and `PostgresSchemaBootstrapAdapter(SchemaBootstrapPort)` in `backend/platform/harness/adapters/postgres_schema_bootstrap.py`

- [ ] **Step 1: Write the failing test**

Create `backend/tests/platform/harness/test_postgres_schema_bootstrap.py` testing `EngagementSlug` validation (`Review Focus #3`), idempotency (`Review Focus #5`), and running the port-contract test suite against both `InMemorySchemaBootstrapAdapter` and `PostgresSchemaBootstrapAdapter`:

```python
import os
import pytest
from backend.platform.harness.domain.schema_names import (
    CORE_SCHEMA,
    PERSONAL_SCHEMA,
    REQUIRED_EXTENSIONS,
    UNIVERSAL_SCHEMA,
    EngagementSlug,
    InvalidEngagementSlugError,
)
from backend.platform.harness.ports.schema_bootstrap_port import SchemaBootstrapPort
from backend.platform.harness.adapters.postgres_schema_bootstrap import (
    InMemorySchemaBootstrapAdapter,
    PostgresSchemaBootstrapAdapter,
)

TEST_DATABASE_URL = os.getenv(
    "CHOTU_TEST_DATABASE_URL",
    "postgresql://chotu:chotu_dev_password@localhost:54329/chotu",
)


def test_engagement_slug_normalizes_valid_slugs() -> None:
    slug = EngagementSlug("acme_cloud_2026")
    assert slug.slug == "acme_cloud_2026"
    assert slug.schema_name == "eng_acme_cloud_2026"


@pytest.mark.parametrize(
    "bad_slug",
    [
        "",
        "Acme_Upper",
        "acme-hyphen",
        "_starts_with_underscore",
        "acme; DROP SCHEMA core CASCADE; --",
        "acme' OR '1'='1",
        "a" * 50,
    ],
)
def test_rejects_malicious_or_malformed_engagement_slugs(bad_slug: str) -> None:
    with pytest.raises(InvalidEngagementSlugError):
        EngagementSlug(bad_slug)


@pytest.fixture(params=["in_memory", "postgres"])
def bootstrap_adapter(request: pytest.FixtureRequest) -> SchemaBootstrapPort:
    if request.param == "in_memory":
        return InMemorySchemaBootstrapAdapter()
    adapter = PostgresSchemaBootstrapAdapter(dsn=TEST_DATABASE_URL)
    if not adapter.can_connect():
        pytest.skip("Live Postgres container not reachable on CHOTU_TEST_DATABASE_URL")
    return adapter


def test_schema_bootstrap_and_engagement_provisioning_are_idempotent(
    bootstrap_adapter: SchemaBootstrapPort,
) -> None:
    # First run provisions extensions and core/universal/personal schemas
    first_status = bootstrap_adapter.bootstrap_core_schemas()
    # Second run must be completely idempotent (Review Focus #5)
    second_status = bootstrap_adapter.bootstrap_core_schemas()

    for ext in REQUIRED_EXTENSIONS:
        assert ext in second_status.installed_extensions
    for schema in (CORE_SCHEMA, UNIVERSAL_SCHEMA, PERSONAL_SCHEMA):
        assert schema in second_status.existing_schemas
    assert first_status.installed_extensions <= second_status.installed_extensions

    # Provision an isolated engagement schema twice idempotently
    eng_slug = EngagementSlug("test_slice0_client")
    created_first = bootstrap_adapter.provision_engagement_schema(eng_slug)
    created_second = bootstrap_adapter.provision_engagement_schema(eng_slug)
    assert created_first == "eng_test_slice0_client"
    assert created_second == "eng_test_slice0_client"

    final_status = bootstrap_adapter.inspect_status()
    assert "eng_test_slice0_client" in final_status.existing_schemas
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest backend/tests/platform/harness/test_postgres_schema_bootstrap.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'backend.platform.harness.domain.schema_names'`

- [ ] **Step 3: Implement `infrastructure/docker-compose.yml`, `schema_names.py`, `schema_bootstrap_port.py`, and `postgres_schema_bootstrap.py`**

1. Create `infrastructure/docker-compose.yml` running `pgvector/pgvector:pg16` on host port `54329` (`chotu` user, `chotu_dev_password`, database `chotu`) with tuned memory/HNSW settings (`shared_buffers=256MB`, `work_mem=32MB`, `maintenance_work_mem=256MB`) and mounting `infrastructure/postgres/init/01-extensions-and-schemas.sql`.
2. Implement `EngagementSlug` in `backend/platform/harness/domain/schema_names.py` enforcing regex `^[a-z0-9][a-z0-9_]{0,48}$` and raising `InvalidEngagementSlugError` otherwise.
3. Implement `SchemaBootstrapPort` ABC in `backend/platform/harness/ports/schema_bootstrap_port.py`.
4. Implement `InMemorySchemaBootstrapAdapter` and `PostgresSchemaBootstrapAdapter` (using `psycopg.connect` and `psycopg.sql.Identifier(slug.schema_name)` for safe identifier quoting in `CREATE SCHEMA IF NOT EXISTS {}`) in `backend/platform/harness/adapters/postgres_schema_bootstrap.py`.

- [ ] **Step 4: Start the Postgres container and run the full test and boundary suite**

Run:
```bash
docker compose -f infrastructure/docker-compose.yml up -d --wait
pytest -v
ruff check backend
mypy --strict backend
python -m backend.scripts.check_hexagonal_boundaries
```
Expected: All tests PASS (including both `in_memory` and live `postgres` contract parameterized runs), `ruff` clean, `mypy --strict` clean, and 0 hexagonal boundary violations.

- [ ] **Step 5: Commit**

```bash
git add infrastructure/ backend/platform/harness/ backend/tests/platform/harness/
git commit -m "feat(slice-00): add hardened pgvector docker-compose and multi-schema bootstrap port and adapters"
```
