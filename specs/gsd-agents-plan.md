# GSD Agent Routing - Strategic Plan

## Overview

Augment the GSD meta-prompting system so its orchestrator commands dynamically route tasks to specialized agents from a configurable pool, instead of hardcoding `subagent_type="general-purpose"` in every Task tool call. This also adds optional verification agents after execution waves.

The change affects 2 commands that spawn subagents (`execute-phase`, `execute-plan`), 2 templates that document the hardcoded type (`subagent-task-prompt.md`, `continuation-prompt.md`), the config template, the `new-project` command, and both execute workflows. One new template is created for verification prompts.

**Implementation strategy:** All changes should be implemented on a feature branch. If agent routing causes issues, the branch can be reverted as a unit.

## Objectives

- Replace all hardcoded `subagent_type="general-purpose"` references with dynamic agent resolution
- Introduce a 3-level resolution chain: user prompt override > config.json agents section > fallback to "general-purpose"
- Add optional verification wave after each execution wave
- Maintain full backwards compatibility: projects without an `agents` key in config.json behave identically to today

## Scope

### In Scope

- Adding `agents` section to `templates/config.json` (empty by default)
- Removing hardcoded `subagent_type="general-purpose"` from template documentation (subagent-task-prompt.md, continuation-prompt.md)
- Creating a new `templates/verify-prompt.md` verification agent template
- Adding an optional agent-configuration step to `new-project.md`
- Adding agent routing logic to `execute-phase.md` command and workflow (including verification wave)
- Adding agent routing logic to `execute-plan.md` command and workflow

### Out of Scope

- Agent definitions themselves (these live in user's `.claude/agents/` or org repos -- not GSD's concern)
- Changes to `map-codebase.md` (uses Explore agents for a different purpose, intentionally separate)
- Runtime agent health checking or fallback chains (if an agent fails, standard GSD failure handling applies)
- Agent marketplace, discovery, or installation tooling
- Changes to the PLAN.md execution model (tasks, commits, summaries remain unchanged)
- Changes to checkpoint handling (checkpoints work the same regardless of agent type)
- Modifying how SUMMARY.md, STATE.md, or ROADMAP.md are structured
- Planning delegation (spawning planner/reviewer subagents) -- separate feature, not a routing change
- Post-phase external tool hooks (shell commands are not agents -- belongs in a separate `hooks` feature)

## Methodology

### Resolution Chain

The orchestrator resolves which agent to use for each Task tool call via a priority chain:

```
Priority 1: User prompt override (per-invocation)
    "use typescript-pro for this phase"
    -> Orchestrator honors explicit user instruction

Priority 2: .planning/config.json agents section (per-project)
    "agents": { "execution": ["org:agent-a"] }
    -> If one agent: use it for all plans. If multiple: orchestrator picks the most appropriate agent for each task based on task content and agent descriptions.

Priority 3: Fallback
    -> "general-purpose" (current behavior, always available)
```

### Agent Selection Logic

When the orchestrator has execution agents in the pool, selection is deterministic with no guessing:

1. If `agents.execution` has exactly one agent: use it for all plans
2. If `agents.execution` has multiple agents: orchestrator picks the most appropriate agent for each task based on task content and agent descriptions. User override can explicitly specify an agent, which takes priority over orchestrator selection.
3. If `agents.execution` is empty or missing: fallback to "general-purpose"

No domain matching. No keyword heuristics. The orchestrator picks from the configured pool based on task content. User prompt override always takes priority. This keeps the orchestrator lean and eliminates fuzzy matching bugs.

### Verification Flow

After each execution wave completes (optional, only when `agents.verification` is configured):

1. Orchestrator spawns verification agent with verify-prompt template
2. Verification agent checks: compilation, linting, test suite, plan success criteria
3. If pass: proceed to next wave
4. If fail: report issues, ask user how to proceed

### Config Resolution Note

The orchestrator always reads `.planning/config.json` relative to the current working directory. In monorepo setups where each sub-project has its own `.planning/` directory, GSD already scopes to CWD, so the correct config is always resolved automatically.

