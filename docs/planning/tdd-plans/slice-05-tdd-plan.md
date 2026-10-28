# Slice Group 5: Personal R&D Mode, Self-Extension Builders, Evaluation Judges & Continuous Learning Loop Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build Chotu's isolated Personal R&D Mode (`personal.*` schema + `workspaces/personal_rd/`), the sandboxed Self-Extension Builder Workbench (`tool_builder`, `skill_builder`, `agent_builder`, `app_onboarding`), the Epistemic & Retrieval Evaluation Judges (`CitationFirewallVerifier`, `FaithfulnessJudgePort`, `CompletenessJudgePort`) wired into closed-loop `RetrievalTrace` scoring, the Laya Classifier Calibration & Golden-Set Distillation Flywheel, and the `TechniquePromoter` bridge that elevates validated personal techniques and tools into Universal Knowledge (`universal.*`) and `ServiceDiscoveryPort`.

**Architecture:** Slice Group 5 extends the 9-layer Fractal Hexagonal microkernel (`backend/platform/`) and capability inventory (`backend/inventory/`) without breaking closed-world engagement isolation. Personal R&D notes and technique studies execute inside the dedicated `personal.*` PostgreSQL schema and `workspaces/personal_rd/` directory with mandatory `perspective: personal_rd` and `scope: internal` tags. Self-extending builders scaffold candidate capabilities inside ephemeral sandbox worktrees (`workspaces/personal_rd/sandbox/<build_id>/`), run deterministic static and test gates (`ruff`, `mypy --strict`, `check_hexagonal_boundaries.py`, `pytest`), and require human approval via `ApprovalBusPort` before installing into `~/.chotu/` or promoting into `universal.*`.

**Tech Stack:** Python 3.12+, `asyncio`, PostgreSQL 16 + `pgvector` (`asyncpg` / SQLAlchemy 2.0 async), Pydantic v2 / `dataclasses(frozen=True)`, PyYAML, Laya encoder checkpoint runtime (`onnxruntime` / PyTorch adapter), `pytest`, `pytest-asyncio`, `ruff`, `mypy --strict`.

