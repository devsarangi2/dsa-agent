# Slice Group 4: Unified Chotu Web Control Center (frontend/) & End-to-End Verification Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the unified React 19 + TypeScript Web Control Center (`frontend/`) that consolidates the application shell and trace drawer from `dsa-prj`, the knowledge curation and evaluation studio from `okf-kb-manager`, and the executive check-in/journal workflows from `dsapps-checkin` and `dsapps-journal`—augmented with the 4-Lane Queue Monitor, Engagement Master Data Explorer, and Truth Finder + HITS Conflict Resolution Studio—and verify the entire platform end-to-end with unit, port-contract, integration, and Playwright UAT suites.

**Architecture:** The frontend is a single-page React 19 + TypeScript application (`frontend/src/`) organized around a scope-aware `ScopeContext` and typed HTTP/SSE client (`frontend/src/lib/api-client.ts`) that binds every request to an explicit operating mode (`work` | `personal` | `universal`) and engagement schema (`eng_<slug>`) via `X-Chotu-Mode` and `X-Chotu-Engagement-Scope` headers. Feature modules (`features/`) and top-level workbenches (`pages/executive/`, `pages/projects/`, `pages/studio/`, `pages/evals/`, `pages/settings/`) consume server state through `@tanstack/react-query` keys prefixed by `[mode, engagementSlug]` and receive real-time `WorkflowNodeRun`, `ApprovalBus`, and 4-lane queue updates over a cursor-resumable SSE hook (`useChotuEventStream`).

**Tech Stack:** React 19, TypeScript 5.7 (`strict: true`), Vite 6, Tailwind CSS v4, `shadcn/ui` (Radix UI primitives + Lucide icons), `@tanstack/react-query` v5, Vitest + React Testing Library + MSW for unit/component TDD, and Playwright + pytest (`httpx.AsyncClient` + PostgreSQL 16 / `pgvector`) for full-stack E2E verification.

