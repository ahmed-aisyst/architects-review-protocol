# Synthesizer Protocol

## Purpose

Combine all chunk-level findings into a single prioritized review document
the architect can act on in 10 minutes or less. You read findings documents
only — not source code. Your job is to deduplicate, cross-reference, rank,
and present.

## Inputs

You will receive:
1. **All chunk findings documents** from Phase 2
2. **The review plan** from Phase 1 (for project metadata and chunk relationships)
3. **Shared context chunk findings** (if cross-cutting files were reviewed separately)

## Step 1: Collect and Parse

Read all chunk findings. Build a flat list of all findings across all chunks,
preserving their tier, severity, file location, and originating chunk.

## Step 2: Deduplicate

The same issue may appear in multiple chunks because:
- Two modules both call the same unprotected endpoint
- The same anti-pattern is used in several places (e.g., missing error handling)
- A shared dependency issue surfaces in every chunk that imports it

Deduplication rules:
- **Identical file:line** → merge into one finding, note all chunks that flagged it
- **Same pattern, different files** → merge into one finding that says "pattern found
  in N files across M modules" and list the most critical instance
- **Same root cause** → merge into one finding that identifies the root cause, list
  all affected locations as sub-items

Do not over-deduplicate. Two modules both missing auth is ONE finding (pattern).
Module A missing auth and Module B having a SQL injection are TWO findings (different issues).

## Step 3: Cross-Boundary Analysis

This is the step chunk-level reviewers cannot do. Compare findings across chunks
to detect issues that only become visible when you see the full picture:

### Contract Contradictions
Check boundary briefs for mismatches:
- Module A calls `PaymentService.charge(amount, currency)` but Module B's
  implementation signature is `charge(amount_cents, currency_code)` — the
  parameter names suggest a unit mismatch (dollars vs cents)
- Module A expects `{success: boolean, id: string}` but Module B returns
  `{ok: boolean, transaction_id: string}` — field name mismatch

### Error Propagation Gaps
Trace error handling across module boundaries:
- Module A throws `PaymentError` but Module B catches `HTTPError` — the error
  type doesn't match, so Module A's errors will be unhandled in Module B
- Module A returns `null` on failure, Module B doesn't null-check → potential
  null reference error at runtime

### Auth Boundary Leaks
Check the full request flow path:
- Route → middleware → service → database: is auth checked at every step?
- If the auth middleware is in the shared context, verify every route chunk
  actually uses it (look for routes that bypass the auth middleware)

### Implicit Dependencies
Look for patterns where modules depend on each other's behavior without
an explicit import:
- Module A writes to a queue, Module B reads from it — if neither chunk
  reviewer saw the other side, this dependency is invisible
- Module A sets a cache key, Module B reads it — same pattern
- Module A modifies a database row, Module B reads it expecting certain state

Flag these as "implicit coupling" findings.

### Decision Conflicts
Check if different chunks made contradictory architectural decisions:
- Module A uses UTC timestamps, Module B uses local time
- Module A handles errors with exceptions, Module B uses result types
- Module A uses camelCase in API responses, Module B uses snake_case
- Module A retries on failure, Module B fails fast for the same operation

## Step 4: Risk Ranking

Score each finding using three dimensions:

**Irreversibility** (1-3):
- 1: Easily fixed, no data impact (rename a variable, add a log line)
- 2: Requires a migration or config change (add an index, fix a schema)
- 3: Cannot be undone without data loss or downtime (wrong encryption, data corruption)

**Blast Radius** (1-3):
- 1: Affects one function or component
- 2: Affects one module or user-facing flow
- 3: Affects multiple modules, all users, or external systems

**Likelihood** (1-3):
- 1: Would only trigger under unusual conditions
- 2: Could trigger in normal operation under load or edge cases
- 3: Will trigger in normal operation

**Risk Score** = Irreversibility × Blast Radius × Likelihood (max 27)

Ranking:
- Score 18-27 → Tier 1 (Stop and Look)
- Score 8-17 → Tier 2 (Verify with a Test)
- Score 1-7 → Tier 3 (Noted)