## Key Milestones

0. Pre-flight verified -- confirmed Task tool accepts custom `subagent_type` values
1. Config schema defined and template updated -- agents section exists with backwards-compatible empty default
2. Templates cleaned up -- all hardcoded "general-purpose" references removed from documentation, verify-prompt created
3. new-project offers optional agent configuration -- users can set up agent pool during project init
4. execute-phase and execute-plan route dynamically -- core routing works end-to-end
5. Integration validated -- end-to-end smoke test confirms routing works with and without config

## Risk Assessment

| Risk | Impact | Mitigation |
|------|--------|------------|
| Config.json parsing fails on missing agents key | High - breaks all execution | Every read of agents key uses defensive access: `config.agents \|\| {}`. Empty = fallback to general-purpose. Explicit in every routing section. |
| Agent name typo in config leads to Task tool error | Medium - plan execution fails | Orchestrator does not validate agent names. Task tool handles unknown types. Standard GSD failure handling (report, ask user) applies. Document this in config template comments. |
| Routing logic adds too much orchestrator context | Medium - context budget exceeded | Keep routing to ~5 lines of pseudocode per command. No AI-based routing. No domain matching. Deterministic selection only. |
| Backwards compatibility regression | High - breaks existing projects | Every routing path has explicit fallback to "general-purpose". No agents key = identical behavior to today. Test with both old and new config formats. |
| Verification wave adds unwanted overhead | Low - slows execution | Verification only runs when agents.verification is configured. Empty config = no verification wave. |
| Task tool rejects custom subagent_type values | Critical - entire plan fails | Pre-flight verification task (Group 0) confirms this works before any file changes. |

## Dependencies

### Internal Dependencies (within this change set)

- Config template must be updated before any command changes (everything reads config)
- verify-prompt.md must exist before execute-phase references it
- Template cleanup (subagent-task-prompt, continuation-prompt) should happen before command changes for consistency

### External Dependencies

- Agent definitions must exist in user's `.claude/agents/` directory or be available via `org:agent-name` format
- Claude Code Task tool must support custom `subagent_type` values (must be verified in pre-flight task before implementation begins)
- No new CLI tools, packages, or infrastructure required

## Success Criteria

- Pre-flight verification confirms Task tool accepts custom `subagent_type` values
- All 6 current hardcoded `subagent_type="general-purpose"` locations in commands/templates are replaced with dynamic resolution
- All 4 hardcoded references in workflow files are updated
- A project with NO agents key in config.json behaves identically to today (backwards compatible)
- A project WITH agents configured routes to the correct agent for each Task call
- The resolution chain (user override > config > fallback) works correctly at each priority level
- Verification wave runs when configured, skips when not
- All changes are in existing GSD files (except the one new verify-prompt template)
- No increase in orchestrator context budget beyond ~5% for routing logic
- End-to-end smoke test passes with both configured and unconfigured agents

## Files Changed

| # | File | Type | Change Summary |
|---|------|------|----------------|
| 1 | `get-shit-done/templates/config.json` | Modify | Add `agents` section with empty defaults |
| 2 | `get-shit-done/templates/subagent-task-prompt.md` | Modify | Remove hardcoded `subagent_type="general-purpose"` |
| 3 | `get-shit-done/templates/continuation-prompt.md` | Modify | Remove hardcoded `subagent_type="general-purpose"` |
| 4 | `get-shit-done/templates/verify-prompt.md` | Create | New verification agent prompt template |
| 5 | `commands/gsd/new-project.md` | Modify | Add optional agents setup step |
| 6 | `commands/gsd/execute-phase.md` | Modify | Add routing and verification wave |
| 7 | `get-shit-done/workflows/execute-phase.md` | Modify | Add routing step and verification step |
| 8 | `commands/gsd/execute-plan.md` | Modify | Add routing to spawn step |
| 9 | `get-shit-done/workflows/execute-plan.md` | Modify | Update spawn examples with routing |
