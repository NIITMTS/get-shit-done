# Planner Debate Log

## Debate Log (Round 2)

### Feedback Point 1: Over-engineered (9 tasks / 3 phases for ~50 lines)

**Verdict: CONCEDE in full.**

The advocate is right. Three adjacent edits to the same file (Tasks 2.1, 2.2, 2.3) are one task. Verification tasks (2.4, 3.4) are completion criteria, not standalone work items. The original plan had 9 tasks across 3 phases for extracting ~52 lines of content and making 2 file edits. That is ceremonial bloat.

Revised plan: **3 tasks, 1 phase.** Task 1 creates the reference file. Task 2 modifies plan-phase.md. Task 3 is a single integration smoke test that validates everything works together. No separate "structural integrity" tasks -- those checks belong in completion criteria of the task that made the edit.

### Feedback Point 2: Alternative B -- don't touch gsd-planner.md

**Verdict: CONCEDE. Accept Alternative B.**

The advocate's argument is strong:

1. **Risk asymmetry.** gsd-planner.md is 1386 lines of battle-tested agent definition. Replacing 3 working inline steps with `@` references creates regression risk for zero functional gain. The built-in planner already works. The problem is that *custom agents* don't have these instructions -- that's what we're solving.

2. **Duplication is acceptable.** The output contract reference file and gsd-planner's inline steps will say the same thing in two places. Both are controlled by us. If the contract drifts from the inline steps, the damage is limited to custom agents getting slightly different instructions -- and we can fix that later. The alternative (breaking gsd-planner for refactoring purity) is worse.

3. **Scope reduction.** Removing the gsd-planner.md modification eliminates a whole class of risk (malformed XML tags, broken step references, `@` resolution failures) and simplifies the plan from 3 touch-points to 2 (create reference file + modify plan-phase.md).

**What this means:** The output contract is a NEW file consumed ONLY by custom agents via plan-phase.md injection. gsd-planner.md is untouched. The original Phase 2 (all of Tasks 2.1-2.4) is eliminated entirely.

### Feedback Point 3: Revision-mode commit format conflict (CRITICAL)

**Verdict: CONCEDE. This is a real bug in the original plan.**

The problem: The output contract contains `docs($PHASE): create phase plan` as the git commit format. But in revision mode, the correct format is `fix($PHASE): revise plans based on checker feedback` (gsd-planner.md line 962). A custom agent receiving the output contract in revision mode would see `docs($PHASE)` and use the wrong commit prefix.

**Solution: The output contract includes BOTH commit formats, keyed by mode.**

The `<git_commit>` section in the contract will contain:

- **Initial planning mode:** `docs($PHASE): create phase plan ...`
- **Revision mode:** `fix($PHASE): revise plans based on checker feedback`

The revision_context block in plan-phase.md step 12 already tells the agent `Mode: revision`. The contract says "if mode is revision, use the fix() format." This makes the contract self-contained -- the custom agent has all the information it needs regardless of which mode the orchestrator spawns it in.

This is better than the original plan's approach of "revision has its own commit step; contract covers initial planning only" -- that left custom agents in revision mode without git commit guidance at all.

### Feedback Point 4: Prompt size budget concern

**Verdict: PUSH BACK (partially).**

The concern: custom agents already receive ~54 lines of planning_context + ~10 lines downstream_consumer + ~10 lines quality_gate + ~23 lines return_protocol = ~97 lines. Adding ~40-50 lines of output contract would push it to ~140-150 lines.

Why this is acceptable:

1. **These are operational instructions, not bloat.** The planning_context is project-specific data (state, roadmap, research). The output contract is procedural instructions (where to write, how to commit). Both are necessary for correct output. Cutting the contract means custom agents produce wrong output.

2. **The content is mostly code blocks.** Bash snippets and template references compress well in context. An LLM reading 150 lines of structured instructions with code blocks is not meaningfully impaired compared to 100 lines.

3. **The alternative is worse.** Without the contract, custom agents must independently figure out file naming, ROADMAP.md update rules, and git commit conventions. They will get them wrong, producing outputs the orchestrator can't parse.

**Partial concession:** The plan now includes a line budget target of 45 lines for the contract file. This keeps the total injected content under 150 lines. The contract must be dense and operational, not explanatory.

### Feedback Point 5: Should custom agents ALWAYS follow gsd-planner conventions?

**Verdict: PUSH BACK. Yes, this is intentional and the plan should state so explicitly.**

The output contract is prescriptive because the orchestrator (plan-phase.md) depends on specific outputs:

- PLAN.md files at specific paths with specific naming (`{phase}-{NN}-PLAN.md`)
- ROADMAP.md updated with plan counts and checkboxes
- Git commits with parseable messages
- Signal headers in specific format (already handled by return_protocol)

A custom agent that ignores the file naming convention breaks `execute-phase`. A custom agent that skips ROADMAP.md update leaves stale placeholders. These are not style preferences -- they are interface contracts.

If someone wants truly different output behavior, they should fork the orchestrator (plan-phase.md), not use a custom agent within the existing pipeline. The config system lets you swap the *planning methodology* (how the agent thinks), not the *output format* (what the orchestrator expects).

The revised plan explicitly states this rationale.

---