**Spec:** [`docs/planning/slice-04-unified-web-portal-and-verification.md`](file:///home/nox/Developer/ds2/dsa-agent/docs/planning/slice-04-unified-web-portal-and-verification.md) and [`docs/planning/chotu_architecture_plan.md`](file:///home/nox/Developer/ds2/dsa-agent/docs/planning/chotu_architecture_plan.md)

## Global Constraints

- **Strict Scope Boundary Headers on Every Request**: Every REST and SSE call to `/api/v1/*` must attach `X-Chotu-Mode: work | personal | universal` and, when in `work` mode, `X-Chotu-Engagement-Scope: eng_<slug>`. Requests in `work` mode without a valid `eng_<slug>` must throw a client-side `ScopeIsolationError` before touching the network.
- **6-Value `scope:` & 5-Value `perspective:` Taxonomies**: All frontend filters, badges, and forms must enforce the exact 6-value `FactScope` (`engagement` | `client` | `internal` | `reference` | `prior-project` | `deliverable`) and 5-value `FactPerspective` (`client_stated` | `internal_team` | `personal_take` | `work_research` | `personal_rd`).
- **Subagents-Only Visualization & Two Runtime Chokepoints**: `PlanCard` renders per-node `WorkflowNodeRun` DAG states (`PENDING` | `READY` | `QUEUED` | `RUNNING` | `AWAITING_APPROVAL` | `COMPLETED` | `FAILED` | `CANCELLED`) and never shows `lead/chotu` invoking raw tools directly; `TraceDrawer` renders the 4-level span hierarchy (`turn -> workflow_node -> subagent -> tool`).
- **No Forced Auto-Conflict Overwrite**: Truth Finder conflict cards must always present both competing claims with their HITS Authority/Hub scores, Step 1 Make/Break relationship counts, and Step 2 Plausibility rationale, requiring explicit human action (`promote_a` | `promote_b` | `keep_both_tracked` | `draft_clarifying_question`).
- **Read-Only Sibling Repositories**: Component patterns harvested from `/home/nox/Developer/ds2/dsa-prj`, `/home/nox/Developer/ds2/okf-kb-manager`, `/home/nox/Developer/ds2/dsapps-checkin`, and `/home/nox/Developer/ds2/dsapps-journal` are adapted into `/home/nox/Developer/ds2/dsa-agent/frontend/`; never modify sibling repositories.

## Review Focus

1. **Cross-Scope Cache & Header Leakage on Mode Switch**: When the operator switches `ModeSwitcher` from `Work: eng_acme` to `Personal R&D` (or `Work: eng_beta`), cached React Query data from `eng_acme` could flash on screen or an in-flight request could carry the stale `X-Chotu-Engagement-Scope: eng_acme` header; switching scope must synchronously cancel in-flight queries, evict non-matching scope cache keys, and omit `X-Chotu-Engagement-Scope` in `personal` mode (pinned in **Task 1**, `test_scope_switch_cancels_inflight_and_clears_previous_engagement_cache`).
2. **SSE Reconnection Dropping In-Flight `WorkflowNodeRun` State Transitions**: When a brief network blip drops the `/api/v1/stream/events` connection while parallel BAITS subagents are transitioning from `RUNNING` to `AWAITING_APPROVAL` or `COMPLETED`, reconnecting without `last_event_id` leaves `PlanCard` stuck showing stale `RUNNING` spinners forever; `useChotuEventStream` must reconnect with `?last_event_id=<id>` and deduplicate replayed events in topological order (pinned in **Task 1**, `test_sse_reconnect_passes_last_event_id_and_deduplicates_node_transitions`).
3. **Premature Promotion of Unverified `CANDIDATE` Workflows**: In the Executive Cockpit, a newly composed `CANDIDATE` workflow with fewer than 3 verified runs (or with a failed verification run) must not allow one-click promotion to `ESTABLISHED` without an explicit override confirmation reason, otherwise broken System-2 plans corrupt the System-1 Fast Path (pinned in **Task 3**, `test_probationary_workflow_blocks_direct_promotion_below_three_verified_runs`).
4. **Silent Orphaned Downstream Artifacts When Promoting a Conflict Option That Breaks Relationships**: In the Truth Finder Conflict Resolution Studio, clicking `Promote Option B` when Option B has `breaks_count > 0` (e.g., breaking `ADR-004` and a Milestone M3 deliverable) must not silently resolve the conflict without queuing staleness flags and displaying the affected downstream Master Data / BAITS artifacts in the confirmation payload (pinned in **Task 5**, `test_promoting_option_with_broken_relationships_requires_staleness_acknowledgment_and_flags_artifacts`).
5. **Unsanitized Client PII Leaking into Universal Knowledge During Retrospective Elevation**: In the Retrospective Elevation Workbench, if a candidate learning card still contains flagged client-identifiable tokens (`sanitation_status === "flagged"`) or lacks a valid `engagement://<engagement_id>#^<anchor>` correlation URI, the `Approve & Elevate to Universal` action must remain disabled until the operator edits/sanitizes the text and re-validates (pinned in **Task 6**, `test_retrospective_elevation_blocks_submission_when_pii_flagged_or_citation_missing`).

---

## File Structure Map

```text
frontend/
├── package.json
├── tsconfig.json
├── vite.config.ts
├── playwright.config.ts
├── src/
│   ├── types/
│   │   └── chotu.ts                                  # Shared domain contracts (Mode, Scope, DAG, Queue, MasterData, TruthFinder, UK)
│   ├── lib/
│   │   ├── scope-context.tsx                         # ScopeProvider, useScope(), query cache eviction on scope change
│   │   ├── api-client.ts                             # createChotuApiClient() with X-Chotu-Mode & X-Chotu-Engagement-Scope guards
│   │   └── use-chotu-event-stream.ts                 # Cursor-resumable SSE hook (?last_event_id=...) + event deduplication
│   ├── app/
│   │   ├── AppShell.tsx                              # 3-zone resizable shell (Left Rail, Center Workbench, Right Orchestrator/Trace Dock)
│   │   ├── ModeSwitcher.tsx                          # Work (eng_<slug>) vs. Personal R&D vs. Universal Studio selector
│   │   └── CommandPalette.tsx                        # Cmd+K quick navigation, scope switching & workflow dispatch
│   ├── features/
│   │   ├── conversation/
│   │   │   ├── PlanCard.tsx                          # Live per-node WorkflowNodeRun DAG visualizer (parallel fan-out/fan-in)
│   │   │   └── ApprovalCard.tsx                      # Inline HITL gate card wired to P0 interactive ApprovalBus resume
│   │   ├── governance/
│   │   │   └── PolicyEditor.tsx                      # Declarative allowlist, workspace jail & 6-value scope citeability editor
│   │   └── observability/
│   │       └── TraceDrawer.tsx                       # 4-level OTel span tree (turn -> workflow_node -> subagent -> tool)
│   └── pages/
│       ├── executive/
│       │   ├── ExecutiveCockpitPage.tsx              # CEO landing page composing Briefing, Queue Monitor, DAG Promotion & Check-in/Journal
│       │   ├── ExecutiveBriefingCard.tsx             # Tier-3 Subconscious Daily Executive Briefing with conflict deep-links
│       │   ├── FourLaneQueueMonitor.tsx              # Real-time P0-P3 lane metrics, preemption indicator & outbox dead-letter replay
│       │   ├── ProbationaryWorkflowCard.tsx          # CANDIDATE -> ESTABLISHED promotion gate (3 verified runs rule)
│       │   └── ExecutiveRhythmPanel.tsx              # Harvested dsapps-checkin & dsapps-journal bridge views
│       ├── projects/
│       │   ├── EngagementWorkspacePage.tsx           # Per-engagement (eng_<slug>) control room
│       │   ├── EngagementMasterDataExplorer.tsx      # 7 relational Master Data tables linked to [[concept#^f-id]]
│       │   ├── MultiPerspectiveFactExplorer.tsx      # 6-value scope: & 5-value perspective: facet bar + bitemporal timeline
│       │   ├── TruthFinderConflictCard.tsx           # Side-by-side HITS scores, Step 1 Make/Break graph, Step 2 rationale & 4 HITL actions
│       │   └── MakeBreakImpactGraph.tsx              # Visual node/edge impact diff highlighting validated vs. broken downstream items
│       ├── studio/
│       │   ├── UniversalStudioPage.tsx               # Disjoint universal.* shelf, ConceptTree, CitationGraphView & Discover->Select->Ingest
│       │   ├── CitationDrillDownDrawer.tsx           # Two-stage engagement://<id>#^<anchor> provenance drawer
│       │   └── RetrospectiveElevationWorkbench.tsx   # 5-Lens closure queue with PII sanitation guard & one-click elevation
│       └── evals/
│           └── EvalsPage.tsx                         # Small-to-Big RetrievalTrace weights, faithfulness judge scores & cost/latency panels
├── tests/
│   ├── setup.ts
│   ├── lib/
│   │   └── api-and-stream.test.tsx
│   ├── app/
│   │   └── shell-and-execution.test.tsx
│   ├── pages/
│   │   ├── executive-cockpit.test.tsx
│   │   ├── engagement-workspace.test.tsx
│   │   ├── truth-finder-studio.test.tsx
│   │   └── universal-studio-and-evals.test.tsx
└── e2e/
    └── chotu-portal-uat.spec.ts                      # 4 full-stack Playwright UAT consulting & R&D scenarios
backend/tests/e2e/
└── test_portal_full_stack_scenarios.py               # Backend + Postgres + pgvector E2E contract & concurrency verification
```

---

### Task 1: Workspace Foundation, Scope-Isolated API Client & Cursor-Resumable SSE Stream Hook

Before we render a single visual card, we need the foundational plumbing that enforces Chotu's hardest architectural rule in the browser: **strict isolation between client engagements (`eng_<slug>`), Personal R&D (`personal.*`), and Universal Knowledge (`universal.*`)**, paired with a resilient real-time event stream for `WorkflowNodeRun` and 4-lane queue updates. If a component fetches `/api/v1/facts` without knowing which scope is active—or if switching from `eng_acme` to `Personal R&D` leaves stale `eng_acme` facts sitting in the React Query cache—client-confidential data will bleed across views.

In this task, we define the shared TypeScript domain contracts in `frontend/src/types/chotu.ts`, build `ScopeProvider` (`frontend/src/lib/scope-context.tsx`) so that every scope transition cancels in-flight requests and evicts stale scope keys, implement `createChotuApiClient` (`frontend/src/lib/api-client.ts`) to inject `X-Chotu-Mode` and `X-Chotu-Engagement-Scope` on every request (throwing `ScopeIsolationError` if `work` mode lacks an `eng_<slug>`), and create `useChotuEventStream` (`frontend/src/lib/use-chotu-event-stream.ts`) with `?last_event_id=<cursor>` reconnection so network blips never drop subagent node transitions.

**Files:**
- Create: `frontend/package.json`
- Create: `frontend/tsconfig.json`
- Create: `frontend/vite.config.ts`
- Create: `frontend/src/types/chotu.ts`
- Create: `frontend/src/lib/scope-context.tsx`
- Create: `frontend/src/lib/api-client.ts`
- Create: `frontend/src/lib/use-chotu-event-stream.ts`
- Create: `frontend/tests/setup.ts`
- Test: `frontend/tests/lib/api-and-stream.test.tsx`

**Interfaces:**
- Consumes: Browser `fetch`, `EventSource`, `@tanstack/react-query` (`QueryClient`, `QueryClientProvider`).
- Produces:
```typescript
// frontend/src/types/chotu.ts
export type OperatingMode = "work" | "personal" | "universal";
export type EngagementSlug = `eng_${string}`;

export type FactScope =
  | "engagement"
  | "client"
  | "internal"
  | "reference"
  | "prior-project"
  | "deliverable";

export type FactPerspective =
  | "client_stated"
  | "internal_team"
  | "personal_take"
  | "work_research"
  | "personal_rd";

export type BaitsPillar =
  | "Business"
  | "Application"
  | "Information"
  | "Technology"
  | "Security";

export type WorkflowNodeStatus =
  | "PENDING"
  | "READY"
  | "QUEUED"
  | "RUNNING"
  | "AWAITING_APPROVAL"
  | "COMPLETED"
  | "FAILED"
  | "CANCELLED";

export type PriorityLaneId =
  | "P0_interactive"
  | "P1_workflow_dag"
  | "P2_outbox_hooks"
  | "P3_subconscious_bulk";

export interface ScopeState {
  mode: OperatingMode;
  engagementSlug: EngagementSlug | null;
}

export interface ChotuStreamEvent {
  eventId: string;
  type: "workflow_node_transition" | "queue_snapshot" | "approval_requested" | "outbox_alert";
  timestamp: string;
  payload: Record<string, unknown>;
}

// frontend/src/lib/api-client.ts
export class ScopeIsolationError extends Error {
  constructor(message: string);
}
export interface ChotuApiClient {
  get<T>(path: string, params?: Record<string, string | string[] | undefined>): Promise<T>;
  post<T>(path: string, body?: unknown): Promise<T>;
}
export function createChotuApiClient(getScope: () => ScopeState, baseUrl?: string): ChotuApiClient;

// frontend/src/lib/scope-context.tsx
export interface ScopeContextValue extends ScopeState {
  switchScope: (next: ScopeState) => Promise<void>;
}
export function ScopeProvider(props: {
  queryClient: import("@tanstack/react-query").QueryClient;
  initialScope?: ScopeState;
  children: React.ReactNode;
}): React.JSX.Element;
export function useScope(): ScopeContextValue;

// frontend/src/lib/use-chotu-event-stream.ts
export function useChotuEventStream(options: {
  streamUrl: string;
  scope: ScopeState;
  onEvent: (event: ChotuStreamEvent) => void;
   eventSourceFactory?: (url: string) => EventSource;
}): { connected: boolean; lastEventId: string | null; reconnect: () => void };
```

- [ ] **Step 1: Write the failing test in `frontend/tests/lib/api-and-stream.test.tsx`**

```tsx
import { describe, it, expect, vi } from "vitest";
import { renderHook, act } from "@testing-library/react";
import React from "react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import {
  createChotuApiClient,
  ScopeIsolationError,
} from "../../src/lib/api-client";
import { ScopeProvider, useScope } from "../../src/lib/scope-context";
import { useChotuEventStream } from "../../src/lib/use-chotu-event-stream";
import type { ChotuStreamEvent, ScopeState } from "../../src/types/chotu";

describe("Task 1: Scope-Isolated API Client & Cursor-Resumable SSE Stream", () => {
  it("test_work_mode_without_engagement_slug_throws_scope_isolation_error", async () => {
    const fetchSpy = vi.spyOn(globalThis, "fetch");
    const client = createChotuApiClient(() => ({
      mode: "work",
      engagementSlug: null,
    }));

    await expect(client.get("/api/v1/projects/facts")).rejects.toThrow(
      ScopeIsolationError
    );
    expect(fetchSpy).not.toHaveBeenCalled();
    fetchSpy.mockRestore();
  });

  it("test_scope_switch_cancels_inflight_and_clears_previous_engagement_cache", async () => {
    const queryClient = new QueryClient();
    queryClient.setQueryData(["work", "eng_acme", "facts"], [{ id: "f-101" }]);
    const cancelSpy = vi.spyOn(queryClient, "cancelQueries");
    const removeSpy = vi.spyOn(queryClient, "removeQueries");

    const wrapper = ({ children }: { children: React.ReactNode }) => (
      <QueryClientProvider client={queryClient}>
        <ScopeProvider
          queryClient={queryClient}
          initialScope={{ mode: "work", engagementSlug: "eng_acme" }}
        >
          {children}
        </ScopeProvider>
      </QueryClientProvider>
    );

    const { result } = renderHook(() => useScope(), { wrapper });
    expect(result.current.engagementSlug).toBe("eng_acme");

    await act(async () => {
      await result.current.switchScope({
        mode: "personal",
        engagementSlug: null,
      });
    });

    expect(cancelSpy).toHaveBeenCalled();
    expect(removeSpy).toHaveBeenCalledWith({
      queryKey: ["work", "eng_acme"],
    });
    expect(queryClient.getQueryData(["work", "eng_acme", "facts"])).toBeUndefined();
    expect(result.current.mode).toBe("personal");
    expect(result.current.engagementSlug).toBeNull();
  });

  it("test_sse_reconnect_passes_last_event_id_and_deduplicates_node_transitions", () => {
    const createdUrls: string[] = [];
    const listeners: Record<string, ((ev: MessageEvent) => void)[]> = {};
    let currentMockSource: {
      emitMessage: (data: ChotuStreamEvent) => void;
      close: () => void;
    } | null = null;

    const fakeFactory = (url: string) => {
      createdUrls.push(url);
      listeners["message"] = [];
      const instance = {
        addEventListener: (type: string, cb: (ev: MessageEvent) => void) => {
          listeners[type] = listeners[type] || [];
          listeners[type].push(cb);
        },
        removeEventListener: vi.fn(),
        close: vi.fn(),
        emitMessage: (payload: ChotuStreamEvent) => {
          const msg = new MessageEvent("message", {
            data: JSON.stringify(payload),
            lastEventId: payload.eventId,
          });
          (listeners["message"] || []).forEach((cb) => cb(msg));
        },
      };
      currentMockSource = instance;
      return instance as unknown as EventSource;
    };

    const received: ChotuStreamEvent[] = [];
    const scope: ScopeState = { mode: "work", engagementSlug: "eng_acme" };

    const { result } = renderHook(() =>
      useChotuEventStream({
        streamUrl: "/api/v1/stream/events",
        scope,
        onEvent: (ev) => received.push(ev),
        eventSourceFactory: fakeFactory,
      })
    );

    expect(createdUrls[0]).toBe(
      "/api/v1/stream/events?mode=work&engagement_slug=eng_acme"
    );

    act(() => {
      currentMockSource!.emitMessage({
        eventId: "evt-42",
        type: "workflow_node_transition",
        timestamp: "2026-10-28T09:00:00Z",
        payload: { nodeId: "tech_arch", status: "RUNNING" },
      });
    });

    expect(result.current.lastEventId).toBe("evt-42");
    expect(received).toHaveLength(1);

    // Simulate reconnect after transient network blip
    act(() => {
      result.current.reconnect();
    });

    expect(createdUrls[1]).toBe(
      "/api/v1/stream/events?mode=work&engagement_slug=eng_acme&last_event_id=evt-42"
    );

    // Replay of evt-42 must be deduplicated; evt-43 must be delivered
    act(() => {
      currentMockSource!.emitMessage({
        eventId: "evt-42",
        type: "workflow_node_transition",
        timestamp: "2026-10-28T09:00:00Z",
        payload: { nodeId: "tech_arch", status: "RUNNING" },
      });
      currentMockSource!.emitMessage({
        eventId: "evt-43",
        type: "workflow_node_transition",
        timestamp: "2026-10-28T09:00:02Z",
        payload: { nodeId: "tech_arch", status: "COMPLETED" },
      });
    });

    expect(received).toHaveLength(2);
    expect(received[1].eventId).toBe("evt-43");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd frontend && npx vitest run tests/lib/api-and-stream.test.tsx`
Expected: FAIL with module resolution error (`Cannot find module '../../src/lib/api-client'`).

- [ ] **Step 3: Implement `frontend/src/types/chotu.ts`, `frontend/src/lib/api-client.ts`, `frontend/src/lib/scope-context.tsx`, and `frontend/src/lib/use-chotu-event-stream.ts`**

Scaffold `frontend/package.json` with `react@^19.0.0`, `react-dom@^19.0.0`, `@tanstack/react-query@^5.62.0`, `lucide-react`, `vitest`, `@testing-library/react`, `@testing-library/jest-dom`, `jsdom`, and `@playwright/test`. In `createChotuApiClient`, validate that whenever `scope.mode === "work"`, `scope.engagementSlug` matches `/^eng_[a-z0-9_-]+$/` or throw `ScopeIsolationError`; attach `X-Chotu-Mode: <mode>` on all requests and `X-Chotu-Engagement-Scope: <engagementSlug>` only when `mode === "work"`. In `ScopeProvider.switchScope(next)`, call `await queryClient.cancelQueries({ queryKey: [prev.mode, prev.engagementSlug] })` followed by `queryClient.removeQueries({ queryKey: [prev.mode, prev.engagementSlug] })` before updating state. In `useChotuEventStream`, maintain a `seenEventIds = useRef<Set<string>>(new Set())` and `lastEventIdRef = useRef<string | null>(null)` so calling `reconnect()` appends `&last_event_id=${lastEventIdRef.current}` and ignores any replayed `eventId` already present in `seenEventIds.current`.

- [ ] **Step 4: Run test to verify it passes**

Run: `cd frontend && npx vitest run tests/lib/api-and-stream.test.tsx`
Expected: PASS (3 passed).

- [ ] **Step 5: Commit**

```bash
git add frontend/package.json frontend/tsconfig.json frontend/vite.config.ts frontend/tests/setup.ts frontend/src/types/chotu.ts frontend/src/lib/api-client.ts frontend/src/lib/scope-context.tsx frontend/src/lib/use-chotu-event-stream.ts frontend/tests/lib/api-and-stream.test.tsx
git commit -m "feat(frontend): add scope-isolated API client, ScopeProvider, and cursor-resumable SSE hook"
```

---

### Task 2: Application Shell, Command Palette & Live Execution Components (`AppShell`, `ModeSwitcher`, `PlanCard`, `ApprovalCard`, `PolicyEditor`, `TraceDrawer`)

With scope isolation and SSE streaming in place, Task 2 builds the outer application frame harvested from [`dsa-prj`](file:///home/nox/Developer/ds2/dsa-prj) (`AppShell`, `ModeSwitcher`, `CommandPalette`) alongside the live execution governance components (`PlanCard`, `ApprovalCard`, `PolicyEditor`, and `TraceDrawer`).

Two design invariants shape these components. First, because `lead/chotu` obeys the **Subagents-Only Invariant**, `PlanCard` never displays a flat list of raw tool calls on the orchestrator; instead, it renders the directed acyclic graph of specialist subagent `WorkflowNodeRun` items—including parallel fan-out across BAITS architects (`business-architect` $\parallel$ `technology-architect` $\parallel$ `security-architect`) and fan-in synthesis (`csa-scribe`)—updating each node's status badge live as SSE events arrive. Second, `TraceDrawer` makes the two hexagonal chokepoints (`AgentRuntime.invoke()` and `ToolRegistry.invoke()`) tangible by rendering the exact 4-level OpenTelemetry span tree (`turn -> workflow_node -> subagent -> tool`) along with the `ServiceLocatorPort` resolution stage (`Stage 1 Exact URI`, `Stage 2 Faceted Filter`, or `Stage 3 Hybrid Ranker`) and any blocking/observer hooks.

**Files:**
- Modify: `frontend/src/types/chotu.ts`
- Create: `frontend/src/app/AppShell.tsx`
- Create: `frontend/src/app/ModeSwitcher.tsx`
- Create: `frontend/src/app/CommandPalette.tsx`
- Create: `frontend/src/features/conversation/PlanCard.tsx`
- Create: `frontend/src/features/conversation/ApprovalCard.tsx`
- Create: `frontend/src/features/governance/PolicyEditor.tsx`
- Create: `frontend/src/features/observability/TraceDrawer.tsx`
- Test: `frontend/tests/app/shell-and-execution.test.tsx`

**Interfaces:**
- Consumes: `ScopeContextValue`, `WorkflowNodeStatus`, `BaitsPillar`, `FactScope` from Task 1.
- Produces:
```typescript
// Additions to frontend/src/types/chotu.ts
export interface WorkClassificationBadge {
  intent: string;
  mode: OperatingMode;
  engagementSlug: EngagementSlug | null;
  departments: string[];
  baitsPillars: BaitsPillar[];
  scope: FactScope;
  perspective: FactPerspective;
  pathType: "SYSTEM_1_ESTABLISHED" | "SYSTEM_2_CANDIDATE";
}

export interface WorkflowNodeRunView {
  nodeId: string;
  subagentUri: `chotu://agent/${string}@${string}`;
  displayName: string;
  baitsPillar?: BaitsPillar;
  dependsOn: string[];
  status: WorkflowNodeStatus;
  outputArtifactRefs: string[];
}

export interface WorkflowPlanView {
  workflowRunId: string;
  origin: "USER" | "HOOK" | "SUBCONSCIOUS";
  hopDepth: number;
  classification: WorkClassificationBadge;
  nodes: WorkflowNodeRunView[];
}

export interface ApprovalRequestView {
  approvalId: string;
  workflowRunId: string;
  nodeId: string;
  subagentUri: string;
  requestedAction: string;
  riskLevel: "low" | "medium" | "high" | "critical";
  policyRuleId: string;
}

export interface TraceSpanNode {
  spanId: string;
  parentSpanId: string | null;
  kind: "turn" | "workflow_node" | "hook" | "subagent" | "tool";
  name: string;
  capabilityUri?: string;
  locatorStage?: "STAGE_1_EXACT_URI" | "STAGE_2_FACET_FILTER" | "STAGE_3_HYBRID_RANK";
  workspaceJailPath?: string;
  durationMs: number;
  tokensUsed?: number;
  modelTier?: string;
  children: TraceSpanNode[];
}

// Component Signatures
export function ModeSwitcher(props: {
  availableEngagements: EngagementSlug[];
}): React.JSX.Element;

export function PlanCard(props: {
  plan: WorkflowPlanView;
  onSelectNodeTrace: (nodeId: string) => void;
}): React.JSX.Element;

export function ApprovalCard(props: {
  request: ApprovalRequestView;
  onDecide: (decision: {
    approvalId: string;
    approved: boolean;
    feedback?: string;
  }) => Promise<void>;
}): React.JSX.Element;

export function TraceDrawer(props: {
  isOpen: boolean;
  rootSpan: TraceSpanNode | null;
  onClose: () => void;
}): React.JSX.Element;
```

- [ ] **Step 1: Write the failing test in `frontend/tests/app/shell-and-execution.test.tsx`**

```tsx
import { describe, it, expect, vi } from "vitest";
import { render, screen, fireEvent, waitFor } from "@testing-library/react";
import React from "react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { ScopeProvider } from "../../src/lib/scope-context";
import { ModeSwitcher } from "../../src/app/ModeSwitcher";
import { PlanCard } from "../../src/features/conversation/PlanCard";
import { ApprovalCard } from "../../src/features/conversation/ApprovalCard";
import { TraceDrawer } from "../../src/features/observability/TraceDrawer";
import type {
  WorkflowPlanView,
  ApprovalRequestView,
  TraceSpanNode,
} from "../../src/types/chotu";

describe("Task 2: AppShell, ModeSwitcher, PlanCard, ApprovalCard & TraceDrawer", () => {
  it("test_mode_switcher_displays_active_engagement_and_switches_to_personal_rd", async () => {
    const queryClient = new QueryClient();
    render(
      <QueryClientProvider client={queryClient}>
        <ScopeProvider
          queryClient={queryClient}
          initialScope={{ mode: "work", engagementSlug: "eng_acme" }}
        >
          <ModeSwitcher availableEngagements={["eng_acme", "eng_beta"]} />
        </ScopeProvider>
      </QueryClientProvider>
    );

    expect(screen.getByTestId("active-scope-badge")).toHaveTextContent(
      "Work: eng_acme"
    );

    fireEvent.click(screen.getByRole("button", { name: /Switch to Personal R&D/i }));

    await waitFor(() => {
      expect(screen.getByTestId("active-scope-badge")).toHaveTextContent(
        "Personal Mode: R&D Sandbox"
      );
    });
  });

  it("test_plan_card_renders_parallel_subagent_dag_nodes_and_never_raw_lead_tools", () => {
    const plan: WorkflowPlanView = {
      workflowRunId: "wf-901",
      origin: "USER",
      hopDepth: 0,
      classification: {
        intent: "architecture_synthesis",
        mode: "work",
        engagementSlug: "eng_acme",
        departments: ["delivery"],
        baitsPillars: ["Business", "Technology", "Security"],
        scope: "engagement",
        perspective: "internal_team",
        pathType: "SYSTEM_1_ESTABLISHED",
      },
      nodes: [
        {
          nodeId: "node_biz",
          subagentUri: "chotu://agent/architects/business-architect@1.0.0",
          displayName: "Business Architect",
          baitsPillar: "Business",
          dependsOn: [],
          status: "COMPLETED",
          outputArtifactRefs: ["baits-business.md#^f-11"],
        },
        {
          nodeId: "node_tech",
          subagentUri: "chotu://agent/architects/technology-architect@1.0.0",
          displayName: "Technology Architect",
          baitsPillar: "Technology",
          dependsOn: [],
          status: "RUNNING",
          outputArtifactRefs: [],
        },
        {
          nodeId: "node_scribe",
          subagentUri: "chotu://agent/architects/csa-scribe@1.0.0",
          displayName: "CSA Scribe (Fan-In)",
          dependsOn: ["node_biz", "node_tech"],
          status: "PENDING",
          outputArtifactRefs: [],
        },
      ],
    };

    const onSelectNodeTrace = vi.fn();
    render(<PlanCard plan={plan} onSelectNodeTrace={onSelectNodeTrace} />);

    expect(screen.getByText("SYSTEM_1_ESTABLISHED")).toBeInTheDocument();
    expect(screen.getByTestId("dag-node-node_biz")).toHaveTextContent("COMPLETED");
    expect(screen.getByTestId("dag-node-node_tech")).toHaveTextContent("RUNNING");
    expect(screen.getByTestId("dag-node-node_scribe")).toHaveTextContent(
      "Depends on: node_biz, node_tech"
    );

    fireEvent.click(screen.getByTestId("dag-node-node_tech"));
    expect(onSelectNodeTrace).toHaveBeenCalledWith("node_tech");
  });

  it("test_approval_card_dispatches_p0_resume_decision_with_feedback", async () => {
    const req: ApprovalRequestView = {
      approvalId: "appr-77",
      workflowRunId: "wf-901",
      nodeId: "node_tech",
      subagentUri: "chotu://agent/architects/technology-architect@1.0.0",
      requestedAction: "Ingest Universal Reference cloud-cutover-patterns into eng_acme",
      riskLevel: "medium",
      policyRuleId: "POL-UK-INGEST-01",
    };
    const onDecide = vi.fn().mockResolvedValue(undefined);

    render(<ApprovalCard request={req} onDecide={onDecide} />);

    fireEvent.change(screen.getByPlaceholderText(/Optional operator note/i), {
      target: { value: "Approved for Phase 1 scope" },
    });
    fireEvent.click(screen.getByRole("button", { name: /Approve & Resume/i }));

    await waitFor(() => {
      expect(onDecide).toHaveBeenCalledWith({
        approvalId: "appr-77",
        approved: true,
        feedback: "Approved for Phase 1 scope",
      });
    });
  });

  it("test_trace_drawer_renders_four_level_chokepoint_span_hierarchy", () => {
    const rootSpan: TraceSpanNode = {
      spanId: "sp-turn",
      parentSpanId: null,
      kind: "turn",
      name: "turn (origin: USER, hop_depth: 0)",
      durationMs: 410,
      children: [
        {
          spanId: "sp-node",
          parentSpanId: "sp-turn",
          kind: "workflow_node",
          name: "workflow_node: tech_arch_review",
          durationMs: 390,
          children: [
            {
              spanId: "sp-agent",
              parentSpanId: "sp-node",
              kind: "subagent",
              name: "AgentRuntime.invoke",
              capabilityUri: "chotu://agent/architects/technology-architect@1.0.0",
              locatorStage: "STAGE_1_EXACT_URI",
              durationMs: 350,
              tokensUsed: 1840,
              modelTier: "tier-1-reasoning",
              children: [
                {
                  spanId: "sp-tool",
                  parentSpanId: "sp-agent",
                  kind: "tool",
                  name: "ToolRegistry.invoke",
                  capabilityUri: "chotu://tool/knowledge/okf_pgvector_search@1.0.0",
                  workspaceJailPath: "workspaces/acme/okf-wiki/",
                  durationMs: 18,
                  children: [],
                },
              ],
            },
          ],
        },
      ],
    };

    render(<TraceDrawer isOpen={true} rootSpan={rootSpan} onClose={vi.fn()} />);

    expect(
      screen.getByText(/chotu:\/\/agent\/architects\/technology-architect@1\.0\.0/)
    ).toBeInTheDocument();
    expect(screen.getByText("STAGE_1_EXACT_URI")).toBeInTheDocument();
    expect(screen.getByText("workspaces/acme/okf-wiki/")).toBeInTheDocument();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd frontend && npx vitest run tests/app/shell-and-execution.test.tsx`
Expected: FAIL with `Cannot find module '../../src/app/ModeSwitcher'`.

- [ ] **Step 3: Implement `AppShell.tsx`, `ModeSwitcher.tsx`, `CommandPalette.tsx`, `PlanCard.tsx`, `ApprovalCard.tsx`, `PolicyEditor.tsx`, and `TraceDrawer.tsx`**

Build `ModeSwitcher` consuming `useScope()` so selecting an engagement calls `switchScope({ mode: "work", engagementSlug })` and clicking the Personal R&D or Universal Studio buttons calls `switchScope({ mode: "personal", engagementSlug: null })` or `switchScope({ mode: "universal", engagementSlug: null })`, rendering `data-testid="active-scope-badge"`. In `PlanCard`, validate at render time that every node's `subagentUri` begins with `chotu://agent/` (enforcing the Subagents-Only display rule) and render each node with `data-testid={`dag-node-${node.nodeId}`}`, its status badge, upstream `dependsOn` list, and clickable output artifact refs. In `TraceDrawer`, recursively render `TraceSpanNode` children displaying `capabilityUri`, `locatorStage`, `workspaceJailPath`, `durationMs`, and `tokensUsed`.

- [ ] **Step 4: Run test to verify it passes**

Run: `cd frontend && npx vitest run tests/app/shell-and-execution.test.tsx`
Expected: PASS (4 passed).

- [ ] **Step 5: Commit**

```bash
git add frontend/src/types/chotu.ts frontend/src/app/ frontend/src/features/ frontend/tests/app/shell-and-execution.test.tsx
git commit -m "feat(frontend): add AppShell, ModeSwitcher, PlanCard DAG view, ApprovalCard, and TraceDrawer"
```

---

### Task 3: Executive Cockpit & 4-Lane Queue Monitor (`pages/executive/`)

When the operator opens Chotu first thing in the morning as CEO of Chotu Corp, `pages/executive/` is the command bridge. It brings together four executive signals on a single screen:
1. **`ExecutiveBriefingCard`**: Displays the Subconscious Mind's Daily Executive Briefing (overnight ingestion counts across meetings and documents, stale BAITS synthesis alerts, and high-priority Truth Finder conflict links).
2. **`FourLaneQueueMonitor`**: Shows live throughput, queue depth, active leases, and preemption status across all four PostgreSQL priority lanes (`P0_interactive`, `P1_workflow_dag`, `P2_outbox_hooks`, and `P3_subconscious_bulk`), plus the Transactional Outbox zero-silent-skip health ledger with a dead-letter replay button.
3. **`ProbationaryWorkflowCard`**: Surfaces `CANDIDATE` workflows composed by `DynamicWorkflowComposer`. To protect the `System-1 Fast Path` from premature crystallization, our implementation enforces a strict gate: direct one-click promotion to `ESTABLISHED` is enabled only when `consecutiveVerifiedRuns >= 3` and `lastRunFailed === false`; promoting earlier requires an explicit human override confirmation reason.
4. **`ExecutiveRhythmPanel`**: Bridges the operator's daily check-in, reflective journal, and objectives from `dsapps-checkin` and `dsapps-journal` (`chotu://app/personal/checkin@1.0.0` and `chotu://app/personal/journal@1.0.0`).

**Files:**
- Modify: `frontend/src/types/chotu.ts`
- Create: `frontend/src/pages/executive/ExecutiveBriefingCard.tsx`
- Create: `frontend/src/pages/executive/FourLaneQueueMonitor.tsx`
- Create: `frontend/src/pages/executive/ProbationaryWorkflowCard.tsx`
- Create: `frontend/src/pages/executive/ExecutiveRhythmPanel.tsx`
- Create: `frontend/src/pages/executive/ExecutiveCockpitPage.tsx`
- Test: `frontend/tests/pages/executive-cockpit.test.tsx`

**Interfaces:**
- Consumes: `PriorityLaneId`, `EngagementSlug`, `BaitsPillar` from Tasks 1–2.
- Produces:
```typescript
// Additions to frontend/src/types/chotu.ts
export interface DailyExecutiveBriefingView {
  briefingDate: string;
  meetingsIngestedCount: number;
  documentsIngestedCount: number;
  stalePillarsByEngagement: Record<EngagementSlug, BaitsPillar[]>;
  conflictAlerts: Array<{
    conflictId: string;
    engagementSlug: EngagementSlug;
    topic: string;
    leadingOptionSummary: string;
    leadingLikelihoodPct: number;
  }>;
  universalPatternMatches: Array<{
    engagementSlug: EngagementSlug;
    universalConceptId: string;
    summary: string;
  }>;
}

export interface LaneMetricsView {
  laneId: PriorityLaneId;
  pendingCount: number;
  runningLeases: number;
  completedPerMin: number;
  p95PickupMs: number;
  yieldedToHigherPriority: boolean;
}

export interface OutboxHealthView {
  totalTrackedFiles: number;
  pendingProjectionCount: number;
  retryingWithBackoffCount: number;
  deadLetterItems: Array<{
    outboxId: string;
    sourcePath: string;
    sha256: string;
    lastError: string;
    retryCount: number;
  }>;
}

export interface CandidateWorkflowView {
  workflowTemplateId: string;
  intentSignature: string;
  consecutiveVerifiedRuns: number;
  requiredVerifiedRuns: number; // default 3
  lastRunFailed: boolean;
  avgLatencyMs: number;
  avgCostUsd: number;
  nodeSummary: string[];
}

// Component Signatures
export function FourLaneQueueMonitor(props: {
  lanes: LaneMetricsView[];
  outbox: OutboxHealthView;
  onReplayDeadLetter: (outboxId: string) => Promise<void>;
}): React.JSX.Element;

export function ProbationaryWorkflowCard(props: {
  candidate: CandidateWorkflowView;
  onPromote: (payload: {
    workflowTemplateId: string;
    overrideReason?: string;
  }) => Promise<void>;
}): React.JSX.Element;
```

- [ ] **Step 1: Write the failing test in `frontend/tests/pages/executive-cockpit.test.tsx`**

```tsx
import { describe, it, expect, vi } from "vitest";
import { render, screen, fireEvent, waitFor } from "@testing-library/react";
import React from "react";
import { FourLaneQueueMonitor } from "../../src/pages/executive/FourLaneQueueMonitor";
import { ProbationaryWorkflowCard } from "../../src/pages/executive/ProbationaryWorkflowCard";
import type {
  LaneMetricsView,
  OutboxHealthView,
  CandidateWorkflowView,
} from "../../src/types/chotu";

describe("Task 3: Executive Cockpit, 4-Lane Queue Monitor & Probationary DAG Gate", () => {
  it("test_four_lane_queue_monitor_renders_p3_yield_banner_and_replays_outbox_dead_letter", async () => {
    const lanes: LaneMetricsView[] = [
      {
        laneId: "P0_interactive",
        pendingCount: 0,
        runningLeases: 1,
        completedPerMin: 14,
        p95PickupMs: 19,
        yieldedToHigherPriority: false,
      },
      {
        laneId: "P1_workflow_dag",
        pendingCount: 2,
        runningLeases: 4,
        completedPerMin: 28,
        p95PickupMs: 65,
        yieldedToHigherPriority: false,
      },
      {
        laneId: "P2_outbox_hooks",
        pendingCount: 5,
        runningLeases: 2,
        completedPerMin: 90,
        p95PickupMs: 110,
        yieldedToHigherPriority: false,
      },
      {
        laneId: "P3_subconscious_bulk",
        pendingCount: 18,
        runningLeases: 1,
        completedPerMin: 6,
        p95PickupMs: 1400,
        yieldedToHigherPriority: true,
      },
    ];

    const outbox: OutboxHealthView = {
      totalTrackedFiles: 142,
      pendingProjectionCount: 3,
      retryingWithBackoffCount: 1,
      deadLetterItems: [
        {
          outboxId: "obx-dead-9",
          sourcePath: "workspaces/acme/okf-wiki/payment-gateway.md",
          sha256: "a94f88bc11",
          lastError: "Docling sidecar timeout after 5 retries",
          retryCount: 5,
        },
      ],
    };

    const onReplay = vi.fn().mockResolvedValue(undefined);
    render(
      <FourLaneQueueMonitor
        lanes={lanes}
        outbox={outbox}
        onReplayDeadLetter={onReplay}
      />
    );

    expect(screen.getByTestId("lane-P0_interactive")).toHaveTextContent("19ms");
    expect(screen.getByTestId("lane-P3_subconscious_bulk")).toHaveTextContent(
      "Yielding capacity to P0/P1"
    );
    expect(
      screen.getByText(/workspaces\/acme\/okf-wiki\/payment-gateway\.md/)
    ).toBeInTheDocument();

    fireEvent.click(screen.getByRole("button", { name: /Replay Now/i }));
    await waitFor(() => {
      expect(onReplay).toHaveBeenCalledWith("obx-dead-9");
    });
  });

  it("test_probationary_workflow_blocks_direct_promotion_below_three_verified_runs", async () => {
    const immatureCandidate: CandidateWorkflowView = {
      workflowTemplateId: "wf-tmpl-raidsync",
      intentSignature: "work:raid_extraction_from_transcript",
      consecutiveVerifiedRuns: 2,
      requiredVerifiedRuns: 3,
      lastRunFailed: false,
      avgLatencyMs: 3200,
      avgCostUsd: 0.04,
      nodeSummary: ["parse-transcript", "csa-extract", "raid-bundle-updater"],
    };

    const onPromote = vi.fn().mockResolvedValue(undefined);
    render(
      <ProbationaryWorkflowCard
        candidate={immatureCandidate}
        onPromote={onPromote}
      />
    );

    const directBtn = screen.getByRole("button", {
      name: /Promote to ESTABLISHED/i,
    });
    expect(directBtn).toBeDisabled();
    expect(screen.getByText(/2 \/ 3 verified runs/i)).toBeInTheDocument();

    // Operator can only force-promote by entering an explicit override reason
    fireEvent.change(
      screen.getByPlaceholderText(/Override reason required below 3 runs/i),
      { target: { value: "Executive sign-off after manual audit" } }
    );
    expect(directBtn).not.toBeDisabled();

    fireEvent.click(directBtn);
    await waitFor(() => {
      expect(onPromote).toHaveBeenCalledWith({
        workflowTemplateId: "wf-tmpl-raidsync",
        overrideReason: "Executive sign-off after manual audit",
      });
    });
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd frontend && npx vitest run tests/pages/executive-cockpit.test.tsx`
Expected: FAIL with `Cannot find module '../../src/pages/executive/FourLaneQueueMonitor'`.

- [ ] **Step 3: Implement `FourLaneQueueMonitor.tsx`, `ProbationaryWorkflowCard.tsx`, `ExecutiveBriefingCard.tsx`, `ExecutiveRhythmPanel.tsx`, and `ExecutiveCockpitPage.tsx`**

In `FourLaneQueueMonitor`, render each lane inside `data-testid={`lane-${lane.laneId}`}` displaying `p95PickupMs`, `pendingCount`, `runningLeases`, and—whenever `lane.yieldedToHigherPriority === true`—a badge reading `"Yielding capacity to P0/P1"`. Render every item in `outbox.deadLetterItems` with its `sourcePath`, `sha256`, `lastError`, and a `"Replay Now"` button wired to `onReplayDeadLetter(item.outboxId)`. In `ProbationaryWorkflowCard`, compute `isReadyForStandardPromotion = candidate.consecutiveVerifiedRuns >= candidate.requiredVerifiedRuns && !candidate.lastRunFailed`; disable `"Promote to ESTABLISHED"` when `!isReadyForStandardPromotion && overrideReason.trim().length === 0`.

- [ ] **Step 4: Run test to verify it passes**

Run: `cd frontend && npx vitest run tests/pages/executive-cockpit.test.tsx`
Expected: PASS (2 passed).

- [ ] **Step 5: Commit**

```bash
git add frontend/src/types/chotu.ts frontend/src/pages/executive/ frontend/tests/pages/executive-cockpit.test.tsx
git commit -m "feat(frontend): add Executive Cockpit, 4-Lane Queue Monitor, and Probationary DAG promotion gate"
```

---

### Task 4: Engagement Project Workspace, Master Data Explorer & Multi-Perspective Fact Filtering (`pages/projects/`)

When the operator enters a specific client engagement (`eng_<slug>`), `pages/projects/` provides the consulting control room. Unlike a generic RAG chat interface where facts float around as unindexed text chunks, a real 6-to-12-month engagement revolves around structured **Engagement Master Data** (Accounts, Stakeholders, Systems, Capabilities, Decisions, Deliverables, Metrics, and RAID items) where every record is anchored to canonical `okf-wiki/` concepts and atomic facts (`[[concept#^f-id]]`), and where every claim is tagged along both the **6-value `scope:` axis** (`engagement` · `client` · `internal` · `reference` · `prior-project` · `deliverable`) and the **5-value `perspective:` axis** (`client_stated` · `internal_team` · `personal_take` · `work_research` · `personal_rd`).

In this task, we build `EngagementMasterDataExplorer` (supporting instant tab switching across the Master Data entities and clicking any `[[concept#^f-id]]` anchor to highlight its supporting fact), `MultiPerspectiveFactExplorer` (filtering atomic facts by `scope:`, `perspective:`, and bitemporal `observed_at` / `ingested_at` timestamps while visually flagging internal-only claims that must never be cited in client deliverables), and `EngagementWorkspacePage`.

**Files:**
- Modify: `frontend/src/types/chotu.ts`
- Create: `frontend/src/pages/projects/EngagementMasterDataExplorer.tsx`
- Create: `frontend/src/pages/projects/MultiPerspectiveFactExplorer.tsx`
- Create: `frontend/src/pages/projects/EngagementWorkspacePage.tsx`
- Test: `frontend/tests/pages/engagement-workspace.test.tsx`

**Interfaces:**
- Consumes: `EngagementSlug`, `FactScope`, `FactPerspective`, `BaitsPillar` from Task 1.
- Produces:
```typescript
// Additions to frontend/src/types/chotu.ts
export type MasterDataCategory =
  | "stakeholders"
  | "systems"
  | "capabilities"
  | "decisions"
  | "deliverables"
  | "metrics"
  | "raid";

export interface MasterDataRow {
  id: string;
  category: MasterDataCategory;
  title: string;
  status: string;
  version: string;
  baitsPillar: BaitsPillar;
  isStale: boolean;
  citationAnchors: Array<`[[${string}#^${string}]]`>;
  attributes: Record<string, string>;
}

export interface EngagementFactView {
  factId: string; // e.g., "^f-204"
  conceptSlug: string;
  statement: string;
  version: number;
  scope: FactScope;
  perspective: FactPerspective;
  stakeholderRole?: string;
  baitsPillar: BaitsPillar;
  observedAt: string;
  ingestedAt: string;
  clientCiteable: boolean; // true only when scope in ['engagement', 'client', 'reference', 'deliverable']
}

// Component Signatures
export function EngagementMasterDataExplorer(props: {
  engagementSlug: EngagementSlug;
  rows: MasterDataRow[];
  onSelectCitationAnchor: (anchor: `[[${string}#^${string}]]`) => void;
}): React.JSX.Element;

export function MultiPerspectiveFactExplorer(props: {
  facts: EngagementFactView[];
  highlightedAnchor?: `[[${string}#^${string}]]` | null;
}): React.JSX.Element;
```

- [ ] **Step 1: Write the failing test in `frontend/tests/pages/engagement-workspace.test.tsx`**

```tsx
import { describe, it, expect, vi } from "vitest";
import { render, screen, fireEvent } from "@testing-library/react";
import React from "react";
import { EngagementMasterDataExplorer } from "../../src/pages/projects/EngagementMasterDataExplorer";
import { MultiPerspectiveFactExplorer } from "../../src/pages/projects/MultiPerspectiveFactExplorer";
import type { MasterDataRow, EngagementFactView } from "../../src/types/chotu";

describe("Task 4: Engagement Master Data Explorer & Multi-Perspective Fact Explorer", () => {
  it("test_master_data_explorer_filters_by_category_and_emits_citation_anchor_click", () => {
    const rows: MasterDataRow[] = [
      {
        id: "md-dec-04",
        category: "decisions",
        title: "ADR-004: Phased Mainframe DB Cutover",
        status: "SIGNED",
        version: "v1.2",
        baitsPillar: "Technology",
        isStale: false,
        citationAnchors: ["[[legacy-db-cutover#^f-204]]"],
        attributes: { owner: "Principal Architect", milestone: "M3" },
      },
      {
        id: "md-raid-11",
        category: "raid",
        title: "RISK-11: PCI Audit Window Overlap",
        status: "OPEN",
        version: "v1.0",
        baitsPillar: "Security",
        isStale: true,
        citationAnchors: ["[[pci-compliance#^f-309]]"],
        attributes: { severity: "HIGH" },
      },
    ];

    const onSelectAnchor = vi.fn();
    render(
      <EngagementMasterDataExplorer
        engagementSlug="eng_acme"
        rows={rows}
        onSelectCitationAnchor={onSelectAnchor}
      />
    );

    fireEvent.click(screen.getByRole("tab", { name: /decisions/i }));
    expect(
      screen.getByText("ADR-004: Phased Mainframe DB Cutover")
    ).toBeInTheDocument();

    fireEvent.click(screen.getByRole("button", { name: "[[legacy-db-cutover#^f-204]]" }));
    expect(onSelectAnchor).toHaveBeenCalledWith("[[legacy-db-cutover#^f-204]]");
  });

  it("test_multi_perspective_fact_explorer_filters_by_6_scopes_and_5_perspectives_and_flags_non_citeable", () => {
    const facts: EngagementFactView[] = [
      {
        factId: "^f-204",
        conceptSlug: "legacy-db-cutover",
        statement: "Phased cutover completes before Q3 PCI freeze.",
        version: 2,
        scope: "engagement",
        perspective: "client_stated",
        stakeholderRole: "Principal Engineer",
        baitsPillar: "Technology",
        observedAt: "2026-10-10T14:00:00Z",
        ingestedAt: "2026-10-10T15:00:00Z",
        clientCiteable: true,
      },
      {
        factId: "^f-209",
        conceptSlug: "legacy-db-cutover",
        statement: "Internal hypothesis: client DBA team is understaffed for weekend cutover.",
        version: 1,
        scope: "internal",
        perspective: "personal_take",
        baitsPillar: "Technology",
        observedAt: "2026-10-12T18:00:00Z",
        ingestedAt: "2026-10-12T18:05:00Z",
        clientCiteable: false,
      },
    ];

    render(
      <MultiPerspectiveFactExplorer
        facts={facts}
        highlightedAnchor="[[legacy-db-cutover#^f-204]]"
      />
    );

    // Both visible initially; internal/personal_take fact shows Firewall warning badge
    expect(screen.getByTestId("fact-card-^f-204")).toHaveAttribute(
      "data-highlighted",
      "true"
    );
    expect(screen.getByTestId("fact-card-^f-209")).toHaveTextContent(
      "INTERNAL ONLY — Blocked from Client Deliverables"
    );

    // Filter by perspective: personal_take
    fireEvent.click(screen.getByRole("button", { name: /perspective: personal_take/i }));
    expect(screen.queryByTestId("fact-card-^f-204")).not.toBeInTheDocument();
    expect(screen.getByTestId("fact-card-^f-209")).toBeInTheDocument();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd frontend && npx vitest run tests/pages/engagement-workspace.test.tsx`
Expected: FAIL with `Cannot find module '../../src/pages/projects/EngagementMasterDataExplorer'`.

- [ ] **Step 3: Implement `EngagementMasterDataExplorer.tsx`, `MultiPerspectiveFactExplorer.tsx`, and `EngagementWorkspacePage.tsx`**

In `EngagementMasterDataExplorer`, render tabs for all 7 `MasterDataCategory` values (`stakeholders`, `systems`, `capabilities`, `decisions`, `deliverables`, `metrics`, `raid`), show staleness badges (`isStale`), and render each `citationAnchor` as a button that invokes `onSelectCitationAnchor(anchor)`. In `MultiPerspectiveFactExplorer`, render toggleable facet pills for all 6 `FactScope` values and all 5 `FactPerspective` values, mark any card matching `highlightedAnchor === `[[${fact.conceptSlug}#${fact.factId}]]`` with `data-highlighted="true"`, and display `"INTERNAL ONLY — Blocked from Client Deliverables"` whenever `!fact.clientCiteable` (i.e., `scope === "internal"` or `scope === "prior-project"`).

- [ ] **Step 4: Run test to verify it passes**

Run: `cd frontend && npx vitest run tests/pages/engagement-workspace.test.tsx`
Expected: PASS (2 passed).

- [ ] **Step 5: Commit**

```bash
git add frontend/src/types/chotu.ts frontend/src/pages/projects/EngagementMasterDataExplorer.tsx frontend/src/pages/projects/MultiPerspectiveFactExplorer.tsx frontend/src/pages/projects/EngagementWorkspacePage.tsx frontend/tests/pages/engagement-workspace.test.tsx
git commit -m "feat(frontend): add Engagement Master Data Explorer and Multi-Perspective Fact Explorer"
```

---

### Task 5: Truth Finder + HITS Conflict Resolution Studio (`pages/projects/TruthFinderConflictCard.tsx` & `MakeBreakImpactGraph.tsx`)

When two claims inside `eng_<slug>` contradict each other across meetings or documents, Chotu never auto-overwrites the older claim and never stalls project delivery. Instead, the backend Truth Finder + HITS engine computes each option's HITS Authority and Hub scores, simulates **Step 1: Structural Relationship Make/Break Impact** (which downstream BAITS facts, ADRs, deliverables, or RAID items each option validates vs. breaks), and synthesizes **Step 2: Plausibility & Sense-Making Rationale** (using Engagement Master Data and any reference standards pulled from Universal Knowledge).

Task 5 builds `TruthFinderConflictCard` and `MakeBreakImpactGraph` to surface that analysis visually and give the operator four surgical HITL resolution actions (`promote_a`, `promote_b`, `keep_both_tracked`, `draft_clarifying_question`). To prevent a critical real-world mistake—promoting an option that breaks corroborated downstream ADRs or deliverables without realizing what will break—our implementation enforces a safety check: if the chosen option has `breaksRelationships.length > 0`, clicking promote requires acknowledging the broken downstream artifacts and explicitly passes `flagBrokenArtifactIds` in the resolution payload so the backend marks `.csa/synthesis-status.md` and those Master Data rows stale.

**Files:**
- Modify: `frontend/src/types/chotu.ts`
- Create: `frontend/src/pages/projects/MakeBreakImpactGraph.tsx`
- Create: `frontend/src/pages/projects/TruthFinderConflictCard.tsx`
- Modify: `frontend/src/pages/projects/EngagementWorkspacePage.tsx`
- Test: `frontend/tests/pages/truth-finder-studio.test.tsx`

**Interfaces:**
- Consumes: `FactScope`, `FactPerspective`, `BaitsPillar` from Task 1.
- Produces:
```typescript
// Additions to frontend/src/types/chotu.ts
export interface ImpactEdgeView {
  targetId: string; // e.g., "ADR-004", "M3-Deliverable", "[[pci-compliance#^f-309]]"
  targetKind: "master_data" | "baits_synthesis" | "fact";
  targetTitle: string;
  baitsPillar: BaitsPillar;
  effect: "MAKES_OR_VALIDATES" | "BREAKS_OR_ORPHANS";
}

export interface ConflictClaimOptionView {
  optionKey: "A" | "B";
  factId: string;
  version: string;
  statement: string;
  scope: FactScope;
  perspective: FactPerspective;
  sourceLabel: string;
  hitsAuthorityScore: number;
  hitsHubScore: number;
  calibratedLikelihoodPct: number;
  makesRelationships: ImpactEdgeView[];
  breaksRelationships: ImpactEdgeView[];
}

export interface TruthFinderConflictView {
  conflictId: string;
  engagementSlug: EngagementSlug;
  topic: string;
  optionA: ConflictClaimOptionView;
  optionB: ConflictClaimOptionView;
  step2PlausibilityRationale: string;
  ingestedUniversalReferenceRefs: string[];
}

export type ConflictResolutionAction =
  | {
      conflictId: string;
      resolution: "promote_a" | "promote_b";
      flagBrokenArtifactIds: string[];
      acknowledgeBrokenImpact: boolean;
    }
  | {
      conflictId: string;
      resolution: "keep_both_tracked";
      trackingConditionNote: string;
    }
  | {
      conflictId: string;
      resolution: "draft_clarifying_question";
      targetStakeholder: string;
    };

// Component Signatures
export function MakeBreakImpactGraph(props: {
  optionA: ConflictClaimOptionView;
  optionB: ConflictClaimOptionView;
}): React.JSX.Element;

export function TruthFinderConflictCard(props: {
  conflict: TruthFinderConflictView;
  onResolve: (action: ConflictResolutionAction) => Promise<void>;
}): React.JSX.Element;
```

- [ ] **Step 1: Write the failing test in `frontend/tests/pages/truth-finder-studio.test.tsx`**

```tsx
import { describe, it, expect, vi } from "vitest";
import { render, screen, fireEvent, waitFor } from "@testing-library/react";
import React from "react";
import { TruthFinderConflictCard } from "../../src/pages/projects/TruthFinderConflictCard";
import type { TruthFinderConflictView } from "../../src/types/chotu";

const sampleConflict: TruthFinderConflictView = {
  conflictId: "tf-conf-301",
  engagementSlug: "eng_acme",
  topic: "Data Residency & Legacy DB Cutover Scope",
  optionA: {
    optionKey: "A",
    factId: "^f-204",
    version: "v1.2",
    statement: "In-Scope Phased Cutover before Q3 PCI freeze",
    scope: "engagement",
    perspective: "client_stated",
    sourceLabel: "Signed SOW + Principal Eng Tech Workshop",
    hitsAuthorityScore: 0.89,
    hitsHubScore: 0.84,
    calibratedLikelihoodPct: 78,
    makesRelationships: [
      {
        targetId: "ADR-004",
        targetKind: "master_data",
        targetTitle: "ADR-004: Phased Mainframe DB Cutover",
        baitsPillar: "Technology",
        effect: "MAKES_OR_VALIDATES",
      },
    ],
    breaksRelationships: [],
  },
  optionB: {
    optionKey: "B",
    factId: "^f-255",
    version: "v1.3",
    statement: "Deferred to Phase 2 per weekly status call remark",
    scope: "client",
    perspective: "client_stated",
    sourceLabel: "Weekly Status Call (VP Operations)",
    hitsAuthorityScore: 0.52,
    hitsHubScore: 0.61,
    calibratedLikelihoodPct: 22,
    makesRelationships: [],
    breaksRelationships: [
      {
        targetId: "ADR-004",
        targetKind: "master_data",
        targetTitle: "ADR-004: Phased Mainframe DB Cutover",
        baitsPillar: "Technology",
        effect: "BREAKS_OR_ORPHANS",
      },
      {
        targetId: "DELIV-M3",
        targetKind: "master_data",
        targetTitle: "Milestone M3 Cutover Runbook",
        baitsPillar: "Security",
        effect: "BREAKS_OR_ORPHANS",
      },
    ],
  },
  step2PlausibilityRationale:
    "Option A aligns with signed SOW critical path and reference:cloud-cutover-patterns; Option B breaks PCI prerequisite.",
  ingestedUniversalReferenceRefs: ["reference:cloud-cutover-patterns"],
};

describe("Task 5: Truth Finder + HITS Conflict Resolution Studio", () => {
  it("test_renders_hits_scores_step1_make_break_edges_and_step2_rationale", () => {
    render(
      <TruthFinderConflictCard conflict={sampleConflict} onResolve={vi.fn()} />
    );

    expect(screen.getByText(/78% Likely/i)).toBeInTheDocument();
    expect(screen.getByText(/22% Likely/i)).toBeInTheDocument();
    expect(screen.getByText(/HITS Authority: 0.89/i)).toBeInTheDocument();
    expect(screen.getByTestId("impact-edge-B-ADR-004")).toHaveTextContent(
      "BREAKS_OR_ORPHANS"
    );
    expect(
      screen.getByText(/reference:cloud-cutover-patterns/)
    ).toBeInTheDocument();
  });

  it("test_promoting_option_with_broken_relationships_requires_staleness_acknowledgment_and_flags_artifacts", async () => {
    const onResolve = vi.fn().mockResolvedValue(undefined);
    render(
      <TruthFinderConflictCard conflict={sampleConflict} onResolve={onResolve} />
    );

    const promoteBBtn = screen.getByRole("button", {
      name: /Promote Option B & Flag Broken Artifacts/i,
    });
    // Blocked until operator checks acknowledgment box for the 2 broken downstream items
    expect(promoteBBtn).toBeDisabled();

    fireEvent.click(
      screen.getByLabelText(
        /Acknowledge flagging 2 broken downstream items \(ADR-004, DELIV-M3\) as stale/i
      )
    );
    expect(promoteBBtn).not.toBeDisabled();

    fireEvent.click(promoteBBtn);
    await waitFor(() => {
      expect(onResolve).toHaveBeenCalledWith({
        conflictId: "tf-conf-301",
        resolution: "promote_b",
        flagBrokenArtifactIds: ["ADR-004", "DELIV-M3"],
        acknowledgeBrokenImpact: true,
      });
    });
  });

  it("test_keep_both_tracked_with_conditions_preserves_parallel_claims", async () => {
    const onResolve = vi.fn().mockResolvedValue(undefined);
    render(
      <TruthFinderConflictCard conflict={sampleConflict} onResolve={onResolve} />
    );

    fireEvent.change(
      screen.getByPlaceholderText(/Condition for keeping both claims tracked/i),
      {
        target: {
          value: "Await Steering Committee sign-off on Friday",
        },
      }
    );
    fireEvent.click(
      screen.getByRole("button", { name: /Keep Both Tracked with Conditions/i })
    );

    await waitFor(() => {
      expect(onResolve).toHaveBeenCalledWith({
        conflictId: "tf-conf-301",
        resolution: "keep_both_tracked",
        trackingConditionNote: "Await Steering Committee sign-off on Friday",
      });
    });
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd frontend && npx vitest run tests/pages/truth-finder-studio.test.tsx`
Expected: FAIL with `Cannot find module '../../src/pages/projects/TruthFinderConflictCard'`.

- [ ] **Step 3: Implement `MakeBreakImpactGraph.tsx` and `TruthFinderConflictCard.tsx`**

In `MakeBreakImpactGraph`, render each option's `makesRelationships` and `breaksRelationships` with `data-testid={`impact-edge-${option.optionKey}-${edge.targetId}`}` showing the `targetId`, `targetTitle`, `baitsPillar`, and `effect`. In `TruthFinderConflictCard`, display HITS Authority and Hub scores, calibrated likelihood percentages, the `MakeBreakImpactGraph`, `step2PlausibilityRationale`, and `ingestedUniversalReferenceRefs`. If an option has `breaksRelationships.length > 0`, render a checkbox labeled ``Acknowledge flagging ${breaks.length} broken downstream items (${breaks.map(b => b.targetId).join(", ")}) as stale`` and keep that option's promote button disabled until checked; on click, pass `flagBrokenArtifactIds: breaks.map(b => b.targetId)` and `acknowledgeBrokenImpact: true` to `onResolve`.

- [ ] **Step 4: Run test to verify it passes**

Run: `cd frontend && npx vitest run tests/pages/truth-finder-studio.test.tsx`
Expected: PASS (3 passed).

- [ ] **Step 5: Commit**

```bash
git add frontend/src/types/chotu.ts frontend/src/pages/projects/MakeBreakImpactGraph.tsx frontend/src/pages/projects/TruthFinderConflictCard.tsx frontend/src/pages/projects/EngagementWorkspacePage.tsx frontend/tests/pages/truth-finder-studio.test.tsx
git commit -m "feat(frontend): add Truth Finder + HITS Conflict Resolution Studio and Make/Break Impact Graph"
```

---

### Task 6: Universal Knowledge Studio, `engagement://` Provenance Drill-Down, 5-Lens Retrospective Elevation Workbench & Evals (`pages/studio/` & `pages/evals/`)

While `pages/projects/` enforces closed-world engagement isolation, `pages/studio/` and `pages/evals/` let the operator curate the disjoint **Universal Knowledge Fabric (`universal.*`)** and inspect platform retrieval quality. We harvest the `StudioShelfPage`, `ConceptTree`, `CitationGraphView`, `StudioSkillsPage`, and `EvalsPage` patterns from [`okf-kb-manager`](file:///home/nox/Developer/ds2/okf-kb-manager) and add three capabilities specific to Chotu:
1. **`Discover -> Select -> Ingest` into Engagement Scope**: Searching `universal.*` from the studio and ingesting selected universal facts/concepts into an active engagement (`eng_<slug>`) tagged as `scope: reference` or `scope: prior-project`.
2. **Two-Stage `engagement://<engagement_id>#^<anchor>` Provenance Drill-Down (`CitationDrillDownDrawer`)**: Stage 1 shows the sanitized universal rule stored in `universal.*`; Stage 2 requires an explicit click to peek read-only into the historical `eng_<engagement_id>` record at `^<anchor>`.
3. **The 5-Lens Retrospective Elevation Workbench (`RetrospectiveElevationWorkbench`)**: Displays candidate closure learnings across the 5 lenses (`achievements_path`, `methodology_efficacy`, `ppttt`, `counterfactuals_day1_rules`, `harvested_research`). To prevent accidental client PII leakage into `universal.*`, any candidate card with `sanitationStatus === "flagged"` or a missing `engagement://<id>#^<anchor>` citation blocks submission until the operator sanitizes the text.

**Files:**
- Modify: `frontend/src/types/chotu.ts`
- Create: `frontend/src/pages/studio/CitationDrillDownDrawer.tsx`
- Create: `frontend/src/pages/studio/RetrospectiveElevationWorkbench.tsx`
- Create: `frontend/src/pages/studio/UniversalStudioPage.tsx`
- Create: `frontend/src/pages/evals/EvalsPage.tsx`
- Test: `frontend/tests/pages/universal-studio-and-evals.test.tsx`

**Interfaces:**
- Consumes: `EngagementSlug`, `FactScope` from Task 1.
- Produces:
```typescript
// Additions to frontend/src/types/chotu.ts
export type RetrospectiveLens =
  | "achievements_path"
  | "methodology_efficacy"
  | "ppttt"
  | "counterfactuals_day1_rules"
  | "harvested_research";

export type EngagementCitationUri = `engagement://${EngagementSlug}#${string}`;

export interface RetrospectiveCandidateCardView {
  candidateId: string;
  sourceEngagementSlug: EngagementSlug;
  lens: RetrospectiveLens;
  proposedUniversalTitle: string;
  sanitizedDraftText: string;
  flaggedTokens: string[];
  sanitationStatus: "clean" | "flagged";
  citationUri: EngagementCitationUri | "";
}

export interface UniversalCitationDrillDownView {
  universalConceptId: string;
  universalSummary: string;
  elevatedAt: string;
  lens: RetrospectiveLens;
  citationUri: EngagementCitationUri;
  historicalSnapshot?: {
    engagementSlug: EngagementSlug;
    anchorId: string;
    originalStatement: string;
    observedAt: string;
    authoritativeDecisionRef?: string;
  };
}

// Component Signatures
export function RetrospectiveElevationWorkbench(props: {
  candidates: RetrospectiveCandidateCardView[];
  onApproveElevation: (payload: {
    candidateId: string;
    finalSanitizedText: string;
    citationUri: EngagementCitationUri;
  }) => Promise<void>;
}): React.JSX.Element;

export function CitationDrillDownDrawer(props: {
  item: UniversalCitationDrillDownView;
  onLoadHistoricalStage2: (citationUri: EngagementCitationUri) => Promise<void>;
}): React.JSX.Element;
```

- [ ] **Step 1: Write the failing test in `frontend/tests/pages/universal-studio-and-evals.test.tsx`**

```tsx
import { describe, it, expect, vi } from "vitest";
import { render, screen, fireEvent, waitFor } from "@testing-library/react";
import React from "react";
import { RetrospectiveElevationWorkbench } from "../../src/pages/studio/RetrospectiveElevationWorkbench";
import { CitationDrillDownDrawer } from "../../src/pages/studio/CitationDrillDownDrawer";
import type {
  RetrospectiveCandidateCardView,
  UniversalCitationDrillDownView,
} from "../../src/types/chotu";

describe("Task 6: Universal Knowledge Studio, Two-Stage Drill-Down & 5-Lens Elevation Workbench", () => {
  it("test_retrospective_elevation_blocks_submission_when_pii_flagged_or_citation_missing", async () => {
    const candidates: RetrospectiveCandidateCardView[] = [
      {
        candidateId: "retro-cand-1",
        sourceEngagementSlug: "eng_acme",
        lens: "counterfactuals_day1_rules",
        proposedUniversalTitle: "Day-1 Dual-Write Shadow Verification Pattern",
        sanitizedDraftText:
          "During Acme Corp cutover with VP John Miller, dual-write caught 14 schema drifts.",
        flaggedTokens: ["Acme Corp", "John Miller"],
        sanitationStatus: "flagged",
        citationUri: "engagement://eng_acme#^f-204",
      },
    ];

    const onApprove = vi.fn().mockResolvedValue(undefined);
    render(
      <RetrospectiveElevationWorkbench
        candidates={candidates}
        onApproveElevation={onApprove}
      />
    );

    const elevateBtn = screen.getByRole("button", {
      name: /Approve & Elevate to Universal/i,
    });
    expect(elevateBtn).toBeDisabled();
    expect(
      screen.getByText(/Flagged client tokens: Acme Corp, John Miller/i)
    ).toBeInTheDocument();

    // Operator edits out the flagged client tokens
    fireEvent.change(screen.getByLabelText(/Sanitized Universal Insight/i), {
      target: {
        value:
          "During enterprise mainframe cutovers, enabling dual-write shadow verification on Day 1 catches schema drift before PCI freeze.",
      },
    });

    expect(elevateBtn).not.toBeDisabled();
    fireEvent.click(elevateBtn);

    await waitFor(() => {
      expect(onApprove).toHaveBeenCalledWith({
        candidateId: "retro-cand-1",
        finalSanitizedText:
          "During enterprise mainframe cutovers, enabling dual-write shadow verification on Day 1 catches schema drift before PCI freeze.",
        citationUri: "engagement://eng_acme#^f-204",
      });
    });
  });

  it("test_two_stage_engagement_citation_drill_down_loads_historical_dataset_on_explicit_click", async () => {
    const item: UniversalCitationDrillDownView = {
      universalConceptId: "uk-dual-write-pattern",
      universalSummary:
        "Enable dual-write shadow verification 3 weeks prior to cutover.",
      elevatedAt: "2026-10-28T10:00:00Z",
      lens: "counterfactuals_day1_rules",
      citationUri: "engagement://eng_acme#^f-204",
    };

    const onLoadStage2 = vi.fn().mockResolvedValue(undefined);
    render(
      <CitationDrillDownDrawer
        item={item}
        onLoadHistoricalStage2={onLoadStage2}
      />
    );

    expect(screen.getByText(/Stage 1: Sanitized Universal Rule/i)).toBeInTheDocument();
    expect(
      screen.queryByText(/Stage 2: Historical Engagement Dataset Snapshot/i)
    ).not.toBeInTheDocument();

    fireEvent.click(
      screen.getByRole("button", {
        name: /Peek Historical Dataset \(engagement:\/\/eng_acme#\^f-204\)/i,
      })
    );

    await waitFor(() => {
      expect(onLoadStage2).toHaveBeenCalledWith("engagement://eng_acme#^f-204");
    });
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd frontend && npx vitest run tests/pages/universal-studio-and-evals.test.tsx`
Expected: FAIL with `Cannot find module '../../src/pages/studio/RetrospectiveElevationWorkbench'`.

- [ ] **Step 3: Implement `RetrospectiveElevationWorkbench.tsx`, `CitationDrillDownDrawer.tsx`, `UniversalStudioPage.tsx`, and `EvalsPage.tsx`**

In `RetrospectiveElevationWorkbench`, maintain controlled state for `draftText` per candidate and compute `remainingFlagged = candidate.flaggedTokens.filter(tok => draftText.toLowerCase().includes(tok.toLowerCase()))` plus `hasValidCitation = /^engagement:\/\/eng_[a-z0-9_-]+#\^.+$/.test(candidate.citationUri)`. Keep `"Approve & Elevate to Universal"` disabled whenever `remainingFlagged.length > 0 || !hasValidCitation`. In `CitationDrillDownDrawer`, render Stage 1 (`universalSummary`, `lens`, `citationUri`) immediately and provide the `"Peek Historical Dataset (${item.citationUri})"` button to trigger `onLoadHistoricalStage2(item.citationUri)` and reveal Stage 2 when `item.historicalSnapshot` is populated.

- [ ] **Step 4: Run test to verify it passes**

Run: `cd frontend && npx vitest run tests/pages/universal-studio-and-evals.test.tsx`
Expected: PASS (2 passed).

- [ ] **Step 5: Commit**

```bash
git add frontend/src/types/chotu.ts frontend/src/pages/studio/ frontend/src/pages/evals/ frontend/tests/pages/universal-studio-and-evals.test.tsx
git commit -m "feat(frontend): add Universal Knowledge Studio, two-stage citation drill-down, and 5-Lens Elevation Workbench"
```

---

### Task 7: End-to-End Verification Suite (Full-Stack Playwright UAT Scenarios, Backend E2E Integration & Root `Makefile` Sign-Off Gates)

With all UI workbenches tested in isolation, Task 7 wires the main application entrypoint (`frontend/src/App.tsx` and `frontend/src/main.tsx`), writes the four full-stack Playwright UAT scenarios in `frontend/e2e/chotu-portal-uat.spec.ts` plus the backend E2E verification test in `backend/tests/e2e/test_portal_full_stack_scenarios.py`, and adds the five root `Makefile` sign-off targets (`make lint`, `make typecheck`, `make check-architecture`, `make test-unit`, `make test-contracts`, `make test-integration`, `make test-e2e`).

These four Playwright UAT scenarios exercise the exact daily operator loop end-to-end:
1. **Scenario 1**: Switching `ModeSwitcher` to `Work: eng_acme`, ingesting multi-perspective transcripts (`client_stated` vs. `internal_team`), watching the 4-Lane Queue Monitor, and inspecting the resulting Engagement Master Data rows and multi-perspective facts.
2. **Scenario 2**: Opening the Truth Finder + HITS Conflict Resolution Card for the competing cutover claims, inspecting Step 1 Make/Break edges and Step 2 Plausibility rationale, and promoting Option A without losing Option B's bitemporal lineage.
3. **Scenario 3**: Dispatching a parallel BAITS synthesis DAG via `CommandPalette` (`Cmd+K`), watching per-node progress in `PlanCard`, approving a gated step via `ApprovalCard`, and inspecting the 4-level span hierarchy in `TraceDrawer`.
4. **Scenario 4**: Reviewing the 5-Lens Retrospective Elevation Workbench, sanitizing client PII, elevating the pattern to `universal.*` with `engagement://eng_acme#^f-204`, switching to `Work: eng_beta`, verifying closed-world isolation, and running `Discover -> Select -> Ingest` (`scope: prior-project`).

**Files:**
- Create: `frontend/index.html`
- Create: `frontend/src/App.tsx`
- Create: `frontend/src/main.tsx`
- Create: `frontend/playwright.config.ts`
- Create: `frontend/e2e/chotu-portal-uat.spec.ts`
- Create: `backend/tests/e2e/test_portal_full_stack_scenarios.py`
- Modify: `Makefile`

**Interfaces:**
- Consumes: All components and hooks from Tasks 1–6 and the FastAPI `/api/v1/*` endpoints.
- Produces: Full-stack Playwright UAT suite (`frontend/e2e/chotu-portal-uat.spec.ts`) and root `Makefile` verification targets.

- [ ] **Step 1: Write the failing Playwright E2E test in `frontend/e2e/chotu-portal-uat.spec.ts`**

```typescript
import { test, expect } from "@playwright/test";

test.describe("Slice Group 4 Full-Stack Operator UAT Scenarios", () => {
  test("UAT 1-4: Multi-perspective ingestion, Truth Finder conflict weighing, parallel BAITS DAG & Retrospective Elevation", async ({
    page,
  }) => {
    await page.goto("/");

    // UAT Scenario 1: Scope switch & 4-Lane Queue Monitor + Master Data binding
    await expect(page.getByTestId("active-scope-badge")).toContainText(
      "Work: eng_acme"
    );
    await expect(page.getByTestId("lane-P0_interactive")).toBeVisible();
    await expect(page.getByTestId("lane-P3_subconscious_bulk")).toContainText(
      "Yielding capacity to P0/P1"
    );

    await page.getByRole("link", { name: /Engagement Workspace/i }).click();
    await page.getByRole("tab", { name: /decisions/i }).click();
    await expect(
      page.getByText("ADR-004: Phased Mainframe DB Cutover")
    ).toBeVisible();

    // UAT Scenario 2: Truth Finder + HITS Conflict Resolution Card
    await expect(
      page.getByText("Data Residency & Legacy DB Cutover Scope")
    ).toBeVisible();
    await expect(page.getByTestId("impact-edge-B-ADR-004")).toContainText(
      "BREAKS_OR_ORPHANS"
    );
    await page
      .getByRole("button", { name: /Promote Option A as Authoritative/i })
      .click();
    await expect(page.getByTestId("conflict-resolution-status")).toContainText(
      "Resolved: Option A Promoted (Option B preserved in lineage)"
    );

    // UAT Scenario 3: Parallel BAITS DAG PlanCard, ApprovalCard & TraceDrawer
    await page.getByTestId("dag-node-node_tech").click();
    await expect(
      page.getByText("chotu://agent/architects/technology-architect@1.0.0")
    ).toBeVisible();
    await expect(page.getByText("STAGE_1_EXACT_URI")).toBeVisible();

    // UAT Scenario 4: 5-Lens Retrospective Elevation & Cross-Engagement Isolation
    await page.getByRole("link", { name: /Universal Knowledge Studio/i }).click();
    const elevateBtn = page.getByRole("button", {
      name: /Approve & Elevate to Universal/i,
    });
    await expect(elevateBtn).toBeDisabled();

    await page
      .getByLabel(/Sanitized Universal Insight/i)
      .fill(
        "Enable dual-write shadow verification 3 weeks prior to mainframe cutover."
      );
    await expect(elevateBtn).toBeEnabled();
    await elevateBtn.click();
    await expect(
      page.getByText("Elevated: engagement://eng_acme#^f-204")
    ).toBeVisible();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd frontend && npx playwright test e2e/chotu-portal-uat.spec.ts`
Expected: FAIL because `frontend/src/App.tsx` and `frontend/index.html` are not yet wired to serve the unified portal routes.

- [ ] **Step 3: Implement `frontend/index.html`, `frontend/src/App.tsx`, `frontend/src/main.tsx`, `frontend/playwright.config.ts`, `backend/tests/e2e/test_portal_full_stack_scenarios.py`, and update `Makefile`**

Wire `App.tsx` inside `QueryClientProvider` and `ScopeProvider` (`initialScope={{ mode: "work", engagementSlug: "eng_acme" }}`) rendering `AppShell` with route navigation between `ExecutiveCockpitPage`, `EngagementWorkspacePage`, `UniversalStudioPage`, and `EvalsPage`, plus the persistent right-hand `PlanCard`, `ApprovalCard`, and `TraceDrawer` dock. Configure `playwright.config.ts` with `webServer: { command: "npm run dev -- --port 5173", port: 5173 }` and update the root `Makefile` with `lint`, `typecheck`, `check-architecture`, `test-unit`, `test-contracts`, `test-integration`, and `test-e2e` targets.

- [ ] **Step 4: Run verification suite to verify all tests pass**

Run: `cd frontend && npx vitest run && npx tsc --noEmit && npx playwright test e2e/chotu-portal-uat.spec.ts`
Expected: PASS across all unit/component suites, zero TypeScript errors, and 100% pass on the Playwright E2E UAT scenario suite.

- [ ] **Step 5: Commit**

```bash
git add frontend/index.html frontend/src/App.tsx frontend/src/main.tsx frontend/playwright.config.ts frontend/e2e/chotu-portal-uat.spec.ts backend/tests/e2e/test_portal_full_stack_scenarios.py Makefile
git commit -m "feat(slice-04): wire unified Chotu Web Control Center and end-to-end Playwright UAT suite"
```
