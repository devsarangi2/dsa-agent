# Chotu (`dsa-agent`) — Brain Logs & Conversation Retrospectives

This directory preserves the full narrative, reasoning journeys, course-corrections, and architectural discoveries from our design and brainstorming conversations.

Unlike terse ADRs or stripped-down specification tables, **Conversation Retrospectives** are organized **by topic (not by session)** and written as formal, comprehensive documentation so the *why*, the rejected assumptions, the domain realities, and the exact mental models behind Chotu are never lost.

---

## Topic Catalog (`conversation-retrospectives/`)

1. **[2026-10-27 — Identity & The "Chotu Corp" Operating Model](conversation-retrospectives/2026-10-27-01-identity-and-chotu-corp-operating-model.md)**
   - Chotu as a capability-driven autonomous company ("Chotu Corp"); the Conscious vs. Subconscious mind; System-1 Established Paths vs. System-2 Escalation; and the boundary between Internal Tools, Skills/Playbooks, and Independent External Apps (`dsapps-*`).
2. **[2026-10-27 — Hexagonal Microkernel, OOP Philosophy & Sovereign Agent Harness](conversation-retrospectives/2026-10-27-02-hexagonal-microkernel-and-sovereign-harness.md)**
   - Why Hexagonal Architecture (Ports & Adapters) is the framework; demystifying "Agent" vs. "Agent Harness"; why Chotu runs a 100% self-contained in-process Python engine rather than wrapping an external CLI; the `platform/` vs. `inventory/` split; and workspace jailing.
3. **[2026-10-27 — Service Locator & Service Discovery Architecture](conversation-retrospectives/2026-10-27-03-service-discovery-and-service-locator.md)**
   - Constructor DI for infrastructure ports vs. Runtime Service Locator & Discovery for capabilities; canonical `chotu://` URIs; Hybrid Pull + Push registration; the 3-stage Zero-LLM resolution pipeline; and lifecycle-aware activation (`lazy_start`, fallbacks, circuit-breaking).
4. **[2026-10-27 — Classifier-First Routing, Subagents-Only Invariant, Dynamic Workflows & Lifecycle Hooks](conversation-retrospectives/2026-10-27-04-classifier-routing-dynamic-workflows-and-hooks.md)**
   - Closing the orchestration gap in prior repos; upfront work classification (`WorkClassifierPort`); the hard rule that the Lead Orchestrator never touches raw tools; dynamic Subagent DAG composition and probationary crystallization; and the dual-mode Lifecycle Hook engine with origin-tagged recursion fencing.
5. **[2026-10-27 — Disjoint Universal Knowledge Fabric, Engagement-as-Dataset & The Elevation Technique](conversation-retrospectives/2026-10-27-05-knowledge-fabric-engagement-datasets-and-elevation.md)**
   - Why Universal Knowledge is a disjoint fast-recall reference layer rather than an accumulation of project files; closed-world engagement execution; Discover $\rightarrow$ Select $\rightarrow$ Ingest into engagement scope; solving the `okf-wiki/` $\rightarrow$ `pgvector` projection and Small-to-Big retrieval; and engagement closure retrospectives elevated with `engagement_id` as the correlation citation.
6. **[2026-10-27 — Versioned Knowledge, Multi-Perspective Reality & The Truth Finder + HITS Engine](conversation-retrospectives/2026-10-27-06-versioned-knowledge-truth-finder-and-hits-weighing.md)**
   - Why unversioned knowledge creates conflicts and blockers mid-engagement; multi-perspective claims (`client`, `internal_team`, `personal_take`, `work_research`, `personal_rd`); why forced auto-conflict resolution is rejected; and the active Truth Finder + HITS + 2-Step Relationship Make/Break & Plausibility Weighing engine.
7. **[2026-10-27 — Real-World Consulting Scale, Hardened Postgres + `pgvector` Infrastructure & Master Task Queues](conversation-retrospectives/2026-10-27-07-real-world-scale-postgres-hardening-and-master-queues.md)**
   - Rejecting the "single user = small data" fallacy; the real daily volume (10 meetings/day across 3+ perspectives, 20 docs/day, Engagement Master Data, 6–12 month horizons, Work vs. Personal modes); why hardened Postgres + `pgvector` HNSW is mandatory Tier-0 infrastructure; event-sourced per-node DAG state management; and the 4-lane priority Master Task Queue.