**Spec:** [`docs/planning/slice-05-personal-rd-builders-evals-and-learning-loop.md`](file:///home/nox/Developer/ds2/dsa-agent/docs/planning/slice-05-personal-rd-builders-evals-and-learning-loop.md)

## Global Constraints

- Domain layer purity: All files under `backend/platform/*/domain/` MUST contain zero imports from `fastapi`, `sqlalchemy`, `asyncpg`, `litellm`, `onnxruntime`, `torch`, or `backend.inventory` (verified by `scripts/check_hexagonal_boundaries.py`).
- Personal Mode epistemic firewall: Every record created in `personal.*` MUST carry `perspective = "personal_rd"` and `scope = "internal"`. Work Mode retrieval against `eng_<slug>.*` or `universal.*` MUST NEVER query `personal.*` directly.
- Builder graduation gate: No builder subagent (`tool_builder`, `skill_builder`, `agent_builder`, `app_onboarding`) may write directly to `backend/inventory/` or `~/.chotu/` or call `ServiceDiscoveryPort.register()` until `SandboxWorktreeRunner.verify()` passes all 4 gates (`ruff`, `mypy --strict`, `check_hexagonal_boundaries.py`, `pytest`) AND `ApprovalBusPort` returns `ApprovalStatus.APPROVED`.
- Bounded corrective faithfulness loop: `FaithfulnessCorrectiveLoop` MUST cap corrective synthesis retries at `max_retries: int = 2` (harvested from `okf-kb-manager` ADR-0009) and explicitly surface unverified claims if the retry budget is exhausted rather than looping infinitely or silently dropping citations.
- Single-source YAML taxonomy & Golden-Set non-regression gate: Classifier taxonomies under `backend/inventory/agents/internal/classifiers/taxonomies/*.yaml` are the sole source of valid labels; a candidate Laya checkpoint may only be promoted if its macro-F1 on the human-verified golden set is $\ge$ the current champion checkpoint's macro-F1 and its ECE (Expected Calibration Error) is $\le 0.05$.

## Review Focus

- **Unpromoted `personal.*` technique study or note leaking into Work Mode retrieval**: If a query in Work Mode (`mode="work"`, `engagement_id="eng_acme"`) accidentally searches across `personal.*` or accepts a `perspective="personal_rd"` fact into a `client` or `deliverable` synthesis, unverified Sunday experiments could appear in a client deliverable. Pinned by `test_personal_study_enforces_personal_rd_and_blocks_unvalidated_promotion` in Task 1 and `test_citation_firewall_blocks_personal_rd_and_fabricated_anchors` in Task 3.
- **Unverified or boundary-violating builder code registering into live `ServiceDiscoveryPort`**: If `tool_builder` generates a Python tool that imports a forbidden vendor SDK in `domain/` or fails unit tests, and registers it before sandbox verification and `ApprovalBus` approval, it can crash the active harness. Pinned by `test_sandbox_builder_blocks_registration_when_boundary_or_pytest_fails` in Task 2.
- **Ungrounded LLM synthesis claim bypassing `FaithfulnessJudge` or looping infinitely on retry**: When a synthesizer fabricates a metric not entailed by its cited `ContextPacket` facts, the system must retry with exact judge feedback up to `max_retries=2` and then mark the claim `unverified_claim_warning=True` if still unfaithful. Pinned by `test_faithfulness_loop_retries_with_feedback_and_halts_at_max_retries` in Task 3.
- **Noisy or hallucination-inducing capability retaining high `ServiceLocator` rank**: If a retrieved `SKILL.md` or `chotu://` tool consistently triggers corrective retries or goes uncited in verified outputs without having its `historical_reliability_score` decayed, Stage 3 of `ServiceDiscoveryPort` will keep ranking it at the top. Pinned by `test_retrieval_effectiveness_updater_boosts_cited_and_decays_penalized_capabilities` in Task 4.
- **Personal scratch notes or private paths leaking into `universal.*` during `TechniquePromoter` elevation, or a regressed Laya checkpoint replacing the champion**: Pinned by `test_laya_golden_set_gate_rejects_regressed_checkpoint` in Task 5 and `test_technique_promoter_rejects_unvalidated_or_unsanitized_study_and_promotes_approved` in Task 6.

---

## File Structure & Responsibilities

```text
backend/
├── platform/
│   ├── knowledge/
│   │   ├── domain/
│   │   │   ├── personal_rd.py                  # Task 1: TechniqueReadiness, TechniqueStudy, ExperimentRun
│   │   │   ├── evaluation.py                   # Task 3: CitationViolation, FaithfulnessVerdict, CompletenessVerdict
│   │   │   ├── retrieval_effectiveness.py      # Task 4: RetrievalTraceRecord, CapabilityReliabilityUpdate
│   │   │   └── technique_promotion.py          # Task 6: PromotionCandidate, PromotionPackage, PromotionSanitizer
│   │   ├── ports/
│   │   │   ├── personal_rd_port.py             # Task 1: PersonalRDRepositoryPort
│   │   │   ├── evaluation_port.py              # Task 3: FaithfulnessJudgePort, CompletenessJudgePort
│   │   │   └── technique_promoter_port.py      # Task 6: TechniquePromoterPort
│   │   └── adapters/
│   │       ├── postgres_personal_rd_repo.py    # Task 1: PostgresPersonalRDRepository (personal.* schema)
│   │       ├── citation_firewall_verifier.py   # Task 3: Deterministic anchor + scope/perspective firewall verifier
│   │       ├── llm_evaluation_judges.py        # Task 3: Rubric Faithfulness & Completeness judges + ADR-0009 retry loop
│   │       ├── retrieval_effectiveness_updater.py # Task 4: Closed-loop reliability updater for ServiceDiscovery & memory
│   │       └── technique_promoter_service.py   # Task 6: Sanitization + ApprovalBus + universal.* & ServiceDiscovery bridge
│   ├── multiagent/
│   │   ├── domain/
│   │   │   └── builder_spec.py                 # Task 2: BuilderArtifactKind, BuilderSpec, SandboxVerificationReport
│   │   ├── ports/
│   │   │   └── builder_sandbox_port.py         # Task 2: BuilderSandboxPort
│   │   └── adapters/
│   │       └── sandbox_worktree_runner.py      # Task 2: Jailed sandbox runner (ruff, mypy, boundary check, pytest) + ApprovalBus graduation
│   └── brain/
│       ├── domain/
│       │   └── taxonomy.py                     # Task 5: TaxonomyDefinition, ClassifierPrediction, FeedbackLedgerEntry, GoldenSetEvalReport
│       ├── ports/
│       │   └── classifier_calibration_port.py  # Task 5: ClassifierFeedbackPort, GoldenSetGatePort
│       └── adapters/
│           └── laya_classifier_adapter.py      # Task 5: YAML taxonomy loader, Laya confidence gate, feedback ledger & golden-set gate
└── inventory/
    └── agents/
        ├── domain/
        │   └── builders/
        │       └── builder_workflows.py        # Task 2: tool_builder, skill_builder, agent_builder, app_onboarding handlers
        └── internal/
            └── classifiers/
                └── taxonomies/
                    ├── work_intent.yaml        # Task 5: Single-source routing intent taxonomy
                    ├── epistemic_scope.yaml    # Task 5: 6-value scope taxonomy
                    └── epistemic_perspective.yaml # Task 5: 5-value perspective taxonomy
```

---

### Task 1: Personal R&D Domain Model, Readiness State Machine & `personal.*` Repository

In Work Mode, every note and synthesis is tied to a client engagement (`eng_<slug>.*`) and governed by SOW deliverables. In Personal R&D Mode, the operator needs a low-friction laboratory to explore research papers, prototype prompting or graph-ranking techniques, and record benchmark experiments — without any risk of those half-baked experiments polluting client engagements or Universal Knowledge. Rather than inventing a second knowledge format, we reuse Chotu's hexagonal knowledge contracts inside the dedicated `personal.*` schema and `workspaces/personal_rd/` directory, while enforcing at the domain constructor level that every study and experiment is stamped with `perspective="personal_rd"` and `scope="internal"`.

We also enforce a strict `TechniqueReadiness` state machine (`EXPLORING -> PROTOTYPING -> VALIDATED -> PROMOTED_TO_UNIVERSAL | ARCHIVED`). A study cannot transition to `VALIDATED` unless it has at least one passing `ExperimentRun` whose benchmark metrics beat the baseline, and it cannot transition to `PROMOTED_TO_UNIVERSAL` unless it is already `VALIDATED`. This prevents unverified weekend notes from ever crossing the bridge into Work Mode.

**Files:**
- Create: `backend/platform/knowledge/domain/personal_rd.py`
- Create: `backend/platform/knowledge/ports/personal_rd_port.py`
- Create: `backend/platform/knowledge/adapters/postgres_personal_rd_repo.py`
- Test: `tests/unit/platform/knowledge/test_personal_rd_domain.py`

**Interfaces:**
- Consumes:
  - `backend.platform.knowledge.domain.epistemic.EpistemicScope` (`Literal["engagement", "client", "internal", "reference", "prior-project", "deliverable"]`)
  - `backend.platform.knowledge.domain.epistemic.EpistemicPerspective` (`Literal["client_stated", "internal_team", "personal_take", "work_research", "personal_rd"]`)
- Produces:
  - `TechniqueReadiness(StrEnum)`: `EXPLORING`, `PROTOTYPING`, `VALIDATED`, `PROMOTED_TO_UNIVERSAL`, `ARCHIVED`
  - `ExperimentRun(run_id: str, study_id: str, hypothesis: str, metrics: dict[str, float], beats_baseline: bool, artifact_path: str, created_at: datetime)`
  - `TechniqueStudy(study_id: str, slug: str, title: str, problem_statement: str, readiness: TechniqueReadiness, notebook_dir: str, perspective: str = "personal_rd", scope: str = "internal", experiment_runs: tuple[ExperimentRun, ...] = ())`
  - `TechniqueStudy.record_experiment(run: ExperimentRun) -> TechniqueStudy`
  - `TechniqueStudy.advance_readiness(target: TechniqueReadiness) -> TechniqueStudy`
  - `PersonalRDRepositoryPort.save_study(study: TechniqueStudy) -> None`
  - `PersonalRDRepositoryPort.get_study(study_id: str) -> TechniqueStudy | None`
  - `PersonalRDRepositoryPort.list_by_readiness(readiness: TechniqueReadiness) -> list[TechniqueStudy]`

- [ ] **Step 1: Write the failing test**

```python
# tests/unit/platform/knowledge/test_personal_rd_domain.py
from datetime import datetime, timezone
import pytest

from backend.platform.knowledge.domain.personal_rd import (
    ExperimentRun,
    InvalidReadinessTransitionError,
    PersonalModeIsolationError,
    TechniqueReadiness,
    TechniqueStudy,
)
from backend.platform.knowledge.adapters.postgres_personal_rd_repo import (
    InMemoryPersonalRDRepository,
)


def test_personal_study_enforces_personal_rd_and_blocks_unvalidated_promotion() -> None:
    with pytest.raises(PersonalModeIsolationError, match="perspective='personal_rd'"):
        TechniqueStudy(
            study_id="study_01",
            slug="hits-adr-weighing",
            title="HITS Authority Weighing for ADR Conflicts",
            problem_statement="Weigh competing architecture claims by downstream dependency impact.",
            readiness=TechniqueReadiness.EXPLORING,
            notebook_dir="workspaces/personal_rd/techniques/hits-adr-weighing",
            perspective="client_stated",
            scope="internal",
        )

    study = TechniqueStudy(
        study_id="study_01",
        slug="hits-adr-weighing",
        title="HITS Authority Weighing for ADR Conflicts",
        problem_statement="Weigh competing architecture claims by downstream dependency impact.",
        readiness=TechniqueReadiness.EXPLORING,
        notebook_dir="workspaces/personal_rd/techniques/hits-adr-weighing",
    )
    assert study.perspective == "personal_rd"
    assert study.scope == "internal"

    # Cannot jump directly from EXPLORING to VALIDATED without a baseline-beating ExperimentRun
    with pytest.raises(InvalidReadinessTransitionError, match="passing ExperimentRun"):
        study.advance_readiness(TechniqueReadiness.VALIDATED)

    # Record a prototyping experiment that fails to beat baseline
    failed_run = ExperimentRun(
        run_id="exp_01",
        study_id="study_01",
        hypothesis="Unweighted degree centrality beats HITS hub/authority.",
        metrics={"ndcg_at_5": 0.68, "baseline_ndcg_at_5": 0.74},
        beats_baseline=False,
        artifact_path="workspaces/personal_rd/techniques/hits-adr-weighing/runs/exp_01.json",
        created_at=datetime.now(timezone.utc),
    )
    prototyping_study = study.record_experiment(failed_run)
    assert prototyping_study.readiness == TechniqueReadiness.PROTOTYPING

    with pytest.raises(InvalidReadinessTransitionError, match="beats_baseline=True"):
        prototyping_study.advance_readiness(TechniqueReadiness.VALIDATED)

    # Record a passing experiment run
    passing_run = ExperimentRun(
        run_id="exp_02",
        study_id="study_01",
        hypothesis="2-step Make/Break + Plausibility beats baseline.",
        metrics={"ndcg_at_5": 0.89, "baseline_ndcg_at_5": 0.74},
        beats_baseline=True,
        artifact_path="workspaces/personal_rd/techniques/hits-adr-weighing/runs/exp_02.json",
        created_at=datetime.now(timezone.utc),
    )
    validated_study = prototyping_study.record_experiment(passing_run).advance_readiness(
        TechniqueReadiness.VALIDATED
    )
    assert validated_study.readiness == TechniqueReadiness.VALIDATED


@pytest.mark.asyncio
async def test_personal_rd_repository_roundtrip_and_readiness_filter() -> None:
    repo = InMemoryPersonalRDRepository()
    study = TechniqueStudy(
        study_id="study_02",
        slug="laya-calibration",
        title="Laya Temperature Scaling Calibration",
        problem_statement="Calibrate ModernBERT confidence gates.",
        readiness=TechniqueReadiness.EXPLORING,
        notebook_dir="workspaces/personal_rd/techniques/laya-calibration",
    )
    await repo.save_study(study)
    loaded = await repo.get_study("study_02")
    assert loaded == study
    exploring = await repo.list_by_readiness(TechniqueReadiness.EXPLORING)
    assert [s.study_id for s in exploring] == ["study_02"]
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/unit/platform/knowledge/test_personal_rd_domain.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'backend.platform.knowledge.domain.personal_rd'`

- [ ] **Step 3: Implement `TechniqueStudy`, `PersonalRDRepositoryPort`, and `PostgresPersonalRDRepository`**

In `backend/platform/knowledge/domain/personal_rd.py`, define `TechniqueReadiness(StrEnum)`, `PersonalModeIsolationError(ValueError)`, `InvalidReadinessTransitionError(ValueError)`, `ExperimentRun`, and `TechniqueStudy` as `@dataclass(frozen=True)`. In `TechniqueStudy.__post_init__`, raise `PersonalModeIsolationError` if `self.perspective != "personal_rd"`, `self.scope != "internal"`, or `not self.notebook_dir.startswith("workspaces/personal_rd/")`. In `record_experiment`, append the `ExperimentRun` and transition `EXPLORING -> PROTOTYPING` automatically. In `advance_readiness`, enforce:
- `VALIDATED` requires at least one `ExperimentRun` in `self.experiment_runs` with `beats_baseline is True`.
- `PROMOTED_TO_UNIVERSAL` requires `self.readiness == TechniqueReadiness.VALIDATED`.

In `backend/platform/knowledge/ports/personal_rd_port.py`, define the `PersonalRDRepositoryPort(ABC)` with `save_study`, `get_study`, and `list_by_readiness`. In `backend/platform/knowledge/adapters/postgres_personal_rd_repo.py`, implement `InMemoryPersonalRDRepository` (for fast unit/contract tests) and `PostgresPersonalRDRepository` targeting `personal.technique_studies` and `personal.experiment_runs` with parameterized SQL (`INSERT INTO personal.technique_studies ... ON CONFLICT (study_id) DO UPDATE ...`).

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/unit/platform/knowledge/test_personal_rd_domain.py -v`
Expected: PASS (2 passed)

- [ ] **Step 5: Commit**

```bash
git add backend/platform/knowledge/domain/personal_rd.py \
        backend/platform/knowledge/ports/personal_rd_port.py \
        backend/platform/knowledge/adapters/postgres_personal_rd_repo.py \
        tests/unit/platform/knowledge/test_personal_rd_domain.py
git commit -m "feat(knowledge): add Personal R&D domain model, readiness state machine, and personal.* repo"
```

---

### Task 2: Isolated Builder Sandbox & Self-Extension Builders (`tool_builder`, `skill_builder`, `agent_builder`, `app_onboarding`)

One of Chotu's signature capabilities in Personal Mode (and on capability misses in Work Mode) is building new tools, skills, specialist subagents, and external `dsapps-*` bridges for itself. However, letting an LLM agent write generated code directly into `~/.chotu/` or `backend/inventory/` and register it into the live `ServiceDiscoveryPort` without verification is a recipe for broken imports, hexagonal boundary violations, or runtime crashes.

To make self-extension safe, we route all four builders (`tool_builder`, `skill_builder`, `agent_builder`, `app_onboarding`) through `SandboxWorktreeRunner`. Every build happens inside an isolated sandbox directory (`workspaces/personal_rd/sandbox/<build_id>/`), where `SandboxWorktreeRunner.verify_and_request_graduation()` runs four mandatory checks (`ruff check`, `mypy --strict`, `check_hexagonal_boundaries.py`, and `pytest`). If any check fails, graduation is blocked immediately and no `ApprovalRequest` or `ServiceDiscoveryPort.register()` call is made. Only when all four checks pass does the runner emit an `ApprovalRequest` on `ApprovalBusPort` and, once approved, copy the verified artifact bundle into `~/.chotu/{tools,skills,agents,packs}/<slug>/` and register its `chotu://` URI.

**Files:**
- Create: `backend/platform/multiagent/domain/builder_spec.py`
- Create: `backend/platform/multiagent/ports/builder_sandbox_port.py`
- Create: `backend/platform/multiagent/adapters/sandbox_worktree_runner.py`
- Create: `backend/inventory/agents/domain/builders/builder_workflows.py`
- Test: `tests/unit/platform/multiagent/test_sandbox_builder_graduation.py`

**Interfaces:**
- Consumes:
  - `backend.platform.governance.ports.approval_bus_port.ApprovalBusPort`
  - `backend.platform.discovery.ports.service_discovery_port.ServiceDiscoveryPort`
- Produces:
  - `BuilderArtifactKind(StrEnum)`: `TOOL`, `SKILL`, `AGENT`, `EXTERNAL_APP`
  - `BuilderSpec(build_id: str, kind: BuilderArtifactKind, namespace: str, name: str, version: str, description: str, files: dict[str, str], mode_visibility: tuple[str, ...] = ("personal",))`
  - `SandboxCheckResult(check_name: str, passed: bool, output: str)`
  - `SandboxVerificationReport(build_id: str, capability_uri: str, checks: tuple[SandboxCheckResult, ...], all_passed: bool)`
  - `BuilderSandboxPort.stage_and_verify(spec: BuilderSpec) -> SandboxVerificationReport`
  - `BuilderSandboxPort.graduate(spec: BuilderSpec, report: SandboxVerificationReport) -> str`
  - `BuilderWorkflowService.build_and_graduate(spec: BuilderSpec) -> tuple[SandboxVerificationReport, str | None]`

- [ ] **Step 1: Write the failing test**

```python
# tests/unit/platform/multiagent/test_sandbox_builder_graduation.py
from pathlib import Path
import pytest

from backend.platform.multiagent.domain.builder_spec import (
    BuilderArtifactKind,
    BuilderGraduationError,
    BuilderSpec,
    SandboxCheckResult,
)
from backend.platform.multiagent.adapters.sandbox_worktree_runner import (
    SandboxWorktreeRunner,
)
from backend.inventory.agents.domain.builders.builder_workflows import (
    BuilderWorkflowService,
)


class FakeApprovalBus:
    def __init__(self, approve: bool = True) -> None:
        self.approve = approve
        self.requests: list[dict[str, object]] = []

    async def request_approval(self, payload: dict[str, object]) -> bool:
        self.requests.append(payload)
        return self.approve


class FakeServiceDiscovery:
    def __init__(self) -> None:
        self.registered_uris: list[str] = []

    async def register_capability(self, uri: str, manifest: dict[str, object]) -> None:
        self.registered_uris.append(uri)


@pytest.mark.asyncio
async def test_sandbox_builder_blocks_registration_when_boundary_or_pytest_fails(
    tmp_path: Path,
) -> None:
    approval_bus = FakeApprovalBus(approve=True)
    discovery = FakeServiceDiscovery()

    async def failing_check_executor(sandbox_dir: Path) -> list[SandboxCheckResult]:
        return [
            SandboxCheckResult("ruff", True, "All checks passed!"),
            SandboxCheckResult("mypy", True, "Success: no issues found"),
            SandboxCheckResult(
                "hexagonal_boundaries",
                False,
                "Forbidden import 'litellm' inside domain module",
            ),
            SandboxCheckResult("pytest", True, "1 passed"),
        ]

    runner = SandboxWorktreeRunner(
        sandbox_root=tmp_path / "sandbox",
        install_root=tmp_path / "chotu_home",
        check_executor=failing_check_executor,
    )
    service = BuilderWorkflowService(
        sandbox=runner,
        approval_bus=approval_bus,
        service_discovery=discovery,
    )

    bad_tool_spec = BuilderSpec(
        build_id="build_bad_01",
        kind=BuilderArtifactKind.TOOL,
        namespace="personal_rd",
        name="wardley_parser",
        version="1.0.0",
        description="Parse Wardley map DSL into nodes and edges.",
        files={
            "manifest.yaml": "uri: chotu://tool/personal_rd/wardley_parser@1.0.0\n",
            "handler.py": "import litellm\n",
            "tests/test_handler.py": "def test_noop(): assert True\n",
        },
    )

    report, graduated_uri = await service.build_and_graduate(bad_tool_spec)
    assert report.all_passed is False
    assert graduated_uri is None
    assert approval_bus.requests == []
    assert discovery.registered_uris == []

    with pytest.raises(BuilderGraduationError, match="Failed checks: hexagonal_boundaries"):
        await runner.graduate(bad_tool_spec, report)


@pytest.mark.asyncio
async def test_sandbox_builder_graduates_and_registers_when_all_gates_and_approval_pass(
    tmp_path: Path,
) -> None:
    approval_bus = FakeApprovalBus(approve=True)
    discovery = FakeServiceDiscovery()

    async def passing_check_executor(sandbox_dir: Path) -> list[SandboxCheckResult]:
        return [
            SandboxCheckResult("ruff", True, "All checks passed!"),
            SandboxCheckResult("mypy", True, "Success: no issues found"),
            SandboxCheckResult("hexagonal_boundaries", True, "0 violations"),
            SandboxCheckResult("pytest", True, "2 passed"),
        ]

    runner = SandboxWorktreeRunner(
        sandbox_root=tmp_path / "sandbox",
        install_root=tmp_path / "chotu_home",
        check_executor=passing_check_executor,
    )
    service = BuilderWorkflowService(
        sandbox=runner,
        approval_bus=approval_bus,
        service_discovery=discovery,
    )

    good_tool_spec = BuilderSpec(
        build_id="build_good_01",
        kind=BuilderArtifactKind.TOOL,
        namespace="personal_rd",
        name="wardley_parser",
        version="1.0.0",
        description="Parse Wardley map DSL into nodes and edges.",
        files={
            "manifest.yaml": "uri: chotu://tool/personal_rd/wardley_parser@1.0.0\n",
            "handler.py": "def parse_map(text: str) -> dict[str, list[str]]: return {'nodes': []}\n",
            "tests/test_handler.py": "from handler import parse_map\ndef test_parse(): assert parse_map('') == {'nodes': []}\n",
        },
    )

    report, graduated_uri = await service.build_and_graduate(good_tool_spec)
    assert report.all_passed is True
    assert graduated_uri == "chotu://tool/personal_rd/wardley_parser@1.0.0"
    assert len(approval_bus.requests) == 1
    assert discovery.registered_uris == ["chotu://tool/personal_rd/wardley_parser@1.0.0"]
    installed_manifest = (
        tmp_path / "chotu_home" / "tools" / "wardley_parser" / "manifest.yaml"
    )
    assert installed_manifest.exists()
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/unit/platform/multiagent/test_sandbox_builder_graduation.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'backend.platform.multiagent.domain.builder_spec'`

- [ ] **Step 3: Implement `BuilderSpec`, `SandboxWorktreeRunner`, and `BuilderWorkflowService`**

In `backend/platform/multiagent/domain/builder_spec.py`, define `BuilderArtifactKind(StrEnum)` (`TOOL="tool"`, `SKILL="skill"`, `AGENT="agent"`, `EXTERNAL_APP="app"`), `BuilderGraduationError(RuntimeError)`, `SandboxCheckResult`, `SandboxVerificationReport`, and `BuilderSpec` with property `capability_uri -> str` returning `f"chotu://{self.kind.value}/{self.namespace}/{self.name}@{self.version}"`. Reject any relative path in `BuilderSpec.files` containing `..` or starting with `/` to prevent path traversal out of the sandbox.

In `backend/platform/multiagent/adapters/sandbox_worktree_runner.py`, implement `SandboxWorktreeRunner(BuilderSandboxPort)`:
- `stage_and_verify(spec)` writes all `spec.files` into `sandbox_root / spec.build_id`, invokes `check_executor(sandbox_dir)` (which defaults to running `ruff check`, `mypy --strict`, `scripts/check_hexagonal_boundaries.py`, and `pytest -q` via `asyncio.create_subprocess_exec`), and returns `SandboxVerificationReport` with `all_passed = all(c.passed for c in checks)` and confirm all 4 required check names (`{"ruff", "mypy", "hexagonal_boundaries", "pytest"}`) are present.
- `graduate(spec, report)` raises `BuilderGraduationError` if `not report.all_passed`; otherwise copies the staged files into `install_root / f"{spec.kind.value}s" / spec.name` and returns `spec.capability_uri`.

In `backend/inventory/agents/domain/builders/builder_workflows.py`, implement `BuilderWorkflowService.build_and_graduate(spec)` which calls `sandbox.stage_and_verify(spec)`, returns `(report, None)` immediately if `not report.all_passed`, requests operator confirmation on `approval_bus.request_approval(...)`, and upon approval calls `sandbox.graduate(spec, report)` followed by `service_discovery.register_capability(uri, ...)`.

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/unit/platform/multiagent/test_sandbox_builder_graduation.py -v`
Expected: PASS (2 passed)

- [ ] **Step 5: Commit**

```bash
git add backend/platform/multiagent/domain/builder_spec.py \
        backend/platform/multiagent/ports/builder_sandbox_port.py \
        backend/platform/multiagent/adapters/sandbox_worktree_runner.py \
        backend/inventory/agents/domain/builders/builder_workflows.py \
        tests/unit/platform/multiagent/test_sandbox_builder_graduation.py
git commit -m "feat(builders): add jailed SandboxWorktreeRunner and 4-gate graduation workflow"
```

---

### Task 3: Deterministic Citation & Scope Firewall Verifier + Epistemic Evaluation Judges (`FaithfulnessJudgePort` & `CompletenessJudgePort`)

When a BAITS specialist or synthesis agent produces an answer, how do we verify that it didn't hallucinate an anchor, leak a `personal_rd` or `internal` note into a client deliverable, or extrapolate beyond the retrieved facts? Harvested from `okf-kb-manager` (including ADR-0009's bounded corrective loop), our evaluation pipeline runs in two stages. First, `CitationFirewallVerifier` performs a zero-LLM deterministic scan of every `[[concept#^f-id]]` and `engagement://<id>#^<anchor>` citation in the draft synthesis against the `ContextPacket`, immediately rejecting fabricated anchors or epistemic scope/perspective leaks (`personal_rd` or `internal` cited inside a `client` or `deliverable` target scope).

Second, `FaithfulnessJudgePort` and `CompletenessJudgePort` evaluate whether substantive claims are strictly entailed by the cited facts and whether all required BAITS dimensions were addressed. If faithfulness falls below threshold (e.g., `< 0.85`), `FaithfulnessCorrectiveLoop` feeds the exact unsupported claims back to the synthesizer for up to `max_retries = 2` corrective passes. If the synthesis still fails after exhausting `max_retries`, the loop halts deterministically and attaches `unverified_claim_warning=True` alongside the remaining unsupported claims so the operator is explicitly warned.

**Files:**
- Create: `backend/platform/knowledge/domain/evaluation.py`
- Create: `backend/platform/knowledge/ports/evaluation_port.py`
- Create: `backend/platform/knowledge/adapters/citation_firewall_verifier.py`
- Create: `backend/platform/knowledge/adapters/llm_evaluation_judges.py`
- Test: `tests/unit/platform/knowledge/test_epistemic_evaluation_judges.py`

**Interfaces:**
- Consumes:
  - `ContextFact(anchor: str, text: str, scope: str, perspective: str)`
- Produces:
  - `CitationViolation(anchor: str, reason: Literal["missing_anchor", "scope_firewall_breach", "perspective_firewall_breach"])`
  - `CitationVerificationResult(is_valid: bool, cited_anchors: tuple[str, ...], violations: tuple[CitationViolation, ...])`
  - `FaithfulnessVerdict(score: float, passed: bool, unsupported_claims: tuple[str, ...], critique: str)`
  - `CompletenessVerdict(score: float, passed: bool, missing_dimensions: tuple[str, ...], required_baits_pillars: tuple[str, ...])`
  - `CorrectiveSynthesisOutcome(final_text: str, faithfulness: FaithfulnessVerdict, completeness: CompletenessVerdict, citation_check: CitationVerificationResult, attempts_used: int, unverified_claim_warning: bool)`
  - `CitationFirewallVerifier.verify(text: str, context_facts: dict[str, ContextFact], target_scope: str) -> CitationVerificationResult`
  - `FaithfulnessJudgePort.evaluate(text: str, context_facts: dict[str, ContextFact]) -> FaithfulnessVerdict`
  - `CompletenessJudgePort.evaluate(query: str, text: str, required_pillars: tuple[str, ...]) -> CompletenessVerdict`
  - `FaithfulnessCorrectiveLoop.run(...) -> CorrectiveSynthesisOutcome`

- [ ] **Step 1: Write the failing test**

```python
# tests/unit/platform/knowledge/test_epistemic_evaluation_judges.py
import pytest

from backend.platform.knowledge.domain.evaluation import (
    CompletenessVerdict,
    ContextFact,
    FaithfulnessVerdict,
)
from backend.platform.knowledge.adapters.citation_firewall_verifier import (
    CitationFirewallVerifier,
)
from backend.platform.knowledge.adapters.llm_evaluation_judges import (
    FaithfulnessCorrectiveLoop,
)


def test_citation_firewall_blocks_personal_rd_and_fabricated_anchors() -> None:
    verifier = CitationFirewallVerifier()
    context_facts = {
        "payment-hub#^f-latency": ContextFact(
            anchor="payment-hub#^f-latency",
            text="P99 latency is 420ms.",
            scope="client",
            perspective="client_stated",
        ),
        "hits-study#^f-exp1": ContextFact(
            anchor="hits-study#^f-exp1",
            text="Experimental HITS dampener 0.85.",
            scope="internal",
            perspective="personal_rd",
        ),
    }

    draft = (
        "Payment hub latency is 420ms [[payment-hub#^f-latency]], "
        "weighed using [[hits-study#^f-exp1]] and [[ghost-concept#^f-999]]."
    )
    result = verifier.verify(
        text=draft,
        context_facts=context_facts,
        target_scope="deliverable",
    )
    assert result.is_valid is False
    reasons = {(v.anchor, v.reason) for v in result.violations}
    assert ("hits-study#^f-exp1", "perspective_firewall_breach") in reasons
    assert ("hits-study#^f-exp1", "scope_firewall_breach") in reasons
    assert ("ghost-concept#^f-999", "missing_anchor") in reasons


@pytest.mark.asyncio
async def test_faithfulness_loop_retries_with_feedback_and_halts_at_max_retries() -> None:
    context_facts = {
        "payment-hub#^f-latency": ContextFact(
            anchor="payment-hub#^f-latency",
            text="P99 latency is 420ms.",
            scope="client",
            perspective="client_stated",
        )
    }

    class StubFaithfulnessJudge:
        async def evaluate(
            self, text: str, facts: dict[str, ContextFact]
        ) -> FaithfulnessVerdict:
            if "99.999% uptime" in text:
                return FaithfulnessVerdict(
                    score=0.45,
                    passed=False,
                    unsupported_claims=("Claim of 99.999% uptime is not in ContextPacket",),
                    critique="Remove ungrounded uptime SLA claim.",
                )
            return FaithfulnessVerdict(
                score=0.96,
                passed=True,
                unsupported_claims=(),
                critique="All claims grounded.",
            )

    class StubCompletenessJudge:
        async def evaluate(
            self, query: str, text: str, required_pillars: tuple[str, ...]
        ) -> CompletenessVerdict:
            return CompletenessVerdict(
                score=0.92,
                passed=True,
                missing_dimensions=(),
                required_baits_pillars=required_pillars,
            )

    # Case 1: Synthesizer fixes hallucination on retry 1
    prompts_seen: list[str] = []

    async def recovering_synthesizer(feedback: str | None) -> str:
        prompts_seen.append(feedback or "")
        if feedback is None:
            return "Latency is 420ms [[payment-hub#^f-latency]] with 99.999% uptime."
        return "Latency is 420ms [[payment-hub#^f-latency]]."

    loop = FaithfulnessCorrectiveLoop(
        citation_verifier=CitationFirewallVerifier(),
        faithfulness_judge=StubFaithfulnessJudge(),
        completeness_judge=StubCompletenessJudge(),
        max_retries=2,
    )
    outcome = await loop.run(
        query="What is the payment hub latency?",
        synthesizer=recovering_synthesizer,
        context_facts=context_facts,
        target_scope="deliverable",
        required_pillars=("technology",),
    )
    assert outcome.attempts_used == 2
    assert outcome.faithfulness.passed is True
    assert outcome.unverified_claim_warning is False
    assert "99.999% uptime" in prompts_seen[1]

    # Case 2: Stubborn synthesizer never fixes claim -> halts after max_retries=2 (3 total attempts)
    async def stubborn_synthesizer(feedback: str | None) -> str:
        return "Latency is 420ms [[payment-hub#^f-latency]] with 99.999% uptime."

    failed_outcome = await loop.run(
        query="What is the payment hub latency?",
        synthesizer=stubborn_synthesizer,
        context_facts=context_facts,
        target_scope="deliverable",
        required_pillars=("technology",),
    )
    assert failed_outcome.attempts_used == 3
    assert failed_outcome.faithfulness.passed is False
    assert failed_outcome.unverified_claim_warning is True
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/unit/platform/knowledge/test_epistemic_evaluation_judges.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'backend.platform.knowledge.domain.evaluation'`

- [ ] **Step 3: Implement `CitationFirewallVerifier`, `FaithfulnessJudgePort`, `CompletenessJudgePort`, and `FaithfulnessCorrectiveLoop`**

In `backend/platform/knowledge/domain/evaluation.py`, define `ContextFact`, `CitationViolation`, `CitationVerificationResult`, `FaithfulnessVerdict`, `CompletenessVerdict`, and `CorrectiveSynthesisOutcome`.

In `backend/platform/knowledge/adapters/citation_firewall_verifier.py`, use regex `r"\[\[([^\[\]]+#\^[a-zA-Z0-9_-]+)\]\]"` (plus `r"engagement://([a-zA-Z0-9_-]+#\^[a-zA-Z0-9_-]+)"`) to extract all cited anchors in order. For each anchor:
- If not in `context_facts`, record `CitationViolation(anchor, "missing_anchor")`.
- If `target_scope in {"client", "deliverable"}` and `fact.scope == "internal"`, record `CitationViolation(anchor, "scope_firewall_breach")`.
- If `target_scope in {"client", "deliverable", "engagement"}` and `fact.perspective == "personal_rd"`, record `CitationViolation(anchor, "perspective_firewall_breach")`.

In `backend/platform/knowledge/adapters/llm_evaluation_judges.py`, implement `FaithfulnessCorrectiveLoop.run()` which runs up to `1 + self.max_retries` attempts. On each attempt, it verifies citations and evaluates faithfulness; if either fails and retries remain, it builds a structured critique string listing the exact violation reasons and `unsupported_claims` and invokes `synthesizer(feedback)`. If exhausted, it returns `CorrectiveSynthesisOutcome(..., unverified_claim_warning=True)`.

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/unit/platform/knowledge/test_epistemic_evaluation_judges.py -v`
Expected: PASS (2 passed)

- [ ] **Step 5: Commit**

```bash
git add backend/platform/knowledge/domain/evaluation.py \
        backend/platform/knowledge/ports/evaluation_port.py \
        backend/platform/knowledge/adapters/citation_firewall_verifier.py \
        backend/platform/knowledge/adapters/llm_evaluation_judges.py \
        tests/unit/platform/knowledge/test_epistemic_evaluation_judges.py
git commit -m "feat(evals): add CitationFirewallVerifier and ADR-0009 FaithfulnessCorrectiveLoop"
```

---

### Task 4: Closed-Loop `RetrievalTrace` Effectiveness Scorer (`RetrievalEffectivenessUpdater`)

Retrieving capabilities (`chotu://` tools, skills, subagents) and knowledge concepts (`okf-wiki/` sections) is only half the battle; the system also needs to learn which retrieved items actually help produce faithful, operator-approved answers and which ones are dead weight or hallucination triggers. Whenever a turn finishes, Chotu logs a `RetrievalTraceRecord` capturing what was retrieved into the `ContextPacket`, what was actually cited in the verified output, the `FaithfulnessVerdict` and `CompletenessVerdict` scores, how many corrective retries were needed, and any explicit operator feedback (`"approved" | "edited" | "rejected" | None`).

`RetrievalEffectivenessUpdater` processes these traces and updates each capability's `historical_reliability_score` (bounded in `[0.05, 0.99]`) using an exponentially weighted moving average (`alpha = 0.20`). Capabilities that were cited in a faithful, first-pass synthesis (or explicitly approved by the operator) receive a positive reward signal, while capabilities that triggered corrective retries, failed faithfulness, or were rejected by the operator are decayed so Stage 3 of `ServiceDiscoveryPort` and the Small-to-Big Librarian automatically self-correct over time.

**Files:**
- Create: `backend/platform/knowledge/domain/retrieval_effectiveness.py`
- Create: `backend/platform/knowledge/adapters/retrieval_effectiveness_updater.py`
- Test: `tests/unit/platform/knowledge/test_retrieval_effectiveness_updater.py`

**Interfaces:**
- Consumes:
  - `FaithfulnessVerdict`, `CompletenessVerdict` from Task 3
- Produces:
  - `RetrievalTraceRecord(trace_id: str, turn_id: str, retrieved_capability_uris: tuple[str, ...], cited_capability_uris: tuple[str, ...], retrieved_concept_anchors: tuple[str, ...], cited_concept_anchors: tuple[str, ...], faithfulness_score: float, completeness_score: float, retries_used: int, operator_signal: Literal["approved", "edited", "rejected"] | None = None)`
  - `CapabilityReliabilityUpdate(uri: str, previous_score: float, new_score: float, delta: float, reason: str)`
  - `RetrievalEffectivenessUpdater.compute_updates(trace: RetrievalTraceRecord, current_scores: dict[str, float]) -> list[CapabilityReliabilityUpdate]`

- [ ] **Step 1: Write the failing test**

```python
# tests/unit/platform/knowledge/test_retrieval_effectiveness_updater.py
from backend.platform.knowledge.domain.retrieval_effectiveness import (
    RetrievalTraceRecord,
)
from backend.platform.knowledge.adapters.retrieval_effectiveness_updater import (
    RetrievalEffectivenessUpdater,
)


def test_retrieval_effectiveness_updater_boosts_cited_and_decays_penalized_capabilities() -> None:
    updater = RetrievalEffectivenessUpdater(alpha=0.20)
    current_scores = {
        "chotu://skill/csa/bvf@1.0.0": 0.70,
        "chotu://skill/csa/noisy-draft@1.0.0": 0.70,
    }

    # Turn 1: bvf skill is cited, 0 retries, high faithfulness + operator approved;
    # noisy-draft was retrieved into context but never cited.
    good_trace = RetrievalTraceRecord(
        trace_id="trace_01",
        turn_id="turn_01",
        retrieved_capability_uris=(
            "chotu://skill/csa/bvf@1.0.0",
            "chotu://skill/csa/noisy-draft@1.0.0",
        ),
        cited_capability_uris=("chotu://skill/csa/bvf@1.0.0",),
        retrieved_concept_anchors=("payment-hub#^f-latency",),
        cited_concept_anchors=("payment-hub#^f-latency",),
        faithfulness_score=0.95,
        completeness_score=0.90,
        retries_used=0,
        operator_signal="approved",
    )
    updates_1 = {
        u.uri: u for u in updater.compute_updates(good_trace, current_scores)
    }
    assert updates_1["chotu://skill/csa/bvf@1.0.0"].new_score > 0.70
    assert updates_1["chotu://skill/csa/noisy-draft@1.0.0"].new_score < 0.70

    # Turn 2: noisy-draft is used in a turn that required 2 corrective retries and was rejected
    bad_trace = RetrievalTraceRecord(
        trace_id="trace_02",
        turn_id="turn_02",
        retrieved_capability_uris=("chotu://skill/csa/noisy-draft@1.0.0",),
        cited_capability_uris=("chotu://skill/csa/noisy-draft@1.0.0",),
        retrieved_concept_anchors=("payment-hub#^f-latency",),
        cited_concept_anchors=(),
        faithfulness_score=0.40,
        completeness_score=0.50,
        retries_used=2,
        operator_signal="rejected",
    )
    updates_2 = {
        u.uri: u for u in updater.compute_updates(bad_trace, current_scores)
    }
    assert updates_2["chotu://skill/csa/noisy-draft@1.0.0"].new_score < 0.60
    assert updates_2["chotu://skill/csa/noisy-draft@1.0.0"].delta < -0.10
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/unit/platform/knowledge/test_retrieval_effectiveness_updater.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'backend.platform.knowledge.domain.retrieval_effectiveness'`

- [ ] **Step 3: Implement `RetrievalTraceRecord` and `RetrievalEffectivenessUpdater`**

In `backend/platform/knowledge/domain/retrieval_effectiveness.py`, define `RetrievalTraceRecord` and `CapabilityReliabilityUpdate`.

In `backend/platform/knowledge/adapters/retrieval_effectiveness_updater.py`, implement `RetrievalEffectivenessUpdater(alpha: float = 0.20)`. For each `uri` in `trace.retrieved_capability_uris`:
- Compute target observation score `obs`:
  - If `uri in trace.cited_capability_uris`: base `obs = 0.6 * trace.faithfulness_score + 0.4 * trace.completeness_score - (0.15 * trace.retries_used)`.
  - If `uri not in trace.cited_capability_uris`: `obs = max(0.10, previous_score - 0.15)` (mild decay for retrieved-but-unused noise).
  - Apply operator signal modifier: if `trace.operator_signal == "approved"`, `obs = min(1.0, obs + 0.15)`; if `"rejected"`, `obs = 0.05`; if `"edited"`, `obs = max(0.10, obs - 0.10)`.
- Clamp `obs` to `[0.05, 0.99]`, compute `new_score = round( clamp((1 - alpha) * previous_score + alpha * obs, 0.05, 0.99), 4 )`, and return `CapabilityReliabilityUpdate`.

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/unit/platform/knowledge/test_retrieval_effectiveness_updater.py -v`
Expected: PASS (1 passed)

- [ ] **Step 5: Commit**

```bash
git add backend/platform/knowledge/domain/retrieval_effectiveness.py \
        backend/platform/knowledge/adapters/retrieval_effectiveness_updater.py \
        tests/unit/platform/knowledge/test_retrieval_effectiveness_updater.py
git commit -m "feat(knowledge): add closed-loop RetrievalTrace effectiveness scorer"
```

---

### Task 5: Single-Source YAML Taxonomies, `LayaClassifierAdapter` Confidence Gate, Feedback Ledger & Golden-Set Distillation Gate

Harvested from `classifier-training`, Chotu avoids burning slow, expensive autoregressive LLM calls on every single fixed-taxonomy routing or epistemic tagging decision. Instead, single-source YAML files under `backend/inventory/agents/internal/classifiers/taxonomies/` define the canonical labels (`work_intent.yaml`, `epistemic_scope.yaml`, `epistemic_perspective.yaml`), and `LayaClassifierAdapter` evaluates incoming text against a provenance-verified Laya encoder checkpoint (`run.json` + calibrated confidence threshold). When the checkpoint's top probability meets or exceeds the calibrated gate, System-1 routing completes immediately; when confidence falls below the gate, the adapter escalates to the System-2 teacher and logs the example into `core.classifier_feedback_ledger`.

Equally important, whenever the human operator corrects a label (or when we distill a new Laya checkpoint in Personal R&D Mode), `GoldenSetPromotionGate` evaluates the candidate checkpoint against the human-verified golden set. A candidate checkpoint can only replace the active champion if its macro-F1 is $\ge$ the champion's macro-F1 and its Expected Calibration Error (`ece`) is $\le 0.05` — guaranteeing that classifier updates never silently regress routing accuracy.

**Files:**
- Create: `backend/inventory/agents/internal/classifiers/taxonomies/work_intent.yaml`
- Create: `backend/inventory/agents/internal/classifiers/taxonomies/epistemic_scope.yaml`
- Create: `backend/inventory/agents/internal/classifiers/taxonomies/epistemic_perspective.yaml`
- Create: `backend/platform/brain/domain/taxonomy.py`
- Create: `backend/platform/brain/ports/classifier_calibration_port.py`
- Create: `backend/platform/brain/adapters/laya_classifier_adapter.py`
- Test: `tests/unit/platform/brain/test_laya_classifier_and_golden_gate.py`

**Interfaces:**
- Consumes:
  - Single-source YAML taxonomy files in `backend/inventory/agents/internal/classifiers/taxonomies/*.yaml`
- Produces:
  - `TaxonomyDefinition(name: str, version: str, confidence_threshold: float, labels: tuple[str, ...])`
  - `ClassifierPrediction(taxonomy_name: str, label: str, confidence: float, routed_tier: Literal["system_1_laya", "system_2_teacher"], checkpoint_id: str)`
  - `FeedbackLedgerEntry(entry_id: str, taxonomy_name: str, input_text: str, predicted_label: str, corrected_label: str, source: Literal["operator_override", "system_2_escalation"], created_at: datetime)`
  - `CheckpointMetrics(checkpoint_id: str, macro_f1: float, ece: float, latency_p95_ms: float)`
  - `GoldenSetEvalReport(candidate: CheckpointMetrics, champion: CheckpointMetrics, promoted: bool, rejection_reason: str | None)`
  - `LayaClassifierAdapter.classify(taxonomy_name: str, text: str) -> ClassifierPrediction`
  - `LayaClassifierAdapter.record_operator_correction(taxonomy_name: str, text: str, predicted_label: str, corrected_label: str) -> FeedbackLedgerEntry`
  - `GoldenSetPromotionGate.evaluate_candidate(candidate: CheckpointMetrics, champion: CheckpointMetrics, max_ece: float = 0.05) -> GoldenSetEvalReport`

- [ ] **Step 1: Write the failing test**

```python
# tests/unit/platform/brain/test_laya_classifier_and_golden_gate.py
from pathlib import Path
import pytest

from backend.platform.brain.domain.taxonomy import (
    CheckpointMetrics,
    UnknownTaxonomyLabelError,
)
from backend.platform.brain.adapters.laya_classifier_adapter import (
    GoldenSetPromotionGate,
    LayaClassifierAdapter,
    load_yaml_taxonomy,
)


def test_yaml_taxonomies_load_canonical_scope_and_perspective_labels() -> None:
    base_dir = Path("backend/inventory/agents/internal/classifiers/taxonomies")
    scope_tax = load_yaml_taxonomy(base_dir / "epistemic_scope.yaml")
    assert scope_tax.labels == (
        "engagement",
        "client",
        "internal",
        "reference",
        "prior-project",
        "deliverable",
    )

    persp_tax = load_yaml_taxonomy(base_dir / "epistemic_perspective.yaml")
    assert persp_tax.labels == (
        "client_stated",
        "internal_team",
        "personal_take",
        "work_research",
        "personal_rd",
    )


@pytest.mark.asyncio
async def test_laya_classifier_confidence_gate_and_operator_feedback_ledger() -> None:
    base_dir = Path("backend/inventory/agents/internal/classifiers/taxonomies")
    persp_tax = load_yaml_taxonomy(base_dir / "epistemic_perspective.yaml")

    def fake_laya_encoder(text: str) -> tuple[str, float]:
        if "Sunday arXiv experiment" in text:
            return ("personal_rd", 0.94)
        return ("client_stated", 0.52)  # below 0.80 threshold

    async def fake_system2_teacher(text: str, labels: tuple[str, ...]) -> str:
        return "internal_team"

    adapter = LayaClassifierAdapter(
        taxonomies={"epistemic_perspective": persp_tax},
        checkpoint_id="laya-mmbert-v1.2",
        encoder_fn=fake_laya_encoder,
        teacher_fn=fake_system2_teacher,
    )

    # High-confidence System-1 fast path
    pred_fast = await adapter.classify(
        "epistemic_perspective", "My Sunday arXiv experiment on HITS damping"
    )
    assert pred_fast.label == "personal_rd"
    assert pred_fast.routed_tier == "system_1_laya"
    assert len(adapter.feedback_ledger) == 0

    # Low-confidence System-2 teacher fallback -> automatically logged to feedback ledger
    pred_fallback = await adapter.classify(
        "epistemic_perspective", "Ambiguous slack thread from architect"
    )
    assert pred_fallback.label == "internal_team"
    assert pred_fallback.routed_tier == "system_2_teacher"
    assert len(adapter.feedback_ledger) == 1
    assert adapter.feedback_ledger[0].source == "system_2_escalation"

    # Explicit operator correction rejects out-of-taxonomy label
    with pytest.raises(UnknownTaxonomyLabelError):
        adapter.record_operator_correction(
            "epistemic_perspective",
            "Note text",
            predicted_label="client_stated",
            corrected_label="not_a_valid_perspective",
        )


def test_laya_golden_set_gate_rejects_regressed_checkpoint() -> None:
    gate = GoldenSetPromotionGate()
    champion = CheckpointMetrics(
        checkpoint_id="laya-mmbert-v1.2",
        macro_f1=0.91,
        ece=0.031,
        latency_p95_ms=28.0,
    )
    regressed_f1 = CheckpointMetrics(
        checkpoint_id="laya-mmbert-v1.3-bad",
        macro_f1=0.89,
        ece=0.029,
        latency_p95_ms=27.0,
    )
    uncalibrated = CheckpointMetrics(
        checkpoint_id="laya-mmbert-v1.3-uncal",
        macro_f1=0.92,
        ece=0.072,
        latency_p95_ms=27.5,
    )
    worthy_challenger = CheckpointMetrics(
        checkpoint_id="laya-mmbert-v1.4",
        macro_f1=0.93,
        ece=0.024,
        latency_p95_ms=26.0,
    )

    rep_regressed = gate.evaluate_candidate(regressed_f1, champion)
    assert rep_regressed.promoted is False
    assert "macro_f1" in (rep_regressed.rejection_reason or "")

    rep_uncal = gate.evaluate_candidate(uncalibrated, champion)
    assert rep_uncal.promoted is False
    assert "ece" in (rep_uncal.rejection_reason or "")

    rep_good = gate.evaluate_candidate(worthy_challenger, champion)
    assert rep_good.promoted is True
    assert rep_good.rejection_reason is None
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/unit/platform/brain/test_laya_classifier_and_golden_gate.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'backend.platform.brain.domain.taxonomy'`

- [ ] **Step 3: Implement YAML taxonomies, `LayaClassifierAdapter`, and `GoldenSetPromotionGate`**

Create the three YAML taxonomy files in `backend/inventory/agents/internal/classifiers/taxonomies/`:
- `epistemic_scope.yaml`: `name: epistemic_scope`, `version: "1.0.0"`, `confidence_threshold: 0.80`, `labels: [engagement, client, internal, reference, prior-project, deliverable]`.
- `epistemic_perspective.yaml`: `name: epistemic_perspective`, `version: "1.0.0"`, `confidence_threshold: 0.80`, `labels: [client_stated, internal_team, personal_take, work_research, personal_rd]`.
- `work_intent.yaml`: `name: work_intent`, `version: "1.0.0"`, `confidence_threshold: 0.82`, `labels: [engagement_synthesis, knowledge_ingestion, truth_weighing, personal_rd_study, capability_build, briefing_prep]`.

In `backend/platform/brain/domain/taxonomy.py`, define `UnknownTaxonomyLabelError(ValueError)`, `TaxonomyDefinition`, `ClassifierPrediction`, `FeedbackLedgerEntry`, `CheckpointMetrics`, and `GoldenSetEvalReport`.

In `backend/platform/brain/adapters/laya_classifier_adapter.py`, implement `load_yaml_taxonomy(path: Path) -> TaxonomyDefinition`, `LayaClassifierAdapter`, and `GoldenSetPromotionGate`. In `LayaClassifierAdapter.classify()`, check that the predicted label is in `taxonomy.labels`; if `confidence >= taxonomy.confidence_threshold`, return `routed_tier="system_1_laya"`. Otherwise invoke `teacher_fn(text, taxonomy.labels)`, append a `FeedbackLedgerEntry(source="system_2_escalation", ...)` to `self.feedback_ledger`, and return `routed_tier="system_2_teacher"`.

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/unit/platform/brain/test_laya_classifier_and_golden_gate.py -v`
Expected: PASS (3 passed)

- [ ] **Step 5: Commit**

```bash
git add backend/inventory/agents/internal/classifiers/taxonomies/work_intent.yaml \
        backend/inventory/agents/internal/classifiers/taxonomies/epistemic_scope.yaml \
        backend/inventory/agents/internal/classifiers/taxonomies/epistemic_perspective.yaml \
        backend/platform/brain/domain/taxonomy.py \
        backend/platform/brain/ports/classifier_calibration_port.py \
        backend/platform/brain/adapters/laya_classifier_adapter.py \
        tests/unit/platform/brain/test_laya_classifier_and_golden_gate.py
git commit -m "feat(brain): add YAML taxonomies, LayaClassifierAdapter confidence gate, and GoldenSetPromotionGate"
```

---

### Task 6: `TechniquePromoter` Bridge (`personal.*` $\rightarrow$ `universal.*` & `ServiceDiscoveryPort`)

When a Personal R&D technique study reaches `TechniqueReadiness.VALIDATED` and its companion skill or tool passes sandbox verification, the operator needs an intentional, audited bridge to promote that capability so Work Mode engagements (`eng_<slug>`) can discover and use it. Just as client engagements never mutate one another directly, Personal R&D never writes directly into an engagement schema. Instead, `TechniquePromoterService` packages the validated technique into Universal Knowledge (`universal.*`) with `scope="reference"` and `perspective="work_research"`, stamped with provenance `personal_rd://<study_id>#^<anchor>`, and updates any companion capability's `mode_visibility` to `("personal", "work")`.

Before presenting the promotion package on `ApprovalBusPort`, `PromotionSanitizer` scans the markdown concepts, facts, and tool manifests to ensure no raw personal scratch paths (`workspaces/personal_rd/scratch/`), local home paths (`/home/`), or forbidden client identifiers accidentally slipped into the reference package. Once the operator approves the promotion on `ApprovalBusPort`, `TechniquePromoterService` indexes the package into `universal.*`, updates the capability visibility in `ServiceDiscoveryPort`, and advances the study's state in `PersonalRDRepositoryPort` to `TechniqueReadiness.PROMOTED_TO_UNIVERSAL`.

**Files:**
- Create: `backend/platform/knowledge/domain/technique_promotion.py`
- Create: `backend/platform/knowledge/ports/technique_promoter_port.py`
- Create: `backend/platform/knowledge/adapters/technique_promoter_service.py`
- Test: `tests/unit/platform/knowledge/test_technique_promoter_bridge.py`

**Interfaces:**
- Consumes:
  - `TechniqueStudy`, `TechniqueReadiness`, `PersonalRDRepositoryPort` from Task 1
  - `SandboxVerificationReport` from Task 2
- Produces:
  - `PromotionSanitizationError(ValueError)`
  - `PromotionCandidate(study: TechniqueStudy, concept_slug: str, distilled_summary: str, atomic_facts: tuple[tuple[str, str], ...], companion_capability_uri: str | None = None, sandbox_report: SandboxVerificationReport | None = None)`
  - `UniversalKnowledgeEntry(concept_slug: str, summary: str, facts: dict[str, str], scope: str, perspective: str, provenance_citations: tuple[str, ...])`
  - `PromotionResult(study_id: str, promoted_readiness: TechniqueReadiness, universal_concept_slug: str, capability_uri: str | None, mode_visibility: tuple[str, ...])`
  - `TechniquePromoterPort.promote(candidate: PromotionCandidate) -> PromotionResult`

- [ ] **Step 1: Write the failing test**

```python
# tests/unit/platform/knowledge/test_technique_promoter_bridge.py
from datetime import datetime, timezone
import pytest

from backend.platform.knowledge.domain.personal_rd import (
    ExperimentRun,
    InvalidReadinessTransitionError,
    TechniqueReadiness,
    TechniqueStudy,
)
from backend.platform.knowledge.adapters.postgres_personal_rd_repo import (
    InMemoryPersonalRDRepository,
)
from backend.platform.knowledge.domain.technique_promotion import (
    PromotionCandidate,
    PromotionSanitizationError,
)
from backend.platform.knowledge.adapters.technique_promoter_service import (
    TechniquePromoterService,
)


class FakeApprovalBus:
    def __init__(self, approve: bool = True) -> None:
        self.approve = approve

    async def request_approval(self, payload: dict[str, object]) -> bool:
        return self.approve


class FakeUniversalKnowledgeStore:
    def __init__(self) -> None:
        self.entries: dict[str, object] = {}

    async def upsert_reference_concept(self, entry: object) -> None:
        self.entries[getattr(entry, "concept_slug")] = entry


class FakeServiceDiscovery:
    def __init__(self) -> None:
        self.visibility_by_uri: dict[str, tuple[str, ...]] = {}

    async def update_mode_visibility(
        self, uri: str, mode_visibility: tuple[str, ...]
    ) -> None:
        self.visibility_by_uri[uri] = mode_visibility


@pytest.mark.asyncio
async def test_technique_promoter_rejects_unvalidated_or_unsanitized_study_and_promotes_approved() -> None:
    repo = InMemoryPersonalRDRepository()
    uk_store = FakeUniversalKnowledgeStore()
    discovery = FakeServiceDiscovery()
    promoter = TechniquePromoterService(
        personal_repo=repo,
        universal_store=uk_store,
        service_discovery=discovery,
        approval_bus=FakeApprovalBus(approve=True),
    )

    exploring_study = TechniqueStudy(
        study_id="study_wardley",
        slug="wardley-value-chain",
        title="Wardley Value-Chain Extraction",
        problem_statement="Extract anchor need and evolution axis from transcripts.",
        readiness=TechniqueReadiness.EXPLORING,
        notebook_dir="workspaces/personal_rd/techniques/wardley-value-chain",
    )
    await repo.save_study(exploring_study)

    # 1. Rejects unvalidated study
    with pytest.raises(InvalidReadinessTransitionError, match="VALIDATED"):
        await promoter.promote(
            PromotionCandidate(
                study=exploring_study,
                concept_slug="wardley-mapping-extraction",
                distilled_summary="Extract value chain components ordered by evolution.",
                atomic_facts=(("f-1", "Anchor user need sits at the top of the value chain."),),
            )
        )

    # Advance study to VALIDATED
    passing_run = ExperimentRun(
        run_id="exp_w1",
        study_id="study_wardley",
        hypothesis="Two-stage dependency parser extracts value chain with >0.85 recall.",
        metrics={"recall": 0.91, "baseline_recall": 0.72},
        beats_baseline=True,
        artifact_path="workspaces/personal_rd/techniques/wardley-value-chain/runs/exp_w1.json",
        created_at=datetime.now(timezone.utc),
    )
    validated_study = exploring_study.record_experiment(passing_run).advance_readiness(
        TechniqueReadiness.VALIDATED
    )
    await repo.save_study(validated_study)

    # 2. Rejects unsanitized scratch path or local filesystem path
    with pytest.raises(PromotionSanitizationError, match="unsanitized path"):
        await promoter.promote(
            PromotionCandidate(
                study=validated_study,
                concept_slug="wardley-mapping-extraction",
                distilled_summary="See scratch notes in /home/nox/scratch/todo.md",
                atomic_facts=(("f-1", "Anchor user need sits at the top."),),
            )
        )

    # 3. Promotes sanitized VALIDATED study into universal.* and enables work visibility
    result = await promoter.promote(
        PromotionCandidate(
            study=validated_study,
            concept_slug="wardley-mapping-extraction",
            distilled_summary="Extract value chain components ordered by evolution stage.",
            atomic_facts=(
                ("f-anchor", "Anchor user need sits at the top of the value chain."),
                ("f-axis", "Evolution axis spans Genesis, Custom, Product, Commodity."),
            ),
            companion_capability_uri="chotu://skill/personal_rd/wardley-extract@1.0.0",
        )
    )

    assert result.promoted_readiness == TechniqueReadiness.PROMOTED_TO_UNIVERSAL
    assert result.mode_visibility == ("personal", "work")
    assert discovery.visibility_by_uri["chotu://skill/personal_rd/wardley-extract@1.0.0"] == (
        "personal",
        "work",
    )

    uk_entry = uk_store.entries["wardley-mapping-extraction"]
    assert getattr(uk_entry, "scope") == "reference"
    assert getattr(uk_entry, "perspective") == "work_research"
    assert getattr(uk_entry, "provenance_citations") == (
        "personal_rd://study_wardley#^f-anchor",
        "personal_rd://study_wardley#^f-axis",
    )

    persisted_study = await repo.get_study("study_wardley")
    assert persisted_study is not None
    assert persisted_study.readiness == TechniqueReadiness.PROMOTED_TO_UNIVERSAL
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/unit/platform/knowledge/test_technique_promoter_bridge.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'backend.platform.knowledge.domain.technique_promotion'`

- [ ] **Step 3: Implement `PromotionSanitizer` and `TechniquePromoterService`**

In `backend/platform/knowledge/domain/technique_promotion.py`, define `PromotionSanitizationError(ValueError)`, `PromotionCandidate`, `UniversalKnowledgeEntry`, `PromotionResult`, and `PromotionSanitizer.validate(candidate: PromotionCandidate) -> None`:
- Check `candidate.distilled_summary` and every fact text in `candidate.atomic_facts` for forbidden substrings (`"/home/"`, `"workspaces/personal_rd/scratch"`, `"TODO:"`, `"FIXME:"`, `"eng_"`). If any match, raise `PromotionSanitizationError("Promotion candidate contains unsanitized path or scratch marker")`.

In `backend/platform/knowledge/adapters/technique_promoter_service.py`, implement `TechniquePromoterService.promote(candidate: PromotionCandidate) -> PromotionResult`:
1. Verify `candidate.study.readiness == TechniqueReadiness.VALIDATED` (otherwise call `candidate.study.advance_readiness(TechniqueReadiness.PROMOTED_TO_UNIVERSAL)` which raises `InvalidReadinessTransitionError`).
2. Run `PromotionSanitizer.validate(candidate)`.
3. Request operator approval via `approval_bus.request_approval(...)`.
4. Build `UniversalKnowledgeEntry` with `scope="reference"`, `perspective="work_research"`, and `provenance_citations=tuple(f"personal_rd://{candidate.study.study_id}#^{fid}" for fid, _ in candidate.atomic_facts)`, and persist via `universal_store.upsert_reference_concept(entry)`.
5. If `candidate.companion_capability_uri` is provided, call `service_discovery.update_mode_visibility(uri, ("personal", "work"))`.
6. Advance the study to `TechniqueReadiness.PROMOTED_TO_UNIVERSAL`, save it in `personal_repo`, and return `PromotionResult`.

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/unit/platform/knowledge/test_technique_promoter_bridge.py -v`
Expected: PASS (1 passed)

- [ ] **Step 5: Commit**

```bash
git add backend/platform/knowledge/domain/technique_promotion.py \
        backend/platform/knowledge/ports/technique_promoter_port.py \
        backend/platform/knowledge/adapters/technique_promoter_service.py \
        tests/unit/platform/knowledge/test_technique_promoter_bridge.py
git commit -m "feat(knowledge): add TechniquePromoter sanitization and personal-to-universal promotion bridge"
```

---

## Self-Review Checklist Verification

1. **Spec coverage**:
   - Workstream 5.1 (`personal.*` schema, `perspective: personal_rd`, `TechniqueReadiness` lifecycle `EXPLORING -> PROTOTYPING -> VALIDATED -> PROMOTED_TO_UNIVERSAL`): Covered by **Task 1**.
   - Workstream 5.2 (Self-extension builders `tool_builder`, `skill_builder`, `agent_builder`, `app_onboarding` + jailed sandbox `ruff`, `mypy --strict`, `check_hexagonal_boundaries.py`, `pytest` gates + `ApprovalBus` graduation): Covered by **Task 2**.
   - Workstream 5.3 (Deterministic Citation & Scope Firewall Verifier, `FaithfulnessJudgePort` with ADR-0009 bounded corrective retry loop, `CompletenessJudgePort`, and closed-loop `RetrievalTrace` effectiveness updater): Covered by **Task 3** and **Task 4**.
   - Workstream 5.4 (Single-source YAML taxonomies, `LayaClassifierAdapter` confidence gate, `core.classifier_feedback_ledger`, and Golden-Set promotion gate): Covered by **Task 5**.
   - Workstream 5.5 (`TechniquePromoter` sanitization, `ApprovalBus` confirmation, `universal.*` indexing with `personal_rd://<study_id>#^<anchor>` provenance, and `mode_visibility: ("personal", "work")` promotion): Covered by **Task 6**.
2. **Step scan & zero placeholders**: Every step across Tasks 1–6 has complete runnable `pytest` code, exact CLI verification commands, exact type signatures, and zero `TODO` or placeholder stubs.
3. **Type consistency**: `TechniqueStudy` and `TechniqueReadiness` defined in Task 1 are consumed identically in Task 6; `SandboxVerificationReport` from Task 2 and `FaithfulnessVerdict`/`CompletenessVerdict` from Task 3 match their consumers in Tasks 4 and 6.
4. **Review Focus coverage**: All 5 failure modes listed in `## Review Focus` are explicitly pinned to named tests across Tasks 1, 2, 3, 4, 5, and 6.
