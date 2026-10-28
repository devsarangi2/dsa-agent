# Slice Group 2: Disjoint Knowledge Fabric, Small-to-Big Recall, Engagement Master Data, Truth Finder + HITS & Elevation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the `platform/knowledge/` fractal hexagonal subsystem (`domain/ | ports/ | adapters/`), PostgreSQL schema provisioning for `eng_<slug>.*`, `universal.*`, and `personal.*`, the durable `OKFIndexer` (`okf-wiki/` $\rightarrow$ `pgvector` facts + pillar sections), the Small-to-Big Librarian, the `Discover -> Select -> Ingest` Universal Knowledge bridge, Engagement Master Data repositories, the active Truth Finder + HITS + 2-Step Make/Break & Plausibility Weighing engine, and the 5-Lens Engagement Closure Retrospective Elevator.

**Architecture:** `okf-wiki/` Markdown on disk is the canonical write-master for both isolated engagements (`workspaces/<slug>/okf-wiki/`) and the disjoint Universal Knowledge vault (`CHOTU_VAULT_PATH`). A durable `OKFIndexer` tracked by SHA-256 content hashes in `<schema>.okf_sync_ledger` projects both atomic claims (`^f-id`) and BAITS pillar sections (`## <Pillar>`) into PostgreSQL 16 `pgvector` HNSW + FTS tables so the Small-to-Big Librarian can match fine-grained vectors in milliseconds and hydrate coherent parent `## <Pillar>` + `baits-<pillar>.md` synthesis blocks. Competing claims across the 6-value `scope:` and 5-value `perspective:` axes never auto-overwrite one another; instead, `TruthFinderEngine` runs iterative Truth Finder + HITS scoring followed by Step 1 (`RelationshipImpactAnalyzer` Make/Break graph simulation across ontology, synthesis, and Engagement Master Data) and Step 2 (`PlausibilityWeigher` domain coherence scoring using reference standards ingested from Universal Knowledge).

**Tech Stack:** Python 3.12+, `pydantic>=2.9`, `asyncpg>=0.30`, PostgreSQL 16+ (`pgvector` 0.7+ HNSW `vector(768)`, `tsvector` GIN, `pg_trgm`), `pytest>=8.3`, `pytest-asyncio>=0.24`, `mypy --strict`.

