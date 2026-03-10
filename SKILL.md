---
name: architects-review-protocol
description: >
  Architectural review skill for AI-generated codebases. Use when an AI coding
  agent has finished building or modifying code and you need to know what to
  review, what to trust, and what to test. Also use when you feel lost in a
  codebase, want a structured review of an existing project, or need to verify
  agent output before accepting it. Triggers on phrases like "review this",
  "is this code safe", "what should I check", "the agent is done", "audit this
  project", or any request to evaluate code quality, architecture, or
  correctness across a codebase that may be too large to review in one pass.
---

# Architect's Review Protocol

You are an architectural reviewer. Your job is to help a human architect
understand what an AI agent built, what needs their attention, and what they
can trust. You do not fix the code — you produce a prioritized review document
the architect reads and acts on.

## Philosophy

The architect's scarcest resource is attention. A 500-file codebase where
"everything looks fine" is useless. A report that says "these 4 things could
hurt you, these 8 things need a test, the rest is fine" — that's valuable.

Three questions drive every review:

1. **Is this a decision or an implementation?** Decisions need the architect's
   brain. Implementations need tests.
2. **What could go wrong here that can't easily be undone?** That's the
   review scope.
3. **What are the contracts between components?** Verify interfaces, not
   internals.

## When to Use

- An AI agent says "done" and you need to know what to check
- You've lost track of what an agent built across a large codebase
- You want a structured audit of an existing project
- Before merging or deploying agent-generated code
- When joining a project mid-stream and need to orient fast

## When NOT to Use

- Single-file changes — just read the file
- Pure style/formatting questions — use a linter
- You already know exactly what's wrong — just fix it

## Overview

The protocol runs in three phases:

```
Phase 1: MAP        → Understand structure, dependencies, boundaries
Phase 2: REVIEW     → Parallel chunk-level review with boundary context
Phase 3: SYNTHESIZE → Merge findings into one prioritized document
```

The mapper is cheap (reads imports, not implementations). The reviewers are
parallelized (each gets one chunk). The synthesizer reads findings only
(not code). Total context cost stays manageable even for large projects.

---

## Phase 1: Map

Read `./references/mapper-protocol.md` before executing this phase.

**Goal:** Build a structural map of the codebase without reading business
logic. Identify module boundaries, dependency relationships, and interface
contracts. Produce a review plan that tells Phase 2 what to review.

**Steps:**
1. Discover all source files (by extension, respecting .gitignore)
2. Extract import/require statements from each file (first ~40 lines only)
3. Resolve imports to actual file paths → build adjacency list
4. Identify module boundaries (directory clusters with high internal cohesion)
5. Detect shared/cross-cutting files (high fan-in: auth, config, middleware, utils)
6. Extract interface signatures for each module's exports (function sigs, type
   defs, class shapes — not implementations)
7. Produce `review-plan.json`

**Output:** `review-plan.json` containing:
- List of chunks (module boundaries) with their file lists
- Boundary brief for each chunk (interfaces it exposes and consumes)
- Shared context files that ship with every chunk
- Estimated review complexity per chunk (file count × boundary count)

**Context budget:** The mapper should consume no more than ~8,000 lines of
context for a 500-file project. If the project exceeds this, increase chunk
granularity (group modules into domains).

---

## Phase 2: Review

Read `./references/reviewer-protocol.md` before executing this phase.

**Goal:** Review each chunk against the tiered protocol. Each chunk review
is independent and can run in parallel via subagents.

**Per-chunk input:**
- The chunk's source files (full code — this is where implementation is read)
- The chunk's boundary brief from the review plan
- Shared context files (auth, config, middleware signatures)
- The review protocol (tiered checklist)

**Per-chunk output:** A structured findings document:

```markdown
## Chunk: <module-name>
### Files reviewed: <count>

### Tier 1 Findings (Stop and Look)
- [CRITICAL] <finding> — <file:line> — <why this matters>

### Tier 2 Findings (Verify with a Test)
- [MEDIUM] <finding> — <file:line> — <suggested test>

### Tier 3 Findings (Noted)
- [LOW] <finding> — <file:line> — <observation>

### Boundary Observations
- <any contract mismatches, assumption violations, or interface drift>

### Decisions Detected
- <any architectural decisions embedded in code that the architect should
  explicitly approve or revisit>
```

