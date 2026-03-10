# Architect's Review Protocol

A structured 3-phase architectural review skill for [Claude Code](https://claude.ai/claude-code). Designed for reviewing AI-generated codebases — or any codebase too large to review in one pass.

**What you get:** A prioritized review document that tells you what needs your attention, what needs a test, and what you can trust.

**Time investment:** 5–20 minutes depending on project size.

## How It Works

```
Phase 1: MAP        → Understand structure, dependencies, boundaries
Phase 2: REVIEW     → Parallel chunk-level review with tiered checklist
Phase 3: SYNTHESIZE → Merge findings into one prioritized document
```

The protocol adapts to project size:

| Size | Strategy |
|------|----------|
| < 30 files | Single-pass review, no chunking |
| 30–150 files | Module-level chunks, 2–6 parallel reviewers |
| 150–500 files | Service/domain boundaries, 5–15 reviewers |
| 500+ files | Hierarchical: domain summaries → final report |

## Installation

Clone this repo and add the skill to your Claude Code configuration:

```bash
git clone https://github.com/ahmed-aisyst/architects-review-protocol.git ~/.claude/skills/architects-review-protocol
```

Or add it as a skill reference in your project's `.claude/settings.json`:

```json
{
  "skills": [
    "~/.claude/skills/architects-review-protocol/SKILL.md"
  ]
}
```

## Usage

Trigger the skill with phrases like:

- "review this codebase"
- "the agent is done — what should I check?"
- "audit this project"
- "is this code safe to ship?"

The skill produces an `architecture-review.md` with:

- **Stop and Look** (max 5) — findings that need your eyes before shipping
- **Verify with a Test** — things that are probably fine but need automated verification
- **Noted** — low-risk observations grouped by category
- **Boundary Health** — cross-module contract status
- **Decisions for the Architect** — choices the agent made that you should explicitly approve

## File Structure

```
SKILL.md                           # Main skill definition and orchestration
references/
  mapper-protocol.md               # Phase 1: structural mapping protocol
  reviewer-protocol.md             # Phase 2: chunk-level review protocol
  synthesizer-protocol.md          # Phase 3: synthesis and final report
```

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

## License

[MIT](./LICENSE)
