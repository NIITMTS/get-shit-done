# Planner Output Contract Extraction - Strategic Plan

## Overview

Custom planner agents (configured via `agents.planning` in `.planning/config.json`) currently receive planning context and a `<return_protocol>` block but have NO instructions about WHERE to write PLAN.md files, WHAT format to use, HOW to update ROADMAP.md, or WHEN to git commit. All of this knowledge is baked into the 1386-line `agents/gsd-planner.md` agent definition. This plan creates a shared output contract reference file that custom agents receive via prompt injection.

**Key design decision (Alternative B):** The built-in gsd-planner.md is NOT modified. It keeps its inline output steps unchanged. The output contract is a NEW file consumed ONLY by custom agents via plan-phase.md injection. This avoids touching a battle-tested 1386-line agent definition for refactoring purity.

## Objectives

- Create a reference file (`get-shit-done/references/planner-output-contract.md`) containing the output contract for planner agents
- Modify `commands/gsd/plan-phase.md` to read and inject this contract into custom agent prompts (both initial planning and revision mode)
- Handle both initial planning (`docs($PHASE)`) and revision mode (`fix($PHASE)`) commit formats in the contract
- Keep the contract under 45 lines to limit prompt size impact

## Scope

### In Scope

- Creating `get-shit-done/references/planner-output-contract.md` with file paths, ROADMAP.md update rules, and git commit conventions (both initial and revision modes)
- Modifying `commands/gsd/plan-phase.md` to read the contract file and append it to custom agent prompts in step 8 (initial planning) and step 12 (revision loop)

### Out of Scope

- Changes to `agents/gsd-planner.md` -- the built-in planner keeps its inline steps untouched (Alternative B)
- Changes to `get-shit-done/templates/phase-prompt.md` (the PLAN.md format template stays as-is)
- Changes to planning METHODOLOGY (philosophy, discovery, task breakdown -- these stay in agent definitions)
- Changes to `<return_protocol>` blocks (already handled by the planning-agents spec)
- Changes to the researcher or checker agents
- New config.json keys or schema changes

## Methodology

### Alternative B: Contract as Custom-Agent-Only Injection

The output contract is consumed in ONE context:

- **As appended content in plan-phase.md**: For custom agents, the orchestrator reads the file content and appends it to the prompt (same pattern as `<return_protocol>` injection). The `@` syntax does not work across Task() boundaries, so content must be inlined.

The built-in gsd-planner does NOT consume this file. It keeps its own inline steps. This means the same procedural knowledge exists in two places (gsd-planner inline steps and the reference file), but both are controlled by us, and the risk of behavioral drift is far lower than the risk of breaking gsd-planner.

### Dual-Mode Commit Format

The contract includes both commit formats because the same custom agent handles both initial planning and revision:

- **Initial mode:** `docs($PHASE): create phase plan` (matches gsd-planner.md line 1233)
- **Revision mode:** `fix($PHASE): revise plans based on checker feedback` (matches gsd-planner.md line 962)

The orchestrator already tells the agent which mode it's in via the `**Mode:**` field in planning_context (step 8) and revision_context (step 12). The contract says "use docs() for initial, fix() for revision."

### Prescriptive by Design

The output contract is intentionally prescriptive. Custom agents swap planning METHODOLOGY (how to think about task decomposition), not output FORMAT (where files go, how ROADMAP.md is updated). The orchestrator (plan-phase.md) and downstream consumer (execute-phase) depend on specific output conventions. This is an interface contract, not a style guide.

### Line Budget

Target: 45 lines maximum for the contract file. The custom agent already receives ~97 lines of injected content (planning_context + downstream_consumer + quality_gate + return_protocol). Adding 45 lines brings total to ~142 lines -- acceptable for operational instructions.

## Key Milestones

1. Reference file created with all 3 output sections (file paths, roadmap update, git commit for both modes)
2. plan-phase.md modified to inject contract for custom agents in steps 8 and 12

## Risk Assessment

| Risk | Impact | Mitigation |
|------|--------|------------|
| Custom agent prompt becomes too long with appended contract | Medium | Contract capped at 45 lines; total injected content stays under 150 lines |
| Contract drifts from gsd-planner inline steps over time | Low | Both are in the same repo; document the duplication in the contract file header |
| plan-phase.md reads contract file at runtime but file path is wrong | Medium | Use exact path `~/.claude/get-shit-done/references/planner-output-contract.md`; verify with smoke test |
| Custom agent in revision mode uses wrong commit format | High | Contract includes BOTH formats keyed by mode; revision_context already sets Mode: revision |

## Dependencies

- `get-shit-done/templates/phase-prompt.md` must continue to exist at its current path (referenced by the output contract)
- `get-shit-done/references/planning-config.md` must exist (referenced for `commit_docs` config details)
- The `<agent_routing>` section in `plan-phase.md` (from the planning-agents spec) must be in place -- defines the custom vs built-in branching. **Status: Already merged** (commit cc1e965)
- The `<return_protocol>` injection pattern in `plan-phase.md` is the model for how the output contract gets appended (same mechanism: read file, append to prompt)

## Success Criteria

- A custom planning agent (not gsd-planner) produces PLAN.md files at the correct path with correct naming convention
- A custom planning agent updates ROADMAP.md with plan count and plan checkboxes
- A custom planning agent in revision mode uses `fix($PHASE)` commit format, not `docs($PHASE)`
- A custom planning agent respects `COMMIT_PLANNING_DOCS` config toggle
- The built-in gsd-planner is completely untouched (zero regression risk)
- No content from `phase-prompt.md` is duplicated in the output contract
- The output contract file is 45 lines or fewer
- `get-shit-done/references/planner-output-contract.md` follows the same structural patterns as other reference files in that directory (XML-style sections, concise, focused)