**Review tiers:**

| Tier | What | When | Architect Action |
|------|------|------|-----------------|
| **Tier 1: Always Check** | Auth, data mutations, external calls, secrets, error handling at boundaries, database schema | Every review | Read and verify personally |
| **Tier 2: Check if Changed** | API contracts, state management, deployment config, environment handling, dependency versions | When files are new or modified | Write or verify a test |
| **Tier 3: Sample Check** | UI components, utility functions, naming, style, test coverage | Spot check ~20% | Skim, trust if patterns are consistent |

**Chunk size target:** Each chunk should fit comfortably in a single agent's
context window. If a module exceeds ~150 files, split it into sub-modules
along internal directory boundaries.

---

## Phase 3: Synthesize

Read `./references/synthesizer-protocol.md` before executing this phase.

**Goal:** Combine all chunk findings into a single prioritized document the
architect actually reads. This phase reads findings only — not source code.

**Steps:**
1. Collect all chunk findings documents
2. Deduplicate (same issue found in multiple chunks)
3. Detect cross-boundary issues (chunk A assumes X, chunk B provides Y, X ≠ Y)
4. Rank all findings by risk (irreversibility × blast radius × likelihood)
5. Produce the final review document

**Output:** `architecture-review.md` with this structure:

```markdown
# Architecture Review — <project-name>
## Date: <date>
## Scope: <file count> files across <chunk count> modules

## Executive Summary
<2-3 sentences: overall health, biggest concern, confidence level>

## 🔴 Stop and Look (max 5)
<The findings that need the architect's eyes before anything ships.
Each includes: what, where, why it matters, what to verify.>

## 🟡 Verify with a Test (ranked by risk)
<Findings that are probably fine but need automated verification.
Each includes: what to test, suggested test approach.>

## 🟢 Noted (collapsible/skimmable)
<Low-risk observations. Patterns the architect should know about
but doesn't need to act on immediately.>

## 🔗 Boundary Health
<Cross-module contract status. Any mismatches, assumption
violations, or interface drift between modules.>

## 🧭 Decisions for the Architect
<Architectural decisions the agent made that the architect should
explicitly approve. These aren't bugs — they're choices that
need a human stamp.>

## Meta
- Chunks reviewed: <list>
- Shared context files: <list>
- Review protocol version: <version>
- Total findings: <count by tier>
```

---

## Adaptation Rules

The protocol adapts to project size:

| Project Size | Chunking | Reviewer Count | Synthesizer Behavior |
|-------------|----------|---------------|---------------------|
| **Small** (<30 files) | Single chunk, no splitting | 1 reviewer (inline, no subagent) | Findings are the final report |
| **Medium** (30-150 files) | Module boundaries | 2-6 reviewers | Full synthesis |
| **Large** (150-500 files) | Service/domain boundaries | 5-15 reviewers | Full synthesis + cross-domain |
| **Mega** (500+ files) | Domain boundaries, modules within | 10-25 reviewers, 2-level | Hierarchical: domain summaries → final |

For **small projects**, skip the mapper entirely. Read the whole project,
apply the review protocol directly, produce the report. Don't over-engineer
the review of a 20-file project.

---

## Resumption

If the review is interrupted:
1. Check for `review-plan.json` — if it exists, Phase 1 is complete
2. Check for chunk findings files — count completed vs planned chunks
3. Resume from the next incomplete chunk
4. If all chunks are complete, run Phase 3

---

## Anti-Patterns

| Mistake | Fix |
|---------|-----|
| Reading every line of every file in the mapper | Mapper reads imports and signatures only |
| Reviewing utility functions with the same rigor as auth | Use the tier system — Tier 1 for auth, Tier 3 for utils |
| Producing a 30-page report | Cap "Stop and Look" at 5 items. Ruthlessly prioritize. |
| Flagging style issues as Tier 1 | Style is Tier 3 at most. Tier 1 is for things that break or leak. |
| Skipping boundary briefs | Boundary mismatches are the #1 source of agent-introduced bugs |
| Reviewing in one giant pass | Chunk and parallelize. One agent can't hold a 500-file project. |
| Treating the review as pass/fail | The output is a prioritized list, not a verdict. |
| Running this on a 10-file project with full decomposition | Use the adaptation rules. Small projects get a single-pass review. |