**Spec:** [`docs/planning/slice-02-knowledge-fabric-truth-finder-and-elevation.md`](file:///home/nox/Developer/ds2/dsa-agent/docs/planning/slice-02-knowledge-fabric-truth-finder-and-elevation.md)

## Global Constraints

- **Zero Vendor SDKs in `domain/`**: Files under `backend/platform/knowledge/domain/` and `backend/platform/memory/domain/` MUST NOT import `asyncpg`, `sqlalchemy`, `pgvector`, `fastapi`, `litellm`, or `httpx` (enforced by CI import grep).
- **Strict Schema Boundary Naming**: Engagement schemas MUST match `^eng_[a-z0-9_]+$`; the Universal Knowledge schema is `universal`; the Personal R&D schema is `personal`. Cross-engagement SQL queries (`eng_a` reading `eng_b`) are forbidden and must raise `CrossScopeAccessError`.
- **6-Value `KnowledgeScope` Enum**: Exact values are `'engagement'`, `'client'`, `'internal'`, `'reference'`, `'prior-project'`, and `'deliverable'`.
- **5-Value `ClaimPerspective` Enum**: Exact values are `'client_stated'`, `'internal_team'`, `'personal_take'`, `'work_research'`, and `'personal_rd'`.
- **5-Value `BaitsPillar` Enum**: Exact values are `'Business'`, `'Application'`, `'Information'`, `'Technology'`, and `'Security'`.
- **No Forced Auto-Conflict Resolution**: Ingesting a newer claim that contradicts an existing active claim on the same `(subject_slug, attribute_key)` MUST keep both claims `is_active=True` and attach both to a `ConflictCluster` unless `evolution_type == 'ITERATION'` with an explicit `lineage_parent_id`.
- **Port-Contract Parity**: Every Port ABC in `backend/platform/knowledge/ports/` ships with a reusable contract test suite executed against both an in-memory Fake adapter and the PostgreSQL/`pgvector` adapter.

## Review Focus

1. **Citeability Firewall Bypass (`scope: internal` or `perspective: personal_rd` leaking into a client-facing `deliverable` context)**: When a subagent or Librarian prepares a `ContextPacket` with `for_client_deliverable=True`, any claim stamped `scope='internal'`, `perspective='personal_take'`, or `perspective='personal_rd'` must be stripped before hydration unless explicitly promoted to `scope='deliverable'` (pinned in `Task 1: test_citeability_firewall_blocks_internal_and_personal_claims_from_deliverable` and `Task 4: test_librarian_deliverable_mode_strips_internal_facts_from_hydrated_section`).
2. **Cross-Engagement Schema Leak or Invalid Schema Slug Injection**: Passing a schema name outside `eng_<slug>`, `universal`, or `personal` (or attempting to query `eng_beta` from an `eng_alpha` Librarian context) must immediately fail with `CrossScopeAccessError` before touching PostgreSQL (pinned in `Task 1: test_knowledge_coordinates_reject_cross_engagement_access` and `Task 5: test_universal_bridge_forbids_cross_engagement_source`).
3. **Un-hydrated Atomic Fact Losing Parent Pillar Narrative**: When a query matches a single atomic fact (`^f-102`) inside `## Technology`, the Librarian must never return the isolated one-line claim alone; it must hydrate the parent `## Technology` section text, the sibling corroborated citations in that section, and the matching `baits-technology.md` synthesis block (pinned in `Task 4: test_small_to_big_hydrates_parent_pillar_section_and_synthesis`).
4. **Silent Fact Loss When Embedding Service Fails Mid-Index**: If `EmbeddingPort.embed_texts()` raises a transient error during `OKFIndexer.index_file()`, `<schema>.okf_sync_ledger` must record `status='RETRY_BACKOFF'` with `attempt_count incremented` and `last_error` populated rather than silently marking the file indexed or dropping the outbox event (pinned in `Task 3: test_okf_indexer_records_retry_backoff_on_embedding_failure_without_silent_skip`).
5. **Silent Auto-Overwrite of Signed ADR by Newer Transcript Claim & Broken `engagement://<id>#^<anchor>` Elevation URI**: A newer `client_stated` meeting claim contradicting an accepted `ADR-002` decision must preserve both claims active, enumerate `ADR-002` and its dependent deliverables in `broken_relationships`, and any elevated retrospective learning missing a valid `engagement://eng_<slug>#^retro-<id>` anchor must raise `InvalidElevationCitationError` (pinned in `Task 6: test_truth_finder_preserves_competing_claims_and_ranks_signed_sow_over_casual_chat`, `Task 7: test_make_break_analyzer_lists_broken_adr_and_deliverables`, and `Task 8: test_elevator_rejects_missing_engagement_correlation_uri`).

---

### Task 1: Epistemic Domain Model (`VersionedClaim`, 6-Value `KnowledgeScope`, 5-Value `ClaimPerspective`, Bitemporal Timestamps, Citeability Matrix Firewall & `EngagementCitationUri`)

Before we touch PostgreSQL or parse a single Markdown file, we need pure Python domain primitives that encode Chotu's epistemic laws with zero external library dependencies. In daily consulting work, every atomic statement extracted from a meeting or document carries not just text, but a specific vantage point (`ClaimPerspective`), a boundary of where it came from and where it may be cited (`KnowledgeScope`), and two distinct timestamps: when it was stated in the real world (`observed_at`) versus when Chotu ingested it (`ingested_at`).

This task locks in those value objects alongside three guards that protect the rest of Slice Group 2: the `CiteabilityFirewall` (which guarantees that `internal` delivery notes or `personal_take` / `personal_rd` hypotheses never leak into a client-facing `deliverable`), `KnowledgeCoordinates` (which enforces closed-world engagement isolation), and `EngagementCitationUri` (which validates the `engagement://eng_<slug>#^<anchor>` correlation pointers used by the Elevation Technique in Task 8).

**Files:**
- Create: `backend/platform/knowledge/domain/__init__.py`
- Create: `backend/platform/knowledge/domain/enums.py`
- Create: `backend/platform/knowledge/domain/errors.py`
- Create: `backend/platform/knowledge/domain/models.py`
- Create: `backend/platform/knowledge/domain/citeability.py`
- Test: `backend/tests/platform/knowledge/domain/test_epistemic_domain.py`

**Interfaces:**
- Consumes: Python standard library only (`dataclasses`, `datetime`, `enum`, `re`, `pathlib.PurePosixPath`)
- Produces:
  - `KnowledgeScope(StrEnum)`: `'engagement' | 'client' | 'internal' | 'reference' | 'prior-project' | 'deliverable'`
  - `ClaimPerspective(StrEnum)`: `'client_stated' | 'internal_team' | 'personal_take' | 'work_research' | 'personal_rd'`
  - `BaitsPillar(StrEnum)`: `'Business' | 'Application' | 'Information' | 'Technology' | 'Security'`
  - `EvolutionType(StrEnum)`: `'INITIAL' | 'ITERATION' | 'COMPETING_CLAIM'`
  - `KnowledgeCoordinates(schema_name: str, workspace_slug: str, is_universal: bool = False, is_personal: bool = False)` with `assert_can_read(target_schema: str) -> None`
  - `EngagementCitationUri.parse(raw_uri: str) -> EngagementCitationUri` (`engagement_slug: str`, `anchor_id: str`, `canonical_uri: str`)
  - `VersionedClaim` & `PillarSectionBlock` frozen dataclasses
  - `CiteabilityFirewall.is_citeable(claim: VersionedClaim, *, for_client_deliverable: bool) -> bool` and `CiteabilityFirewall.filter_claims(claims: Sequence[VersionedClaim], *, for_client_deliverable: bool) -> list[VersionedClaim]`

- [ ] **Step 1: Write the failing test**

```python
# backend/tests/platform/knowledge/domain/test_epistemic_domain.py
from datetime import datetime, timezone
import pytest

from backend.platform.knowledge.domain.citeability import CiteabilityFirewall
from backend.platform.knowledge.domain.enums import (
    BaitsPillar,
    ClaimPerspective,
    EvolutionType,
    KnowledgeScope,
)
from backend.platform.knowledge.domain.errors import (
    CrossScopeAccessError,
    InvalidElevationCitationError,
)
from backend.platform.knowledge.domain.models import (
    EngagementCitationUri,
    KnowledgeCoordinates,
    VersionedClaim,
)


def _make_claim(
    fact_id: str,
    scope: KnowledgeScope,
    perspective: ClaimPerspective,
    text: str = "Core ledger RPO must be <= 15 minutes.",
) -> VersionedClaim:
    now = datetime(2026, 10, 27, 14, 0, tzinfo=timezone.utc)
    return VersionedClaim(
        fact_id=fact_id,
        concept_id="concept-core-ledger",
        pillar=BaitsPillar.TECHNOLOGY,
        subject_slug="core-ledger",
        attribute_key="rpo_minutes",
        claim_text=text,
        verbatim_quote="Our RPO on the core ledger cannot exceed 15 minutes.",
        source_uri="raw/2026-10-27-arch-workshop.md",
        scope=scope,
        perspective=perspective,
        speaker_role="Principal Architect",
        version=1,
        observed_at=now,
        ingested_at=now,
        evolution_type=EvolutionType.INITIAL,
        lineage_parent_id=None,
        is_active=True,
    )


def test_citeability_firewall_blocks_internal_and_personal_claims_from_deliverable() -> None:
    client_claim = _make_claim("^f-001", KnowledgeScope.CLIENT, ClaimPerspective.CLIENT_STATED)
    ref_claim = _make_claim("^f-002", KnowledgeScope.REFERENCE, ClaimPerspective.WORK_RESEARCH)
    internal_claim = _make_claim("^f-003", KnowledgeScope.INTERNAL, ClaimPerspective.INTERNAL_TEAM)
    personal_take = _make_claim("^f-004", KnowledgeScope.CLIENT, ClaimPerspective.PERSONAL_TAKE)
    personal_rd = _make_claim("^f-005", KnowledgeScope.REFERENCE, ClaimPerspective.PERSONAL_RD)

    all_claims = [client_claim, ref_claim, internal_claim, personal_take, personal_rd]

    # Internal working synthesis allows everything except personal_rd (unless in personal mode)
    internal_allowed = CiteabilityFirewall.filter_claims(
        all_claims, for_client_deliverable=False
    )
    assert [c.fact_id for c in internal_allowed] == ["^f-001", "^f-002", "^f-003", "^f-004"]

    # Client-facing deliverable strictly blocks internal scope and personal_take / personal_rd perspectives
    deliverable_allowed = CiteabilityFirewall.filter_claims(
        all_claims, for_client_deliverable=True
    )
    assert [c.fact_id for c in deliverable_allowed] == ["^f-001", "^f-002"]


def test_knowledge_coordinates_reject_cross_engagement_access() -> None:
    coords = KnowledgeCoordinates.for_engagement("2026_acme")
    assert coords.schema_name == "eng_2026_acme"
    coords.assert_can_read("eng_2026_acme")

    with pytest.raises(CrossScopeAccessError, match="eng_2026_globex"):
        coords.assert_can_read("eng_2026_globex")

    with pytest.raises(CrossScopeAccessError, match="universal"):
        coords.assert_can_read("universal")


def test_engagement_citation_uri_parses_and_rejects_malformed_anchors() -> None:
    parsed = EngagementCitationUri.parse("engagement://eng_2026_acme#^retro-07")
    assert parsed.engagement_slug == "eng_2026_acme"
    assert parsed.anchor_id == "^retro-07"
    assert parsed.canonical_uri == "engagement://eng_2026_acme#^retro-07"

    with pytest.raises(InvalidElevationCitationError):
        EngagementCitationUri.parse("https://example.com/no-engagement-id")

    with pytest.raises(InvalidElevationCitationError):
        EngagementCitationUri.parse("engagement://eng_2026_acme")
```

- [ ] **Step 2: Run test to verify it fails**

Run: `uv run pytest backend/tests/platform/knowledge/domain/test_epistemic_domain.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'backend.platform.knowledge'`

- [ ] **Step 3: Implement the epistemic domain enums, models, errors, and `CiteabilityFirewall`**

Create `backend/platform/knowledge/domain/enums.py`, `errors.py`, `models.py`, and `citeability.py` using only the Python standard library. In `KnowledgeCoordinates.for_engagement(slug: str)`, normalize `slug` to lowercase `[a-z0-9_]+` and prefix with `eng_` if not already prefixed; raise `ValueError` on invalid characters. In `EngagementCitationUri.parse(raw_uri: str)`, validate against `^engagement://(eng_[a-z0-9_]+)#(\^[a-zA-Z0-9_-]+)$` and raise `InvalidElevationCitationError` on mismatch. In `CiteabilityFirewall.is_citeable()`, reject `KnowledgeScope.INTERNAL` and `ClaimPerspective in {INTERNAL_TEAM, PERSONAL_TAKE, PERSONAL_RD}` when `for_client_deliverable=True`, and reject `ClaimPerspective.PERSONAL_RD` when `for_client_deliverable=False` unless `allow_personal_rd=True`.

- [ ] **Step 4: Run test to verify it passes**

Run: `uv run pytest backend/tests/platform/knowledge/domain/test_epistemic_domain.py -v`
Expected: PASS (3 passed)

- [ ] **Step 5: Commit**

```bash
git add backend/platform/knowledge/domain/ backend/tests/platform/knowledge/domain/test_epistemic_domain.py
git commit -m "feat(knowledge): add epistemic domain models, bitemporal claims, and citeability firewall"
```

---

### Task 2: `eng_<slug>.*`, `universal.*`, and `personal.*` Schema Provisioning & Engagement Master Data Repositories + Port-Contract Tests

Every client engagement (`eng_<slug>.*`) is a self-contained empirical dataset with its own `pgvector` tables and its own seven **Engagement Master Data** tables (`master_stakeholders`, `master_systems`, `master_capabilities`, `master_decisions`, `master_metrics`, `master_deliverables`, `master_raid_items`), plus relational `entity_fact_links` tying each master record to the `^f-id` claims that substantiate it.

This task defines the `KnowledgeSchemaProvisionerPort` and `EngagementMasterDataPort` interfaces, provides the deterministic DDL builder that stamps out `eng_<slug>.*`, `universal.*`, and `personal.*` schemas (including `okf_sync_ledger`, `okf_concepts`, `okf_pillar_sections`, `okf_facts`, `okf_citations`, `conflict_clusters`, `elevated_learnings`, and the 7 `master_*` tables), and ships contract tests verifying both the in-memory Fake and PostgreSQL adapters.

**Files:**
- Create: `backend/platform/knowledge/domain/master_data.py`
- Create: `backend/platform/knowledge/ports/__init__.py`
- Create: `backend/platform/knowledge/ports/master_data_port.py`
- Create: `backend/platform/knowledge/adapters/__init__.py`
- Create: `backend/platform/knowledge/adapters/schema_ddl.py`
- Create: `backend/platform/knowledge/adapters/memory_master_data.py`
- Create: `backend/platform/knowledge/adapters/postgres_master_data.py`
- Test: `backend/tests/platform/knowledge/test_master_data_contract.py`

**Interfaces:**
- Consumes: `KnowledgeCoordinates`, `BaitsPillar`, `CrossScopeAccessError` from Task 1
- Produces:
  - Domain dataclasses in `domain/master_data.py`:
    - `StakeholderRecord(stakeholder_id: str, name: str, title: str, organization: Literal['client', 'internal'], department: str, influence_level: int, authority_weight: float, linked_fact_ids: tuple[str, ...])`
    - `SystemRecord(system_id: str, name: str, category: str, owner_stakeholder_id: str | None, lifecycle_state: Literal['legacy', 'target', 'transitional'], baits_pillars: tuple[BaitsPillar, ...], linked_fact_ids: tuple[str, ...])`
    - `CapabilityRecord(capability_id: str, name: str, bvf_level: Literal['L1', 'L2', 'L3'], current_maturity: int, target_maturity: int, supporting_system_ids: tuple[str, ...], linked_fact_ids: tuple[str, ...])`
    - `ArchitectureDecisionRecord(decision_id: str, title: str, status: Literal['proposed', 'accepted', 'superseded', 'contested'], decided_by_stakeholder_ids: tuple[str, ...], impacted_system_ids: tuple[str, ...], impacted_capability_ids: tuple[str, ...], rationale_fact_ids: tuple[str, ...])`
    - `EngagementMetricRecord(metric_id: str, name: str, category: Literal['kpi', 'sla', 'nfr', 'financial'], baseline_value: str, target_value: str, unit: str, measured_system_id: str | None, source_fact_id: str)`
    - `DeliverableRecord(deliverable_id: str, title: str, sow_milestone: str, status: Literal['draft', 'in_review', 'approved', 'published'], cited_fact_ids: tuple[str, ...], depended_decision_ids: tuple[str, ...])`
    - `RaidItemRecord(raid_id: str, category: Literal['Risk', 'Assumption', 'Issue', 'Dependency'], severity: Literal['low', 'medium', 'high', 'critical'], owner_stakeholder_id: str | None, mitigation_plan: str, impacted_deliverable_ids: tuple[str, ...], trigger_conflict_id: str | None, linked_fact_ids: tuple[str, ...])`
    - `MasterDataDependencyGraph(decisions_by_fact: dict[str, list[ArchitectureDecisionRecord]], systems_by_fact: dict[str, list[SystemRecord]], deliverables_by_decision: dict[str, list[DeliverableRecord]], raid_by_fact: dict[str, list[RaidItemRecord]])`
  - `EngagementMasterDataPort(ABC)` in `ports/master_data_port.py`:
    - `async def provision_schema(self, coords: KnowledgeCoordinates) -> None`
    - `async def upsert_stakeholder(self, coords: KnowledgeCoordinates, record: StakeholderRecord) -> None`
    - `async def get_stakeholder(self, coords: KnowledgeCoordinates, stakeholder_id: str) -> StakeholderRecord | None`
    - `async def upsert_system(self, coords: KnowledgeCoordinates, record: SystemRecord) -> None`
    - `async def upsert_capability(self, coords: KnowledgeCoordinates, record: CapabilityRecord) -> None`
    - `async def upsert_decision(self, coords: KnowledgeCoordinates, record: ArchitectureDecisionRecord) -> None`
    - `async def upsert_metric(self, coords: KnowledgeCoordinates, record: EngagementMetricRecord) -> None`
    - `async def upsert_deliverable(self, coords: KnowledgeCoordinates, record: DeliverableRecord) -> None`
    - `async def upsert_raid_item(self, coords: KnowledgeCoordinates, record: RaidItemRecord) -> None`
    - `async def get_dependents_for_facts(self, coords: KnowledgeCoordinates, fact_ids: Sequence[str]) -> MasterDataDependencyGraph`
  - `render_knowledge_schema_ddl(schema_name: str) -> str` in `adapters/schema_ddl.py`

- [ ] **Step 1: Write the failing test**

```python
# backend/tests/platform/knowledge/test_master_data_contract.py
import pytest

from backend.platform.knowledge.adapters.memory_master_data import InMemoryEngagementMasterDataAdapter
from backend.platform.knowledge.adapters.schema_ddl import render_knowledge_schema_ddl
from backend.platform.knowledge.domain.enums import BaitsPillar
from backend.platform.knowledge.domain.master_data import (
    ArchitectureDecisionRecord,
    DeliverableRecord,
    RaidItemRecord,
    StakeholderRecord,
    SystemRecord,
)
from backend.platform.knowledge.domain.models import KnowledgeCoordinates


def test_render_knowledge_schema_ddl_includes_pgvector_hnsw_and_master_tables() -> None:
    ddl = render_knowledge_schema_ddl("eng_2026_acme")
    assert 'CREATE SCHEMA IF NOT EXISTS "eng_2026_acme"' in ddl
    assert '"eng_2026_acme".okf_sync_ledger' in ddl
    assert '"eng_2026_acme".okf_facts' in ddl
    assert '"eng_2026_acme".okf_pillar_sections' in ddl
    assert "USING hnsw (embedding vector_cosine_ops)" in ddl
    for table in (
        "master_stakeholders",
        "master_systems",
        "master_capabilities",
        "master_decisions",
        "master_metrics",
        "master_deliverables",
        "master_raid_items",
        "entity_fact_links",
    ):
        assert f'"eng_2026_acme".{table}' in ddl


@pytest.mark.asyncio
async def test_master_data_port_contract_tracks_fact_to_decision_and_deliverable_blast_radius() -> None:
    adapter = InMemoryEngagementMasterDataAdapter()
    coords = KnowledgeCoordinates.for_engagement("2026_acme")
    await adapter.provision_schema(coords)

    await adapter.upsert_stakeholder(
        coords,
        StakeholderRecord(
            stakeholder_id="stk-chief-arch",
            name="Elena Rostova",
            title="Chief Architect",
            organization="client",
            department="Enterprise Architecture",
            influence_level=5,
            authority_weight=0.95,
            linked_fact_ids=("^f-010",),
        ),
    )
    await adapter.upsert_system(
        coords,
        SystemRecord(
            system_id="sys-core-ledger",
            name="Core Retail Ledger",
            category="core-banking",
            owner_stakeholder_id="stk-chief-arch",
            lifecycle_state="legacy",
            baits_pillars=(BaitsPillar.TECHNOLOGY, BaitsPillar.SECURITY),
            linked_fact_ids=("^f-010",),
        ),
    )
    await adapter.upsert_decision(
        coords,
        ArchitectureDecisionRecord(
            decision_id="ADR-002",
            title="Keep Core Ledger On-Premises in Phase 1",
            status="accepted",
            decided_by_stakeholder_ids=("stk-chief-arch",),
            impacted_system_ids=("sys-core-ledger",),
            impacted_capability_ids=(),
            rationale_fact_ids=("^f-010",),
        ),
    )
    await adapter.upsert_deliverable(
        coords,
        DeliverableRecord(
            deliverable_id="deliv-target-arch-v1",
            title="Phase 1 Target Architecture Blueprint",
            sow_milestone="M2",
            status="approved",
            cited_fact_ids=("^f-010",),
            depended_decision_ids=("ADR-002",),
        ),
    )
    await adapter.upsert_raid_item(
        coords,
        RaidItemRecord(
            raid_id="RAID-A04",
            category="Assumption",
            severity="high",
            owner_stakeholder_id="stk-chief-arch",
            mitigation_plan="Validate latency budget over DirectConnect",
            impacted_deliverable_ids=("deliv-target-arch-v1",),
            trigger_conflict_id=None,
            linked_fact_ids=("^f-010",),
        ),
    )

    graph = await adapter.get_dependents_for_facts(coords, ["^f-010"])
    assert [d.decision_id for d in graph.decisions_by_fact["^f-010"]] == ["ADR-002"]
    assert [s.system_id for s in graph.systems_by_fact["^f-010"]] == ["sys-core-ledger"]
    assert [
        deliv.deliverable_id for deliv in graph.deliverables_by_decision["ADR-002"]
    ] == ["deliv-target-arch-v1"]
    assert [r.raid_id for r in graph.raid_by_fact["^f-010"]] == ["RAID-A04"]
```

- [ ] **Step 2: Run test to verify it fails**

Run: `uv run pytest backend/tests/platform/knowledge/test_master_data_contract.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'backend.platform.knowledge.domain.master_data'`

- [ ] **Step 3: Implement `master_data.py`, `EngagementMasterDataPort`, `render_knowledge_schema_ddl`, and the adapters**

Implement the 7 Master Data records and `MasterDataDependencyGraph` in `backend/platform/knowledge/domain/master_data.py`. In `backend/platform/knowledge/adapters/schema_ddl.py`, validate `schema_name` against `^(universal|personal|eng_[a-z0-9_]+)$` before interpolating identifiers, and define the SQL tables for `okf_sync_ledger`, `okf_concepts`, `okf_pillar_sections` (`embedding vector(768)` + HNSW index + `search_tsv tsvector` GIN index), `okf_facts` (`embedding vector(768)` + HNSW index + `search_tsv tsvector` GIN index + bitemporal/scope/perspective columns), `okf_citations`, `conflict_clusters`, `elevated_learnings`, the 7 `master_*` tables, and `entity_fact_links`. Implement `InMemoryEngagementMasterDataAdapter` and `PostgresEngagementMasterDataAdapter` (using an injected `asyncpg.Pool`).

- [ ] **Step 4: Run test to verify it passes**

Run: `uv run pytest backend/tests/platform/knowledge/test_master_data_contract.py -v`
Expected: PASS (2 passed)

- [ ] **Step 5: Commit**

```bash
git add backend/platform/knowledge/domain/master_data.py backend/platform/knowledge/ports/ backend/platform/knowledge/adapters/ backend/tests/platform/knowledge/test_master_data_contract.py
git commit -m "feat(knowledge): add schema DDL provisioner and engagement master data repositories"
```

---

### Task 3: Canonical `okf-wiki/` Markdown Parser & Durable `OKFIndexer` (`okf_sync_ledger` SHA-256 Hashing + Dual-Granularity Projection)

In `dsa-companion-config`, vector persistence tapped a temporary `/tmp` file and silently exited `0` when Postgres hiccuped, leaving `okf-wiki/` and `pgvector` out of sync. In Chotu, `okf-wiki/` Markdown on disk is the single canonical write-master, and `OKFIndexer` projects it deterministically into `pgvector` across two granularities: **atomic claims (`^f-id`)** and **BAITS pillar sections (`## <Pillar>`)**.

This task builds the pure-Python `OkfMarkdownParser` (which extracts frontmatter, `## <Pillar>` section blocks, atomic `^f-id` claims with inline/sub-bullet metadata, and `[[concept#^f-id]]` citation edges) and the `OKFIndexer` service. Every file indexing pass computes the file's SHA-256 digest in `<schema>.okf_sync_ledger`: unchanged files are skipped in $O(1)$, modified files embed both their `## <Pillar>` sections and their `^f-id` claims, and any transient embedding failure transitions the ledger row to `RETRY_BACKOFF` with an incremented `attempt_count` so no fact is ever silently dropped.

**Files:**
- Create: `backend/platform/knowledge/domain/okf_parser.py`
- Create: `backend/platform/knowledge/ports/indexer_port.py`
- Create: `backend/platform/knowledge/adapters/okf_indexer.py`
- Test: `backend/tests/platform/knowledge/test_okf_indexer.py`

**Interfaces:**
- Consumes: `VersionedClaim`, `PillarSectionBlock`, `KnowledgeCoordinates`, `BaitsPillar`, `KnowledgeScope`, `ClaimPerspective`, `EvolutionType` from Task 1
- Produces:
  - `ParsedOkfDocument(file_path: str, sha256_hash: str, concept_id: str, title: str, doc_kind: Literal['concept', 'synthesis', 'retrospective'], pillar_sections: tuple[PillarSectionBlock, ...], claims: tuple[VersionedClaim, ...], citation_edges: tuple[CitationEdge, ...])`
  - `SyncLedgerEntry(file_path: str, sha256_hash: str, status: Literal['PENDING_INDEX', 'INDEXED', 'RETRY_BACKOFF', 'DEAD_LETTER'], attempt_count: int, last_error: str | None)`
  - `EmbeddingPort(ABC)` in `ports/indexer_port.py`: `async def embed_texts(self, texts: Sequence[str]) -> list[list[float]]`
  - `OKFIndexerPort(ABC)` in `ports/indexer_port.py`:
    - `async def index_markdown_file(self, coords: KnowledgeCoordinates, relative_path: str, markdown_content: str) -> SyncLedgerEntry`
    - `async def reindex_workspace(self, coords: KnowledgeCoordinates, files: Mapping[str, str]) -> list[SyncLedgerEntry]`
    - `async def get_sync_entry(self, coords: KnowledgeCoordinates, relative_path: str) -> SyncLedgerEntry | None`
  - `parse_okf_markdown(relative_path: str, content: str) -> ParsedOkfDocument` in `domain/okf_parser.py`

- [ ] **Step 1: Write the failing test**

```python
# backend/tests/platform/knowledge/test_okf_indexer.py
import pytest

from backend.platform.knowledge.adapters.okf_indexer import InMemoryOKFStore, OKFIndexer
from backend.platform.knowledge.domain.enums import BaitsPillar, ClaimPerspective, KnowledgeScope
from backend.platform.knowledge.domain.models import KnowledgeCoordinates
from backend.platform.knowledge.domain.okf_parser import parse_okf_markdown
from backend.platform.knowledge.ports.indexer_port import EmbeddingPort


SAMPLE_CONCEPT_MD = """---
concept_id: concept-core-ledger
title: Core Retail Ledger
doc_kind: concept
---

## Business
Retail banking requires 24/7 instant settlement across regions. ^f-101
- scope: client
- perspective: client_stated
- subject: core-ledger
- attribute: settlement_window
- speaker_role: VP Retail Banking
- observed_at: 2026-10-20T09:00:00Z
- source_uri: raw/2026-10-20-kickoff.md
- quote: "We need 24/7 instant settlement across all retail regions."

## Technology
The legacy mainframe batch window locks the ledger for 4 hours nightly, citing [[concept-batch-scheduler#^f-050]]. ^f-102
- scope: internal
- perspective: work_research
- subject: core-ledger
- attribute: batch_lock_hours
- speaker_role: Lead Technologist
- observed_at: 2026-10-22T15:30:00Z
- source_uri: raw/2026-10-22-tech-dive.md
- quote: "Batch processing locks ledger writes between 01:00 and 05:00 UTC."
"""


class FakeDeterministicEmbeddingAdapter(EmbeddingPort):
    def __init__(self, should_fail: bool = False) -> None:
        self.should_fail = should_fail
        self.call_count = 0

    async def embed_texts(self, texts: list[str] | tuple[str, ...]) -> list[list[float]]:
        self.call_count += 1
        if self.should_fail:
            raise RuntimeError("Embedding sidecar timeout")
        return [[float(len(t) % 10) / 10.0] * 768 for t in texts]


def test_parse_okf_markdown_extracts_dual_granularity_sections_facts_and_citations() -> None:
    parsed = parse_okf_markdown("okf-wiki/systems/concept-core-ledger.md", SAMPLE_CONCEPT_MD)
    assert parsed.concept_id == "concept-core-ledger"
    assert [s.pillar for s in parsed.pillar_sections] == [
        BaitsPillar.BUSINESS,
        BaitsPillar.TECHNOLOGY,
    ]
    assert [c.fact_id for c in parsed.claims] == ["^f-101", "^f-102"]
    assert parsed.claims[0].scope == KnowledgeScope.CLIENT
    assert parsed.claims[0].perspective == ClaimPerspective.CLIENT_STATED
    assert parsed.claims[1].scope == KnowledgeScope.INTERNAL
    assert parsed.claims[1].perspective == ClaimPerspective.WORK_RESEARCH
    assert len(parsed.citation_edges) == 1
    assert parsed.citation_edges[0].source_fact_id == "^f-102"
    assert parsed.citation_edges[0].target_concept_id == "concept-batch-scheduler"
    assert parsed.citation_edges[0].target_fact_id == "^f-050"


@pytest.mark.asyncio
async def test_okf_indexer_records_retry_backoff_on_embedding_failure_without_silent_skip() -> None:
    store = InMemoryOKFStore()
    failing_embedder = FakeDeterministicEmbeddingAdapter(should_fail=True)
    indexer = OKFIndexer(store=store, embedder=failing_embedder)
    coords = KnowledgeCoordinates.for_engagement("2026_acme")

    entry = await indexer.index_markdown_file(
        coords, "okf-wiki/systems/concept-core-ledger.md", SAMPLE_CONCEPT_MD
    )
    assert entry.status == "RETRY_BACKOFF"
    assert entry.attempt_count == 1
    assert "Embedding sidecar timeout" in (entry.last_error or "")
    assert len(store.get_facts(coords)) == 0

    # When embedding recovers, re-indexing succeeds and skips unchanged repeat calls via SHA-256
    healthy_embedder = FakeDeterministicEmbeddingAdapter(should_fail=False)
    indexer_recovered = OKFIndexer(store=store, embedder=healthy_embedder)
    entry_ok = await indexer_recovered.index_markdown_file(
        coords, "okf-wiki/systems/concept-core-ledger.md", SAMPLE_CONCEPT_MD
    )
    assert entry_ok.status == "INDEXED"
    assert len(store.get_facts(coords)) == 2
    assert len(store.get_pillar_sections(coords)) == 2
    assert healthy_embedder.call_count == 1

    # Idempotent repeat call with identical SHA-256 does not re-invoke embedder
    await indexer_recovered.index_markdown_file(
        coords, "okf-wiki/systems/concept-core-ledger.md", SAMPLE_CONCEPT_MD
    )
    assert healthy_embedder.call_count == 1
```

- [ ] **Step 2: Run test to verify it fails**

Run: `uv run pytest backend/tests/platform/knowledge/test_okf_indexer.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'backend.platform.knowledge.domain.okf_parser'`

- [ ] **Step 3: Implement `parse_okf_markdown` and `OKFIndexer`**

In `backend/platform/knowledge/domain/okf_parser.py`, implement `parse_okf_markdown(relative_path: str, content: str) -> ParsedOkfDocument`:
- Compute `hashlib.sha256(content.encode("utf-8")).hexdigest()`.
- Parse YAML frontmatter between leading `---` markers.
- Split `## <Pillar>` headings matching `BaitsPillar` values into `PillarSectionBlock(section_id=f"{concept_id}#{pillar.value.lower()}", concept_id=concept_id, pillar=pillar, section_markdown=..., doc_kind=doc_kind)`.
- Extract claim paragraphs ending with `^(f-[a-zA-Z0-9_-]+|retro-[a-zA-Z0-9_-]+|u-f[a-zA-Z0-9_-]+)` followed by metadata bullets (`- scope:`, `- perspective:`, `- subject:`, `- attribute:`, `- speaker_role:`, `- observed_at:`, `- source_uri:`, `- quote:`, `- version:`, `- lineage_parent_id:`).
- Extract `[[<target_concept>#^(<target_fact>)]]` regex matches into `CitationEdge` objects.

In `backend/platform/knowledge/adapters/okf_indexer.py`, implement `OKFIndexer` so it checks `store.get_sync_entry(coords, relative_path)`: if `status == 'INDEXED'` and `sha256_hash` matches, return existing entry immediately. Otherwise call `embedder.embed_texts()` once with all pillar section texts + claim texts, persist to `store`, and update `SyncLedgerEntry(status='INDEXED')`. Catch exceptions from `embedder`, increment `attempt_count` (`status='DEAD_LETTER'` if `attempt_count >= 5` else `'RETRY_BACKOFF'`), record `last_error`, and return the ledger entry without losing the sync state.

- [ ] **Step 4: Run test to verify it passes**

Run: `uv run pytest backend/tests/platform/knowledge/test_okf_indexer.py -v`
Expected: PASS (2 passed)

- [ ] **Step 5: Commit**

```bash
git add backend/platform/knowledge/domain/okf_parser.py backend/platform/knowledge/ports/indexer_port.py backend/platform/knowledge/adapters/okf_indexer.py backend/tests/platform/knowledge/test_okf_indexer.py
git commit -m "feat(knowledge): add canonical okf-wiki parser and durable SHA-256 OKFIndexer"
```

---

### Task 4: Small-to-Big Knowledge Librarian (`KnowledgeLibrarianPort` — Hybrid `pgvector` + FTS RRF Search, Parent `## <Pillar>` + Synthesis Hydration & `RetrievalTrace` Logging)

If a subagent queries for `"ledger nightly lock"`, returning just the isolated sentence `^f-102` strips away surrounding architectural context, while returning the entire multi-pillar `concept-core-ledger.md` file floods the prompt with unrelated Business or Security notes.

Workstream 2.2's **Small-to-Big Librarian** solves this in two stages: first, it matches fine-grained atomic claims (`okf_facts`) and pillar sections (`okf_pillar_sections`) using Reciprocal Rank Fusion (RRF) over cosine similarity and BM25 lexical overlap, filtered by `CiteabilityFirewall`. Second, it groups the winning facts by their parent `(concept_id, pillar)` section and hydrates the **full parent `## <Pillar>` block** alongside the corresponding `okf-wiki/synthesis/baits-<pillar>.md` synthesis block and verbatim citations, logging a `RetrievalTrace` so Chotu can measure which hydrated facts the subagent actually cites.

**Files:**
- Create: `backend/platform/knowledge/domain/retrieval.py`
- Create: `backend/platform/knowledge/ports/librarian_port.py`
- Create: `backend/platform/knowledge/adapters/small_to_big_librarian.py`
- Test: `backend/tests/platform/knowledge/test_small_to_big_librarian.py`

**Interfaces:**
- Consumes: `KnowledgeCoordinates`, `BaitsPillar`, `KnowledgeScope`, `ClaimPerspective`, `VersionedClaim`, `PillarSectionBlock`, `CiteabilityFirewall` from Task 1; `InMemoryOKFStore`, `EmbeddingPort` from Task 3
- Produces:
  - `LibrarianQuery(query_text: str, pillar_filter: BaitsPillar | None = None, allowed_scopes: tuple[KnowledgeScope, ...] | None = None, for_client_deliverable: bool = False, top_k: int = 5)`
  - `HydratedPillarContext(concept_id: str, pillar: BaitsPillar, parent_section_markdown: str, synthesis_section_markdown: str | None, matched_claims: tuple[VersionedClaim, ...], rrf_score: float)`
  - `ContextPacket(coords: KnowledgeCoordinates, query_text: str, trace_id: str, hydrated_contexts: tuple[HydratedPillarContext, ...], all_cited_fact_ids: tuple[str, ...])`
  - `RetrievalTraceRecord(trace_id: str, schema_name: str, query_text: str, retrieved_fact_ids: tuple[str, ...], cited_in_output_fact_ids: tuple[str, ...], precision_hit_rate: float)`
  - `KnowledgeLibrarianPort(ABC)` in `ports/librarian_port.py`:
    - `async def retrieve(self, coords: KnowledgeCoordinates, query: LibrarianQuery) -> ContextPacket`
    - `async def record_citation_feedback(self, coords: KnowledgeCoordinates, trace_id: str, cited_fact_ids: Sequence[str]) -> RetrievalTraceRecord`

- [ ] **Step 1: Write the failing test**

```python
# backend/tests/platform/knowledge/test_small_to_big_librarian.py
import pytest

from backend.platform.knowledge.adapters.okf_indexer import InMemoryOKFStore, OKFIndexer
from backend.platform.knowledge.adapters.small_to_big_librarian import SmallToBigLibrarian
from backend.platform.knowledge.domain.enums import BaitsPillar
from backend.platform.knowledge.domain.models import KnowledgeCoordinates
from backend.platform.knowledge.domain.retrieval import LibrarianQuery
from backend.tests.platform.knowledge.test_okf_indexer import (
    SAMPLE_CONCEPT_MD,
    FakeDeterministicEmbeddingAdapter,
)

SAMPLE_SYNTHESIS_TECH_MD = """---
concept_id: baits-technology
title: Technology Pillar Synthesis
doc_kind: synthesis
---

## Technology
Current mainframe batch lock (`^f-102`) prevents 24/7 settlement until CDC streaming is introduced.
"""


@pytest.mark.asyncio
async def test_small_to_big_hydrates_parent_pillar_section_and_synthesis() -> None:
    store = InMemoryOKFStore()
    embedder = FakeDeterministicEmbeddingAdapter()
    indexer = OKFIndexer(store=store, embedder=embedder)
    coords = KnowledgeCoordinates.for_engagement("2026_acme")

    await indexer.index_markdown_file(
        coords, "okf-wiki/systems/concept-core-ledger.md", SAMPLE_CONCEPT_MD
    )
    await indexer.index_markdown_file(
        coords, "okf-wiki/synthesis/baits-technology.md", SAMPLE_SYNTHESIS_TECH_MD
    )

    librarian = SmallToBigLibrarian(store=store, embedder=embedder)
    packet = await librarian.retrieve(
        coords,
        LibrarianQuery(
            query_text="mainframe batch window locks ledger nightly",
            pillar_filter=BaitsPillar.TECHNOLOGY,
            for_client_deliverable=False,
        ),
    )

    assert len(packet.hydrated_contexts) == 1
    ctx = packet.hydrated_contexts[0]
    assert ctx.concept_id == "concept-core-ledger"
    assert ctx.pillar == BaitsPillar.TECHNOLOGY
    assert [c.fact_id for c in ctx.matched_claims] == ["^f-102"]
    assert "The legacy mainframe batch window locks the ledger" in ctx.parent_section_markdown
    assert ctx.synthesis_section_markdown is not None
    assert "CDC streaming is introduced" in ctx.synthesis_section_markdown

    trace = await librarian.record_citation_feedback(
        coords, packet.trace_id, cited_fact_ids=["^f-102"]
    )
    assert trace.precision_hit_rate == 1.0


@pytest.mark.asyncio
async def test_librarian_deliverable_mode_strips_internal_facts_from_hydrated_section() -> None:
    store = InMemoryOKFStore()
    embedder = FakeDeterministicEmbeddingAdapter()
    indexer = OKFIndexer(store=store, embedder=embedder)
    coords = KnowledgeCoordinates.for_engagement("2026_acme")
    await indexer.index_markdown_file(
        coords, "okf-wiki/systems/concept-core-ledger.md", SAMPLE_CONCEPT_MD
    )

    librarian = SmallToBigLibrarian(store=store, embedder=embedder)
    # ^f-102 is stamped scope: internal, so deliverable mode must block it
    deliverable_packet = await librarian.retrieve(
        coords,
        LibrarianQuery(
            query_text="mainframe batch window locks ledger",
            for_client_deliverable=True,
        ),
    )
    assert "^f-102" not in deliverable_packet.all_cited_fact_ids
    assert "^f-101" in deliverable_packet.all_cited_fact_ids
```

- [ ] **Step 2: Run test to verify it fails**

Run: `uv run pytest backend/tests/platform/knowledge/test_small_to_big_librarian.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'backend.platform.knowledge.domain.retrieval'`

- [ ] **Step 3: Implement `SmallToBigLibrarian` and RRF hydration**

In `backend/platform/knowledge/adapters/small_to_big_librarian.py`, implement `SmallToBigLibrarian`:
1. Filter candidate claims in `coords` through `CiteabilityFirewall.filter_claims(..., for_client_deliverable=query.for_client_deliverable)` and optional `pillar_filter` / `allowed_scopes`.
2. Score each candidate claim and its parent `PillarSectionBlock` using Reciprocal Rank Fusion ($\text{RRF}(d) = \frac{1}{60 + r_{\text{vec}}(d)} + \frac{1}{60 + r_{\text{fts}}(d)}$) combining cosine similarity against `embedder.embed_texts([query.query_text])[0]` and lexical token overlap (`tsvector` equivalent).
3. Group winning claims by `(concept_id, pillar)`, fetch the parent `PillarSectionBlock` (stripping any lines anchored by non-citeable `^f-id` claims when `for_client_deliverable=True`), attach any matching `doc_kind == 'synthesis'` section for that `pillar`, and store a `RetrievalTraceRecord` keyed by `trace_id`.

- [ ] **Step 4: Run test to verify it passes**

Run: `uv run pytest backend/tests/platform/knowledge/test_small_to_big_librarian.py -v`
Expected: PASS (2 passed)

- [ ] **Step 5: Commit**

```bash
git add backend/platform/knowledge/domain/retrieval.py backend/platform/knowledge/ports/librarian_port.py backend/platform/knowledge/adapters/small_to_big_librarian.py backend/tests/platform/knowledge/test_small_to_big_librarian.py
git commit -m "feat(knowledge): implement Small-to-Big Librarian with RRF hydration and RetrievalTrace"
```

---

### Task 5: Disjoint Universal Knowledge Bridge (`UniversalKnowledgeBridgePort` — Closed-World Local-First Check + `Discover -> Select -> Ingest`)

Chotu never dumps the Universal Knowledge Vault into every engagement turn. Subagents query their active `eng_<slug>.*` scope first, and only when local recall misses (or when Truth Finder needs an industry benchmark to weigh a conflict) does Chotu query the disjoint `universal.*` service. Crucially, once candidate knowledge is discovered and selected from `universal.*`, it is **physically ingested into the engagement's own `okf-wiki/reference/` and `eng_<slug>.*` index** (stamped `scope: reference` or `scope: prior-project`) so the engagement dataset stays 100% self-contained.

This task implements `UniversalKnowledgeBridgePort` and `DisjointUniversalBridgeAdapter`, enforcing that the bridge only reads from `universal` (never from another `eng_*` schema) and automatically re-indexes newly ingested reference facts into the target engagement scope.

**Files:**
- Create: `backend/platform/knowledge/ports/universal_bridge_port.py`
- Create: `backend/platform/knowledge/adapters/universal_bridge.py`
- Test: `backend/tests/platform/knowledge/test_universal_bridge.py`

**Interfaces:**
- Consumes: `KnowledgeCoordinates`, `KnowledgeScope`, `ClaimPerspective`, `CrossScopeAccessError` from Task 1; `OKFIndexerPort` from Task 3; `KnowledgeLibrarianPort`, `LibrarianQuery`, `ContextPacket` from Task 4
- Produces:
  - `UniversalCandidate(fact_id: str, concept_id: str, pillar: BaitsPillar, claim_text: str, verbatim_quote: str, universal_source_uri: str, target_scope: Literal[KnowledgeScope.REFERENCE, KnowledgeScope.PRIOR_PROJECT], correlation_engagement_id: str | None)`
  - `IngestedReferenceReceipt(target_coords: KnowledgeCoordinates, written_markdown_path: str, ingested_fact_ids: tuple[str, ...], sync_status: str)`
  - `UniversalKnowledgeBridgePort(ABC)` in `ports/universal_bridge_port.py`:
    - `async def discover_candidates(self, query_text: str, *, pillar_filter: BaitsPillar | None = None, top_k: int = 5) -> list[UniversalCandidate]`
    - `async def select_and_ingest_into_engagement(self, target_coords: KnowledgeCoordinates, candidates: Sequence[UniversalCandidate], selection_rationale: str) -> IngestedReferenceReceipt`

- [ ] **Step 1: Write the failing test**

```python
# backend/tests/platform/knowledge/test_universal_bridge.py
import pytest

from backend.platform.knowledge.adapters.okf_indexer import InMemoryOKFStore, OKFIndexer
from backend.platform.knowledge.adapters.small_to_big_librarian import SmallToBigLibrarian
from backend.platform.knowledge.adapters.universal_bridge import DisjointUniversalBridgeAdapter
from backend.platform.knowledge.domain.enums import BaitsPillar, KnowledgeScope
from backend.platform.knowledge.domain.errors import CrossScopeAccessError
from backend.platform.knowledge.domain.models import KnowledgeCoordinates
from backend.platform.knowledge.domain.retrieval import LibrarianQuery
from backend.tests.platform.knowledge.test_okf_indexer import FakeDeterministicEmbeddingAdapter

UNIVERSAL_STANDARD_MD = """---
concept_id: concept-zero-trust-mesh-benchmark
title: Zero Trust Service Mesh Latency Benchmark
doc_kind: concept
---

## Technology
Mutual TLS via Envoy sidecars adds 1.8ms to 2.4ms p99 latency per hop under 10k RPS financial workloads. ^u-f401
- scope: reference
- perspective: work_research
- subject: service-mesh
- attribute: p99_latency_ms
- speaker_role: Industry Benchmark
- observed_at: 2026-09-15T12:00:00Z
- source_uri: universal://standards/envoy-mtls-2026.md
- quote: "Measured p99 sidecar overhead is 1.8ms-2.4ms at 10,000 RPS."
"""


@pytest.mark.asyncio
async def test_discover_select_ingest_copies_universal_fact_into_self_contained_engagement_scope() -> None:
    store = InMemoryOKFStore()
    embedder = FakeDeterministicEmbeddingAdapter()
    indexer = OKFIndexer(store=store, embedder=embedder)
    librarian = SmallToBigLibrarian(store=store, embedder=embedder)

    universal_coords = KnowledgeCoordinates.universal()
    await indexer.index_markdown_file(
        universal_coords,
        "okf-wiki/standards/concept-zero-trust-mesh-benchmark.md",
        UNIVERSAL_STANDARD_MD,
    )

    eng_coords = KnowledgeCoordinates.for_engagement("2026_acme")
    # 1. Closed-world local check misses first
    local_packet = await librarian.retrieve(
        eng_coords, LibrarianQuery(query_text="Envoy sidecar p99 latency overhead")
    )
    assert len(local_packet.hydrated_contexts) == 0

    # 2. Discover -> Select -> Ingest from disjoint Universal Knowledge
    bridge = DisjointUniversalBridgeAdapter(
        store=store, indexer=indexer, librarian=librarian
    )
    candidates = await bridge.discover_candidates(
        "Envoy sidecar p99 latency overhead", pillar_filter=BaitsPillar.TECHNOLOGY
    )
    assert len(candidates) == 1
    assert candidates[0].fact_id == "^u-f401"

    receipt = await bridge.select_and_ingest_into_engagement(
        target_coords=eng_coords,
        candidates=candidates,
        selection_rationale="Needed to weigh client mesh latency conflict",
    )
    assert receipt.sync_status == "INDEXED"
    assert receipt.written_markdown_path.startswith("okf-wiki/reference/")

    # 3. Subsequent local engagement query resolves 100% locally with scope: reference
    after_packet = await librarian.retrieve(
        eng_coords, LibrarianQuery(query_text="Envoy sidecar p99 latency overhead")
    )
    assert len(after_packet.hydrated_contexts) == 1
    ingested_claim = after_packet.hydrated_contexts[0].matched_claims[0]
    assert ingested_claim.scope == KnowledgeScope.REFERENCE
    assert "universal://universal/concept-zero-trust-mesh-benchmark#^u-f401" in ingested_claim.source_uri


@pytest.mark.asyncio
async def test_universal_bridge_forbids_cross_engagement_source() -> None:
    store = InMemoryOKFStore()
    embedder = FakeDeterministicEmbeddingAdapter()
    indexer = OKFIndexer(store=store, embedder=embedder)
    librarian = SmallToBigLibrarian(store=store, embedder=embedder)
    bridge = DisjointUniversalBridgeAdapter(
        store=store, indexer=indexer, librarian=librarian
    )

    with pytest.raises(CrossScopeAccessError):
        await bridge.select_and_ingest_into_engagement(
            target_coords=KnowledgeCoordinates.universal(),
            candidates=[],
            selection_rationale="Cannot ingest into universal via engagement bridge",
        )
```

- [ ] **Step 2: Run test to verify it fails**

Run: `uv run pytest backend/tests/platform/knowledge/test_universal_bridge.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'backend.platform.knowledge.ports.universal_bridge_port'`

- [ ] **Step 3: Implement `DisjointUniversalBridgeAdapter`**

In `backend/platform/knowledge/adapters/universal_bridge.py`:
- `discover_candidates()` queries `librarian.retrieve(KnowledgeCoordinates.universal(), ...)` and converts matched claims into `UniversalCandidate` objects (`target_scope = KnowledgeScope.PRIOR_PROJECT` if `claim.scope == KnowledgeScope.PRIOR_PROJECT` else `KnowledgeScope.REFERENCE`).
- `select_and_ingest_into_engagement()` asserts `target_coords.schema_name.startswith("eng_")` (raising `CrossScopeAccessError` otherwise), renders a canonical `okf-wiki/reference/concept-ref-<slug>.md` document preserving each candidate's text, quote, `scope: reference | prior-project`, and provenance URI `universal://universal/<concept_id>#<fact_id>`, and calls `indexer.index_markdown_file(target_coords, ...)` so the facts are immediately searchable inside `target_coords`.

- [ ] **Step 4: Run test to verify it passes**

Run: `uv run pytest backend/tests/platform/knowledge/test_universal_bridge.py -v`
Expected: PASS (2 passed)

- [ ] **Step 5: Commit**

```bash
git add backend/platform/knowledge/ports/universal_bridge_port.py backend/platform/knowledge/adapters/universal_bridge.py backend/tests/platform/knowledge/test_universal_bridge.py
git commit -m "feat(knowledge): implement disjoint UniversalKnowledgeBridgePort with Discover-Select-Ingest"
```

---

### Task 6: Active Truth Finder + HITS Engine (`TruthFinderEngine` — Iterative Source Reliability & Claim Confidence + Hub & Authority Scoring Without Forced Auto-Overwrite)

When two claims inside an engagement disagree on the same `(subject_slug, attribute_key)`—for instance, `^f-010` from a Week 2 signed ADR stating that the core ledger stays on-premises (`phase1_hosting = 'on_prem'`) versus `^f-099` from a Week 5 status call claiming it moves to AWS (`phase1_hosting = 'aws_cloud'`)—Chotu **never overwrites or deactivates either claim**.

Instead, this task builds the `ConflictCluster` detector and the **Stage 0 Algorithm** of Workstream 2.4 (`TruthFinderEngine` in `backend/platform/knowledge/domain/truth_finder.py`):
1. **Iterative Truth Finder**: Starts each source with a prior based on its document class (`engagement` SOW / signed `deliverable` > `reference` standard > `client` workshop > casual chat) blended with the speaking stakeholder's `authority_weight` from `master_stakeholders`, and iteratively updates source trustworthiness $T(s)$ and claim confidence $C(f)$ using independent multi-perspective corroboration.
2. **HITS (Hubs & Authorities)**: Computes each claim's **Authority Score $A(f)$** (primary evidence backing and inbound citations) and **Hub Score $H(f)$** (downstream reach across BAITS syntheses, Master Data decisions/systems/deliverables, and outbound citations).

**Files:**
- Create: `backend/platform/knowledge/domain/truth_finder.py`
- Test: `backend/tests/platform/knowledge/domain/test_truth_finder_hits.py`

**Interfaces:**
- Consumes: `VersionedClaim`, `CitationEdge`, `KnowledgeScope`, `ClaimPerspective`, `EvolutionType` from Task 1; `StakeholderRecord`, `MasterDataDependencyGraph` from Task 2
- Produces:
  - `ConflictCluster(cluster_id: str, subject_slug: str, attribute_key: str, competing_claims: tuple[VersionedClaim, ...])`
  - `ClaimGraphScore(fact_id: str, source_trustworthiness: float, truth_finder_confidence: float, hits_authority_score: float, hits_hub_score: float, combined_algorithm_score: float)`
  - `detect_conflict_clusters(claims: Sequence[VersionedClaim]) -> list[ConflictCluster]`
  - `TruthFinderEngine.compute_scores(cluster: ConflictCluster, all_claims: Sequence[VersionedClaim], citation_edges: Sequence[CitationEdge], stakeholders_by_role: Mapping[str, StakeholderRecord], master_dependents: MasterDataDependencyGraph, iterations: int = 5) -> dict[str, ClaimGraphScore]`

- [ ] **Step 1: Write the failing test**

```python
# backend/tests/platform/knowledge/domain/test_truth_finder_hits.py
from datetime import datetime, timezone

from backend.platform.knowledge.domain.enums import (
    BaitsPillar,
    ClaimPerspective,
    EvolutionType,
    KnowledgeScope,
)
from backend.platform.knowledge.domain.master_data import (
    ArchitectureDecisionRecord,
    DeliverableRecord,
    MasterDataDependencyGraph,
    StakeholderRecord,
    SystemRecord,
)
from backend.platform.knowledge.domain.models import CitationEdge, VersionedClaim
from backend.platform.knowledge.domain.truth_finder import (
    TruthFinderEngine,
    detect_conflict_clusters,
)


def test_truth_finder_preserves_competing_claims_and_ranks_signed_sow_over_casual_chat() -> None:
    week2_sow_claim = VersionedClaim(
        fact_id="^f-010",
        concept_id="concept-core-ledger",
        pillar=BaitsPillar.TECHNOLOGY,
        subject_slug="core-ledger",
        attribute_key="phase1_hosting",
        claim_text="Core ledger remains on-premises in Phase 1 per regulatory residency mandate.",
        verbatim_quote="Phase 1 core ledger hosting shall remain on-premises.",
        source_uri="raw/week2-signed-sow-adr002.md",
        scope=KnowledgeScope.ENGAGEMENT,
        perspective=ClaimPerspective.CLIENT_STATED,
        speaker_role="Chief Architect",
        version=1,
        observed_at=datetime(2026, 10, 10, 10, 0, tzinfo=timezone.utc),
        ingested_at=datetime(2026, 10, 10, 10, 5, tzinfo=timezone.utc),
        evolution_type=EvolutionType.INITIAL,
        lineage_parent_id=None,
        is_active=True,
    )
    week2_corroboration = VersionedClaim(
        fact_id="^f-011",
        concept_id="concept-security-residency",
        pillar=BaitsPillar.SECURITY,
        subject_slug="core-ledger",
        attribute_key="phase1_hosting",
        claim_text="Core ledger remains on-premises in Phase 1 per regulatory residency mandate.",
        verbatim_quote="Security audit confirmed on-prem residency lock for Phase 1.",
        source_uri="raw/week2-security-audit.md",
        scope=KnowledgeScope.INTERNAL,
        perspective=ClaimPerspective.WORK_RESEARCH,
        speaker_role="Security Lead",
        version=1,
        observed_at=datetime(2026, 10, 11, 14, 0, tzinfo=timezone.utc),
        ingested_at=datetime(2026, 10, 11, 14, 5, tzinfo=timezone.utc),
        evolution_type=EvolutionType.INITIAL,
        lineage_parent_id=None,
        is_active=True,
    )
    week5_casual_claim = VersionedClaim(
        fact_id="^f-099",
        concept_id="concept-core-ledger",
        pillar=BaitsPillar.TECHNOLOGY,
        subject_slug="core-ledger",
        attribute_key="phase1_hosting",
        claim_text="We are moving the whole core ledger to AWS cloud in Phase 1.",
        verbatim_quote="Yeah we're just moving the whole ledger to AWS in Phase 1.",
        source_uri="raw/week5-casual-standup.md",
        scope=KnowledgeScope.CLIENT,
        perspective=ClaimPerspective.CLIENT_STATED,
        speaker_role="Associate PM",
        version=1,
        observed_at=datetime(2026, 10, 27, 16, 0, tzinfo=timezone.utc),
        ingested_at=datetime(2026, 10, 27, 16, 5, tzinfo=timezone.utc),
        evolution_type=EvolutionType.COMPETING_CLAIM,
        lineage_parent_id=None,
        is_active=True,
    )

    all_claims = [week2_sow_claim, week2_corroboration, week5_casual_claim]
    clusters = detect_conflict_clusters(all_claims)
    assert len(clusters) == 1
    cluster = clusters[0]
    assert cluster.subject_slug == "core-ledger"
    assert cluster.attribute_key == "phase1_hosting"
    # Neither claim was deactivated
    assert all(c.is_active for c in cluster.competing_claims)

    stakeholders = {
        "Chief Architect": StakeholderRecord(
            stakeholder_id="stk-1",
            name="Elena Rostova",
            title="Chief Architect",
            organization="client",
            department="Architecture",
            influence_level=5,
            authority_weight=0.95,
            linked_fact_ids=("^f-010",),
        ),
        "Associate PM": StakeholderRecord(
            stakeholder_id="stk-2",
            name="Dan Smith",
            title="Associate PM",
            organization="client",
            department="Product",
            influence_level=2,
            authority_weight=0.40,
            linked_fact_ids=("^f-099",),
        ),
    }
    adr = ArchitectureDecisionRecord(
        decision_id="ADR-002",
        title="Keep Core Ledger On-Premises in Phase 1",
        status="accepted",
        decided_by_stakeholder_ids=("stk-1",),
        impacted_system_ids=("sys-core-ledger",),
        impacted_capability_ids=(),
        rationale_fact_ids=("^f-010",),
    )
    dependents = MasterDataDependencyGraph(
        decisions_by_fact={"^f-010": [adr], "^f-099": []},
        systems_by_fact={
            "^f-010": [
                SystemRecord(
                    system_id="sys-core-ledger",
                    name="Core Ledger",
                    category="core",
                    owner_stakeholder_id="stk-1",
                    lifecycle_state="legacy",
                    baits_pillars=(BaitsPillar.TECHNOLOGY,),
                    linked_fact_ids=("^f-010",),
                )
            ],
            "^f-099": [],
        },
        deliverables_by_decision={
            "ADR-002": [
                DeliverableRecord(
                    deliverable_id="deliv-1",
                    title="Target Arch",
                    sow_milestone="M2",
                    status="approved",
                    cited_fact_ids=("^f-010",),
                    depended_decision_ids=("ADR-002",),
                )
            ]
        },
        raid_by_fact={"^f-010": [], "^f-099": []},
    )
    edges = [
        CitationEdge(
            source_concept_id="baits-technology",
            source_fact_id="^synth-01",
            target_concept_id="concept-core-ledger",
            target_fact_id="^f-010",
        )
    ]

    scores = TruthFinderEngine.compute_scores(
        cluster=cluster,
        all_claims=all_claims,
        citation_edges=edges,
        stakeholders_by_role=stakeholders,
        master_dependents=dependents,
    )
    assert scores["^f-010"].truth_finder_confidence > scores["^f-099"].truth_finder_confidence
    assert scores["^f-010"].hits_hub_score > scores["^f-099"].hits_hub_score
    assert scores["^f-010"].combined_algorithm_score > scores["^f-099"].combined_algorithm_score
```

- [ ] **Step 2: Run test to verify it fails**

Run: `uv run pytest backend/tests/platform/knowledge/domain/test_truth_finder_hits.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'backend.platform.knowledge.domain.truth_finder'`

- [ ] **Step 3: Implement `detect_conflict_clusters` and `TruthFinderEngine.compute_scores`**

In `backend/platform/knowledge/domain/truth_finder.py`:
- `detect_conflict_clusters()` groups active claims by `(subject_slug, attribute_key)`. If a group contains $\ge 2$ distinct normalized `claim_text` values where the newer claim does not have `evolution_type == EvolutionType.ITERATION` pointing to the older claim via `lineage_parent_id`, emit a `ConflictCluster`.
- `TruthFinderEngine.compute_scores()` computes base source reliability from `scope` (`ENGAGEMENT: 0.90`, `DELIVERABLE: 0.90`, `REFERENCE: 0.85`, `PRIOR_PROJECT: 0.80`, `CLIENT: 0.70`, `INTERNAL: 0.75`) averaged with stakeholder `authority_weight` (default `0.60`), boosts claims corroborated by independent sources/perspectives with matching normalized text over `iterations` rounds, and computes HITS Authority (inbound citations + corroboration) and Hub (dependent Master Data decisions, systems, deliverables, and synthesis links) normalized to $[0, 1]$.

- [ ] **Step 4: Run test to verify it passes**

Run: `uv run pytest backend/tests/platform/knowledge/domain/test_truth_finder_hits.py -v`
Expected: PASS (1 passed)

- [ ] **Step 5: Commit**

```bash
git add backend/platform/knowledge/domain/truth_finder.py backend/tests/platform/knowledge/domain/test_truth_finder_hits.py
git commit -m "feat(knowledge): implement active TruthFinder and HITS graph scoring engine"
```

---

### Task 7: 2-Step Conflict Weighing Pipeline (Step 1: `RelationshipImpactAnalyzer` Make/Break Simulation; Step 2: `PlausibilityWeigher` & HITL `ConflictWeighingReport`)

Right after the Truth Finder + HITS algorithm finishes, Chotu runs the **two explicit post-algorithm steps** you specified in the retrospective:
- **Step 1 (`RelationshipImpactAnalyzer`)**: Simulates accepting each competing option across the engagement graph to enumerate exactly which existing relationships (corroborated facts, accepted `master_decisions` ADRs, `master_systems`, `master_deliverables`, and BAITS syntheses) that option **breaks** versus which new cross-pillar relationships or open RAID assumptions it **makes**.
- **Step 2 (`PlausibilityWeigher`)**: Evaluates which option **makes more sense** in domain context (using reference standards ingested from Universal Knowledge via Task 5 when needed) and combines the Truth Finder/HITS score, the Make/Break delta, and the Plausibility score into a calibrated `likelihood_weight` ($\sum p_i = 1.0$) and a side-by-side `ConflictWeighingReport` that keeps synthesis unblocked while surfacing the exact impact diff for HITL review.

**Files:**
- Create: `backend/platform/knowledge/domain/conflict_weighing.py`
- Create: `backend/platform/knowledge/ports/truth_finder_port.py`
- Create: `backend/platform/knowledge/adapters/truth_finder_service.py`
- Test: `backend/tests/platform/knowledge/test_conflict_weighing_pipeline.py`

**Interfaces:**
- Consumes: `ConflictCluster`, `ClaimGraphScore`, `TruthFinderEngine` from Task 6; `EngagementMasterDataPort`, `MasterDataDependencyGraph` from Task 2; `UniversalKnowledgeBridgePort` from Task 5
- Produces:
  - `RelationshipImpact(fact_id: str, broken_relationships: tuple[str, ...], made_relationships: tuple[str, ...], net_make_break_score: float)`
  - `PlausibilityAssessment(fact_id: str, plausibility_score: float, supporting_reference_fact_ids: tuple[str, ...], sense_making_rationale: str)`
  - `WeighedOption(fact_id: str, claim_text: str, perspective: ClaimPerspective, scope: KnowledgeScope, graph_score: ClaimGraphScore, relationship_impact: RelationshipImpact, plausibility: PlausibilityAssessment, calibrated_likelihood: float)`
  - `ConflictWeighingReport(cluster_id: str, subject_slug: str, attribute_key: str, options: tuple[WeighedOption, ...], leading_fact_id: str, requires_hitl_confirmation: bool)`
  - `RelationshipImpactAnalyzer.analyze(cluster: ConflictCluster, all_claims: Sequence[VersionedClaim], citation_edges: Sequence[CitationEdge], master_dependents: MasterDataDependencyGraph) -> dict[str, RelationshipImpact]`
  - `PlausibilityWeigher.weigh_cluster(...) -> ConflictWeighingReport`

- [ ] **Step 1: Write the failing test**

```python
# backend/tests/platform/knowledge/test_conflict_weighing_pipeline.py
from datetime import datetime, timezone
import pytest

from backend.platform.knowledge.domain.conflict_weighing import (
    PlausibilityAssessment,
    PlausibilityWeigher,
    RelationshipImpactAnalyzer,
)
from backend.platform.knowledge.domain.enums import (
    BaitsPillar,
    ClaimPerspective,
    EvolutionType,
    KnowledgeScope,
)
from backend.platform.knowledge.domain.master_data import (
    ArchitectureDecisionRecord,
    DeliverableRecord,
    MasterDataDependencyGraph,
    RaidItemRecord,
    StakeholderRecord,
    SystemRecord,
)
from backend.platform.knowledge.domain.models import CitationEdge, VersionedClaim
from backend.platform.knowledge.domain.truth_finder import (
    TruthFinderEngine,
    detect_conflict_clusters,
)


def test_make_break_analyzer_lists_broken_adr_and_deliverables_and_weighs_options() -> None:
    claim_on_prem = VersionedClaim(
        fact_id="^f-010",
        concept_id="concept-core-ledger",
        pillar=BaitsPillar.TECHNOLOGY,
        subject_slug="core-ledger",
        attribute_key="phase1_hosting",
        claim_text="Core ledger remains on-premises in Phase 1.",
        verbatim_quote="Phase 1 core ledger hosting shall remain on-premises.",
        source_uri="raw/week2-signed-sow.md",
        scope=KnowledgeScope.ENGAGEMENT,
        perspective=ClaimPerspective.CLIENT_STATED,
        speaker_role="Chief Architect",
        version=1,
        observed_at=datetime(2026, 10, 10, 10, 0, tzinfo=timezone.utc),
        ingested_at=datetime(2026, 10, 10, 10, 5, tzinfo=timezone.utc),
        evolution_type=EvolutionType.INITIAL,
        lineage_parent_id=None,
        is_active=True,
    )
    claim_aws = VersionedClaim(
        fact_id="^f-099",
        concept_id="concept-core-ledger",
        pillar=BaitsPillar.TECHNOLOGY,
        subject_slug="core-ledger",
        attribute_key="phase1_hosting",
        claim_text="Move core ledger to AWS cloud in Phase 1.",
        verbatim_quote="Moving the whole ledger to AWS in Phase 1.",
        source_uri="raw/week5-standup.md",
        scope=KnowledgeScope.CLIENT,
        perspective=ClaimPerspective.CLIENT_STATED,
        speaker_role="Associate PM",
        version=1,
        observed_at=datetime(2026, 10, 27, 16, 0, tzinfo=timezone.utc),
        ingested_at=datetime(2026, 10, 27, 16, 5, tzinfo=timezone.utc),
        evolution_type=EvolutionType.COMPETING_CLAIM,
        lineage_parent_id=None,
        is_active=True,
    )
    cluster = detect_conflict_clusters([claim_on_prem, claim_aws])[0]

    adr = ArchitectureDecisionRecord(
        decision_id="ADR-002",
        title="Keep Core Ledger On-Premises in Phase 1",
        status="accepted",
        decided_by_stakeholder_ids=("stk-1",),
        impacted_system_ids=("sys-core-ledger",),
        impacted_capability_ids=(),
        rationale_fact_ids=("^f-010",),
    )
    dependents = MasterDataDependencyGraph(
        decisions_by_fact={"^f-010": [adr], "^f-099": []},
        systems_by_fact={
            "^f-010": [
                SystemRecord(
                    system_id="sys-core-ledger",
                    name="Core Ledger",
                    category="core",
                    owner_stakeholder_id="stk-1",
                    lifecycle_state="legacy",
                    baits_pillars=(BaitsPillar.TECHNOLOGY,),
                    linked_fact_ids=("^f-010",),
                )
            ],
            "^f-099": [],
        },
        deliverables_by_decision={
            "ADR-002": [
                DeliverableRecord(
                    deliverable_id="deliv-target-arch-v1",
                    title="Phase 1 Target Architecture Blueprint",
                    sow_milestone="M2",
                    status="approved",
                    cited_fact_ids=("^f-010",),
                    depended_decision_ids=("ADR-002",),
                )
            ]
        },
        raid_by_fact={
            "^f-010": [
                RaidItemRecord(
                    raid_id="RAID-R01",
                    category="Risk",
                    severity="high",
                    owner_stakeholder_id="stk-1",
                    mitigation_plan="On-prem residency lock",
                    impacted_deliverable_ids=("deliv-target-arch-v1",),
                    trigger_conflict_id=None,
                    linked_fact_ids=("^f-010",),
                )
            ],
            "^f-099": [],
        },
    )
    edges = [
        CitationEdge(
            source_concept_id="baits-technology",
            source_fact_id="^synth-tech-01",
            target_concept_id="concept-core-ledger",
            target_fact_id="^f-010",
        )
    ]

    # Step 1: Structural Relationship Make/Break Impact
    impacts = RelationshipImpactAnalyzer.analyze(
        cluster=cluster,
        all_claims=[claim_on_prem, claim_aws],
        citation_edges=edges,
        master_dependents=dependents,
    )
    # Accepting ^f-099 breaks ADR-002, deliv-target-arch-v1, sys-core-ledger, and baits-technology synthesis
    assert "decision:ADR-002" in impacts["^f-099"].broken_relationships
    assert "deliverable:deliv-target-arch-v1" in impacts["^f-099"].broken_relationships
    assert "synthesis:baits-technology#^synth-tech-01" in impacts["^f-099"].broken_relationships
    assert impacts["^f-010"].net_make_break_score > impacts["^f-099"].net_make_break_score

    # Step 2: Contextual Plausibility ("Makes More Sense") + Final Calibrated Weighing
    graph_scores = TruthFinderEngine.compute_scores(
        cluster=cluster,
        all_claims=[claim_on_prem, claim_aws],
        citation_edges=edges,
        stakeholders_by_role={
            "Chief Architect": StakeholderRecord(
                stakeholder_id="stk-1",
                name="Elena Rostova",
                title="Chief Architect",
                organization="client",
                department="Arch",
                influence_level=5,
                authority_weight=0.95,
                linked_fact_ids=("^f-010",),
            )
        },
        master_dependents=dependents,
    )
    plausibility_map = {
        "^f-010": PlausibilityAssessment(
            fact_id="^f-010",
            plausibility_score=0.92,
            supporting_reference_fact_ids=("^u-f401",),
            sense_making_rationale="Aligns with regulatory data residency lock and 4-hour mainframe batch constraint.",
        ),
        "^f-099": PlausibilityAssessment(
            fact_id="^f-099",
            plausibility_score=0.18,
            supporting_reference_fact_ids=("^u-f401",),
            sense_making_rationale="Contradicts signed residency mandate without a data tokenization plan.",
        ),
    }

    report = PlausibilityWeigher.weigh_cluster(
        cluster=cluster,
        graph_scores=graph_scores,
        relationship_impacts=impacts,
        plausibility_assessments=plausibility_map,
    )
    assert report.leading_fact_id == "^f-010"
    assert report.requires_hitl_confirmation is True
    assert abs(sum(opt.calibrated_likelihood for opt in report.options) - 1.0) < 1e-6
    opt_on_prem = next(o for o in report.options if o.fact_id == "^f-010")
    opt_aws = next(o for o in report.options if o.fact_id == "^f-099")
    assert opt_on_prem.calibrated_likelihood > 0.75
    assert opt_aws.calibrated_likelihood < 0.25
```

- [ ] **Step 2: Run test to verify it fails**

Run: `uv run pytest backend/tests/platform/knowledge/test_conflict_weighing_pipeline.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'backend.platform.knowledge.domain.conflict_weighing'`

- [ ] **Step 3: Implement `RelationshipImpactAnalyzer`, `PlausibilityWeigher`, and `TruthFinderService`**

In `backend/platform/knowledge/domain/conflict_weighing.py`:
- `RelationshipImpactAnalyzer.analyze()` iterates over each candidate option $O_i$ in `cluster.competing_claims`. Every dependent decision (`decision:<id>`), dependent deliverable (`deliverable:<id>`), dependent system (`system:<id>`), and inbound synthesis/concept citation (`synthesis:<concept>#<fact>`) tied to a *rival* option $O_j \ne O_i$ is recorded in $O_i$'s `broken_relationships`. Every cross-pillar link, resolved RAID item (`raid:<id>`), and corroborated entity link backed by $O_i$ is recorded in $O_i$'s `made_relationships`. Compute `net_make_break_score = (len(made) - len(broken)) / max(1, len(made) + len(broken))` mapped to $[0, 1]$.
- `PlausibilityWeigher.weigh_cluster()` combines `0.35 * graph_score.combined_algorithm_score + 0.35 * impact.net_make_break_score + 0.30 * plausibility.plausibility_score` via temperature-scaled softmax (`temperature=0.25`) so probabilities sum to `1.0`, sorts options descending by `calibrated_likelihood`, sets `leading_fact_id = options[0].fact_id`, and sets `requires_hitl_confirmation = True`.

- [ ] **Step 4: Run test to verify it passes**

Run: `uv run pytest backend/tests/platform/knowledge/test_conflict_weighing_pipeline.py -v`
Expected: PASS (1 passed)

- [ ] **Step 5: Commit**

```bash
git add backend/platform/knowledge/domain/conflict_weighing.py backend/platform/knowledge/ports/truth_finder_port.py backend/platform/knowledge/adapters/truth_finder_service.py backend/tests/platform/knowledge/test_conflict_weighing_pipeline.py
git commit -m "feat(knowledge): add 2-step Make/Break RelationshipImpactAnalyzer and PlausibilityWeigher"
```

---

### Task 8: 5-Lens Engagement Closure Retrospective Queue & `RetrospectiveElevator` (`engagement://<engagement_id>#^<anchor>` Two-Stage Correlation)

When an engagement closes (or finishes a major milestone), its `eng_<slug>.*` dataset remains intact as an empirical record while Chotu runs the **5-Lens Retrospective Discovery Workflow** and elevates sanitized learnings into `universal.*`. Every elevated claim in `universal.*` carries a mandatory `engagement://eng_<slug>#^retro-<id>` citation correlation key so that six or twelve months later, Chotu can both recall the high-level lesson in milliseconds *and* drill down into `eng_<slug>.*` to inspect the exact Master Data, stakeholder dynamics, and Truth Finder conflicts behind that lesson.

This task implements `RetrospectiveLens` (all 5 lenses), `RetrospectiveElevatorPort`, and `RetrospectiveElevatorService`, including two-stage drill-down (`drill_down_to_engagement_reality()`) and strict validation that rejects any elevated learning lacking a valid `EngagementCitationUri`.

**Files:**
- Create: `backend/platform/knowledge/domain/elevation.py`
- Create: `backend/platform/knowledge/ports/elevation_port.py`
- Create: `backend/platform/knowledge/adapters/retrospective_elevator.py`
- Test: `backend/tests/platform/knowledge/test_retrospective_elevation.py`

**Interfaces:**
- Consumes: `KnowledgeCoordinates`, `EngagementCitationUri`, `InvalidElevationCitationError`, `BaitsPillar`, `KnowledgeScope`, `ClaimPerspective` from Task 1; `EngagementMasterDataPort` from Task 2; `OKFIndexerPort` from Task 3; `KnowledgeLibrarianPort` from Task 4
- Produces:
  - `RetrospectiveLens(StrEnum)`:
    - `'ACHIEVEMENTS_AND_PATH'` (Lens 1: What we achieved & how we had to execute)
    - `'METHODOLOGY_EFFICACY'` (Lens 2: Which methods/frameworks worked vs. failed)
    - `'PPTTT_PATTERNS'` (Lens 3: People, Process, Technology, Tools & Techniques)
    - `'COUNTERFACTUALS_AND_ANTI_REPEAT'` (Lens 4: How we would approach differently & Day-1 rules)
    - `'HARVESTED_RESEARCH'` (Lens 5: Reusable domain & architectural research)
  - `RetrospectiveDiscoveryItem(anchor_id: str, lens: RetrospectiveLens, pillar: BaitsPillar, sanitized_claim_text: str, verbatim_evidence_quote: str, underlying_engagement_fact_ids: tuple[str, ...], citation_uri: str)`
  - `EngagementDrillDownReport(citation_uri: EngagementCitationUri, elevated_claim_text: str, source_engagement_claims: tuple[VersionedClaim, ...], linked_decisions: tuple[ArchitectureDecisionRecord, ...], linked_systems: tuple[SystemRecord, ...])`
  - `RetrospectiveElevatorPort(ABC)` in `ports/elevation_port.py`:
    - `async def elevate_engagement_retrospective(self, eng_coords: KnowledgeCoordinates, items: Sequence[RetrospectiveDiscoveryItem]) -> list[EngagementCitationUri]`
    - `async def drill_down_to_engagement_reality(self, citation_uri: str) -> EngagementDrillDownReport`

- [ ] **Step 1: Write the failing test**

```python
# backend/tests/platform/knowledge/test_retrospective_elevation.py
import pytest

from backend.platform.knowledge.adapters.memory_master_data import InMemoryEngagementMasterDataAdapter
from backend.platform.knowledge.adapters.okf_indexer import InMemoryOKFStore, OKFIndexer
from backend.platform.knowledge.adapters.retrospective_elevator import RetrospectiveElevatorService
from backend.platform.knowledge.adapters.small_to_big_librarian import SmallToBigLibrarian
from backend.platform.knowledge.domain.elevation import (
    RetrospectiveDiscoveryItem,
    RetrospectiveLens,
)
from backend.platform.knowledge.domain.enums import BaitsPillar
from backend.platform.knowledge.domain.errors import InvalidElevationCitationError
from backend.platform.knowledge.domain.master_data import ArchitectureDecisionRecord
from backend.platform.knowledge.domain.models import KnowledgeCoordinates
from backend.platform.knowledge.domain.retrieval import LibrarianQuery
from backend.tests.platform.knowledge.test_okf_indexer import (
    SAMPLE_CONCEPT_MD,
    FakeDeterministicEmbeddingAdapter,
)


@pytest.mark.asyncio
async def test_five_lens_retrospective_elevates_to_universal_and_supports_two_stage_drill_down() -> None:
    store = InMemoryOKFStore()
    embedder = FakeDeterministicEmbeddingAdapter()
    indexer = OKFIndexer(store=store, embedder=embedder)
    librarian = SmallToBigLibrarian(store=store, embedder=embedder)
    master_data = InMemoryEngagementMasterDataAdapter()

    eng_coords = KnowledgeCoordinates.for_engagement("2026_acme")
    await master_data.provision_schema(eng_coords)
    await indexer.index_markdown_file(
        eng_coords, "okf-wiki/systems/concept-core-ledger.md", SAMPLE_CONCEPT_MD
    )
    await master_data.upsert_decision(
        eng_coords,
        ArchitectureDecisionRecord(
            decision_id="ADR-002",
            title="Decouple Read-Replica Reconciliation Before Cutover",
            status="accepted",
            decided_by_stakeholder_ids=("stk-chief-arch",),
            impacted_system_ids=("sys-core-ledger",),
            impacted_capability_ids=(),
            rationale_fact_ids=("^f-102",),
        ),
    )

    elevator = RetrospectiveElevatorService(
        store=store,
        indexer=indexer,
        librarian=librarian,
        master_data=master_data,
    )

    items = [
        RetrospectiveDiscoveryItem(
            anchor_id="^retro-01",
            lens=RetrospectiveLens.ACHIEVEMENTS_AND_PATH,
            pillar=BaitsPillar.BUSINESS,
            sanitized_claim_text="Phased regional settlement rollout achieved zero-downtime cutover.",
            verbatim_evidence_quote="Completed 3-region rollout with zero SLA breaches.",
            underlying_engagement_fact_ids=("^f-101",),
            citation_uri="engagement://eng_2026_acme#^retro-01",
        ),
        RetrospectiveDiscoveryItem(
            anchor_id="^retro-02",
            lens=RetrospectiveLens.METHODOLOGY_EFFICACY,
            pillar=BaitsPillar.APPLICATION,
            sanitized_claim_text="BVF capability-to-system mapping exposed batch dependency in Week 1.",
            verbatim_evidence_quote="BVF matrix flagged the 4-hour mainframe lock early.",
            underlying_engagement_fact_ids=("^f-102",),
            citation_uri="engagement://eng_2026_acme#^retro-02",
        ),
        RetrospectiveDiscoveryItem(
            anchor_id="^retro-03",
            lens=RetrospectiveLens.PPTTT_PATTERNS,
            pillar=BaitsPillar.TECHNOLOGY,
            sanitized_claim_text="Pairing client Principal DBA with internal Security Lead resolved residency deadlock.",
            verbatim_evidence_quote="Joint working session produced ADR-002.",
            underlying_engagement_fact_ids=("^f-102",),
            citation_uri="engagement://eng_2026_acme#^retro-03",
        ),
        RetrospectiveDiscoveryItem(
            anchor_id="^retro-04",
            lens=RetrospectiveLens.COUNTERFACTUALS_AND_ANTI_REPEAT,
            pillar=BaitsPillar.TECHNOLOGY,
            sanitized_claim_text="Factor in 4-hour legacy mainframe batch locks on Day 1 before sizing CDC pipelines.",
            verbatim_evidence_quote="Batch processing locks ledger writes between 01:00 and 05:00 UTC.",
            underlying_engagement_fact_ids=("^f-102",),
            citation_uri="engagement://eng_2026_acme#^retro-04",
        ),
        RetrospectiveDiscoveryItem(
            anchor_id="^retro-05",
            lens=RetrospectiveLens.HARVESTED_RESEARCH,
            pillar=BaitsPillar.SECURITY,
            sanitized_claim_text="Dual-region active-active ledger settlement requires deterministic idempotency keys.",
            verbatim_evidence_quote="Idempotency key standard prevented duplicate settlement.",
            underlying_engagement_fact_ids=("^f-101",),
            citation_uri="engagement://eng_2026_acme#^retro-05",
        ),
    ]

    uris = await elevator.elevate_engagement_retrospective(eng_coords, items)
    assert len(uris) == 5

    # Stage 1: Fast recall in Universal Knowledge finds the elevated anti-repeat rule
    uk_packet = await librarian.retrieve(
        KnowledgeCoordinates.universal(),
        LibrarianQuery(query_text="legacy mainframe batch locks Day 1 CDC"),
    )
    assert len(uk_packet.hydrated_contexts) >= 1
    assert any(
        "engagement://eng_2026_acme#^retro-04" in c.source_uri
        for ctx in uk_packet.hydrated_contexts
        for c in ctx.matched_claims
    )

    # Stage 2: Targeted drill-down via engagement://eng_2026_acme#^retro-04 recovers empirical reality
    drill_down = await elevator.drill_down_to_engagement_reality(
        "engagement://eng_2026_acme#^retro-04"
    )
    assert drill_down.citation_uri.engagement_slug == "eng_2026_acme"
    assert [c.fact_id for c in drill_down.source_engagement_claims] == ["^f-102"]
    assert [d.decision_id for d in drill_down.linked_decisions] == ["ADR-002"]


@pytest.mark.asyncio
async def test_elevator_rejects_missing_engagement_correlation_uri() -> None:
    store = InMemoryOKFStore()
    embedder = FakeDeterministicEmbeddingAdapter()
    indexer = OKFIndexer(store=store, embedder=embedder)
    librarian = SmallToBigLibrarian(store=store, embedder=embedder)
    master_data = InMemoryEngagementMasterDataAdapter()
    elevator = RetrospectiveElevatorService(
        store=store, indexer=indexer, librarian=librarian, master_data=master_data
    )

    bad_item = RetrospectiveDiscoveryItem(
        anchor_id="^retro-99",
        lens=RetrospectiveLens.COUNTERFACTUALS_AND_ANTI_REPEAT,
        pillar=BaitsPillar.TECHNOLOGY,
        sanitized_claim_text="Unanchored claim",
        verbatim_evidence_quote="Quote",
        underlying_engagement_fact_ids=("^f-102",),
        citation_uri="engagement://eng_other_project#^retro-99",
    )
    with pytest.raises(InvalidElevationCitationError):
        await elevator.elevate_engagement_retrospective(
            KnowledgeCoordinates.for_engagement("2026_acme"), [bad_item]
        )
```

- [ ] **Step 2: Run test to verify it fails**

Run: `uv run pytest backend/tests/platform/knowledge/test_retrospective_elevation.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'backend.platform.knowledge.domain.elevation'`

- [ ] **Step 3: Implement `RetrospectiveElevatorService` and two-stage drill-down**

In `backend/platform/knowledge/adapters/retrospective_elevator.py`:
- Validate every `RetrospectiveDiscoveryItem.citation_uri` via `EngagementCitationUri.parse()`, verifying that `parsed.engagement_slug == eng_coords.schema_name` and `parsed.anchor_id == item.anchor_id` (raising `InvalidElevationCitationError` otherwise).
- Write the engagement's canonical retrospective Markdown (`okf-wiki/retrospective/concept-retro-<slug>.md`) into `eng_coords` via `indexer.index_markdown_file()`, record the mapping from `citation_uri` to `underlying_engagement_fact_ids`, and then write the sanitized elevated concept Markdown into `KnowledgeCoordinates.universal()` (`okf-wiki/elevated/concept-elevated-<slug>.md`) with `scope: prior-project`, `perspective: work_research`, and `source_uri: engagement://<eng_slug>#<anchor_id>`.
- Implement `drill_down_to_engagement_reality(citation_uri: str)` to parse the `EngagementCitationUri`, resolve the target engagement's `KnowledgeCoordinates`, fetch the underlying `^f-id` claims from `store`, and query `master_data.get_dependents_for_facts()` to return the complete `EngagementDrillDownReport`.

- [ ] **Step 4: Run test to verify it passes**

Run: `uv run pytest backend/tests/platform/knowledge/test_retrospective_elevation.py -v && uv run mypy --strict backend/platform/knowledge`
Expected: PASS (all 13 tests across Tasks 1–8 pass, 0 mypy errors)

- [ ] **Step 5: Commit**

```bash
git add backend/platform/knowledge/domain/elevation.py backend/platform/knowledge/ports/elevation_port.py backend/platform/knowledge/adapters/retrospective_elevator.py backend/tests/platform/knowledge/test_retrospective_elevation.py
git commit -m "feat(knowledge): implement 5-lens RetrospectiveElevator and engagement_id drill-down"
```