A finding may be promoted from its original tier based on this scoring. For
example, a Tier 2 finding in one chunk may become Tier 1 when cross-boundary
analysis reveals it affects three modules.

A finding should never be demoted — if a chunk reviewer called it CRITICAL,
respect that assessment. You can only promote, not demote.

## Step 5: Cap and Prioritize

The "Stop and Look" section is capped at 5 items. If more than 5 findings
score in the Tier 1 range, include the top 5 by risk score and move the
rest to a "Tier 1 Overflow" subsection that the architect can optionally expand.

Why the cap: a report with 15 critical findings triggers analysis paralysis.
The architect looks at 5, acts on them, and then can decide whether to dig
deeper. This is by design.

## Step 6: Produce the Final Report

Use this exact template:

```markdown
# Architecture Review — <project-name>
**Date:** <date>
**Scope:** <file count> files across <chunk count> modules
**Protocol Version:** 1.0

---

## Executive Summary

<2-3 sentences answering: How healthy is this codebase? What's the single
biggest concern? How confident are you in the agent's output overall?>

<One of these confidence ratings:>
- **🟢 High Confidence** — No critical findings. Agent output is solid.
- **🟡 Moderate Confidence** — Some concerns need architect attention before shipping.
- **🔴 Low Confidence** — Significant issues found. Do not ship without addressing Tier 1 items.

---

## 🔴 Stop and Look

<Max 5 items. Each formatted as:>

### 1. <Title — clear, specific, not vague>
**Risk Score:** <N>/27 (Irreversibility: N, Blast Radius: N, Likelihood: N)
**Location:** <file:line> (and N other locations)
**What:** <1-2 sentences describing the issue>
**Why it matters:** <1 sentence on the consequence if shipped as-is>
**What to verify:** <specific action the architect should take>

<If more than 5 Tier 1 findings exist:>
### Overflow (N additional Tier 1 findings)
<Brief list of remaining items with file locations>

---

## 🟡 Verify with a Test

<Ranked by risk score. Each formatted as:>

- **<Title>** — <file:line> — <what to test in one sentence>

---

## 🟢 Noted

<Grouped by category to reduce noise:>

**Code Quality:** <N findings>
- <brief finding> — <file:line>

**Testing:** <N findings>
- <brief finding> — <file:line>

**Style & Consistency:** <N findings>
- <brief finding> — <file:line>

---

## 🔗 Boundary Health

<For each module boundary that has issues:>

**<Module A> ↔ <Module B>:**
- <Specific contract mismatch or assumption violation>

**Implicit Coupling Detected:**
- <Any event/queue/cache-based dependencies found>

<If all boundaries look healthy:>
All module boundaries verified — contracts match, error types align,
no implicit coupling detected.

---

## 🧭 Decisions for the Architect

<Each formatted as:>

- **<Decision made>** — <file:line>
  - Chosen: <what the agent chose>
  - Alternatives: <what else could have been chosen>
  - Risk if wrong: <consequence of this being the wrong choice>

---

## Meta

| Metric | Value |
|--------|-------|
| Files scanned | <N> |
| Modules reviewed | <N> |
| Tier 1 findings | <N> |
| Tier 2 findings | <N> |
| Tier 3 findings | <N> |
| Boundary issues | <N> |
| Decisions flagged | <N> |
| Cross-cutting files | <list> |
| Protocol version | 1.0 |
```

## Report Quality Checklist

Before delivering the report, verify:

- [ ] Executive summary is 2-3 sentences, not a paragraph
- [ ] "Stop and Look" has at most 5 items
- [ ] Every Tier 1 finding has a specific "What to verify" action
- [ ] Every Tier 2 finding has a concrete test suggestion
- [ ] Tier 3 findings are grouped, not listed individually per-file
- [ ] Boundary Health section addresses cross-module contracts, not single-module issues
- [ ] Decisions section only contains actual choices, not bugs or style issues
- [ ] No findings are vague ("code could be improved") — every finding is specific
- [ ] Risk scores are calibrated (a missing comment is not score 18)
- [ ] The report could be read and acted on in 10 minutes or less
