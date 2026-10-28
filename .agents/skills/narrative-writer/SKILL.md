---
name: narrative-writer
description: >-
  MANDATORY whenever writing or updating any human-facing file on the filesystem
  (extracting topics from conversations or transcripts, writing documentation, overall plans,
  literate TDD implementation plans mixing prose instructions with exact code/tests,
  READMEs, deliverables, guides, or retrospectives). Prevents machine-written AI slop
  by delegating writing to the `narrative-writer` subagent so files read in a natural,
  conversational, professional tone with plain language and zero unnecessary jargon.
---

# Conversational Writer, Topic Extractor & Literate TDD Plan Author (`narrative-writer`) Skill

## Why This Skill Exists: Zero Machine-Written AI Slop
If a human reader cannot read, understand, and reuse a document effortlessly, **it gets discarded** — and all the thinking inside it is wasted.

Left to their default habits, AI agents produce two kinds of unreadable output:
1. **Robotic prose slop**: telegraphic bullet lists, dry tables with no surrounding explanation, inflated vocabulary, and generic filler.
2. **Unexplained code dumps**: walls of TDD task snippets with zero conversational explanation of *why* an interface is shaped that way or *what* failure mode a test is catching.

This skill enforces a hard rule across every harness: whenever an agent writes or updates a human-facing file on disk, it **MUST** route that writing through the `narrative-writer` subagent.

---

## What `narrative-writer` Does

`narrative-writer` is a general-purpose writer, not boxed into a single folder or knowledge format. Use it whenever you need to:
1. **Extract Topics from Conversations or Transcripts**:
   - Separates multi-turn dialogue into distinct **topics** (rather than a chronological dump) and writes each topic up as a clear, self-contained narrative document.
2. **Write Overall Architecture & Slice Group Plans**:
   - High-clarity plans (`docs/planning/slice-*.md`) that explain the *why*, the architecture, the harvest sources, and the verification gates.
3. **Write Literate TDD Implementation Plans (Mix of Conversational Instructions + Exact Code & Tests)**:
   - Combines natural, conversational prose explanations (walking through *what* each task builds, *why* its interfaces are designed that way, and *what* edge cases are being pinned) with the strict `writing-plans` structure (`Goal`, `Architecture`, `Global Constraints`, `Review Focus`, `Files`, `Interfaces: Consumes/Produces`, and `- [ ] Step 1..5` TDD cycles with runnable test assertions and exact signatures).
4. **Write or Update Any Human-Facing Documentation**:
   - READMEs, architecture explanations, onboarding guides, deliverables, and research summaries.

---

## How to Invoke `narrative-writer` Across Any Harness

1. **In Antigravity (`agy` CLI / IDE)**:
   - If `narrative-writer` is not yet registered in the session, call `define_subagent` with `Name: "narrative-writer"`, `EnableWriteTools: true`, and the system prompt from [`.agents/agents/narrative-writer.md`](../../agents/narrative-writer.md).
   - Call `invoke_subagent` with `TypeName: "narrative-writer"`, passing the full prompt instructions (including the `narrative-writer.md` rules and, when writing TDD plans, the `writing-plans` structure), source context, and target file path(s).
2. **In Claude Code (`claude`) / GitHub Copilot (`copilot`)**:
   - Delegate directly to the `narrative-writer` subagent in [`.claude/agents/narrative-writer.md`](../../.claude/agents/narrative-writer.md) or [`.github/agents/narrative-writer.agent.md`](../../.github/agents/narrative-writer.agent.md).
3. **In Harnesses Without Subagent Spawning**:
   - Read and follow [`.agents/agents/narrative-writer.md`](../../agents/narrative-writer.md) directly before writing any markdown or prose file to disk.
