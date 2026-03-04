# Planning Pipeline Agent Routing - Strategic Plan

## Overview

Extend the GSD planning pipeline (`/gsd:plan-phase`) to support custom agent routing, following the same 3-level resolution chain already implemented for the execution pipeline (`/gsd:execute-phase`). Currently, plan-phase hardcodes 3 agent types: `gsd-phase-researcher`, `gsd-planner`, and `gsd-plan-checker`. This change makes all 3 configurable via `config.json` while maintaining backwards compatibility.

The execution pipeline routing (completed in the `gsd-agents` spec) established the 3-level resolution chain pattern (user override > config > fallback). This plan applies that same resolution chain to the planning pipeline.

**What differs from execution routing:** The execution pipeline does zero return_protocol injection -- execution agents (gsd-executor) are spawned with the full plan as the prompt, and the orchestrator checks for SUMMARY.md on disk. Planning agents are different: the orchestrator parses specific signal headers (`## PLANNING COMPLETE`, `## ISSUES FOUND`, etc.) from the agent's response text. Built-in GSD agents know these signals from their agent definitions. Custom agents do not. So this plan adds `<return_protocol>` injection -- a concept that has no equivalent in the execution pipeline.

## Objectives

- Replace all hardcoded agent references in plan-phase.md's 5 Task() spawn points with dynamic resolution
- Introduce the same 3-level resolution chain used by execution: user prompt override > config.json agents key > fallback to built-in GSD agent
- Inject `<return_protocol>` blocks into custom agent prompts so the orchestrator can parse their responses
- Extend config.json schema with `planning`, `research`, and `plan_check` agent keys
- Extend the new-project setup flow to let users configure planning agents
- Maintain full backwards compatibility: projects without planning agent keys behave identically to today

## Scope

### In Scope

- Extending `agents` section in config schema with 3 new keys (`planning`, `research`, `plan_check`)
- Updating `config-agents-example.json` with planning agent examples
- Adding `<agent_routing>` section to `plan-phase.md` with resolution chain and return_protocol definitions
- Modifying 5 Task() spawn points in `plan-phase.md` to use resolved agents
- Extending `new-project.md` Round 3 to ask about planning agents
- Defining `<return_protocol>` blocks for all 3 planning roles (researcher, planner, checker)

### Out of Scope

- Agent definitions themselves (custom agents are the user's responsibility)
- Changes to the execution pipeline routing (already complete)
- Changes to the 3 built-in agent definition files (`gsd-phase-researcher.md`, `gsd-planner.md`, `gsd-plan-checker.md`) -- these already have return protocols in their definitions
- Runtime agent health checking or fallback chains
- Changes to PLAN.md format, RESEARCH.md format, or verification output format
- Model profile changes (custom agents still use the model profile's model selection)
- Workflow files for planning (plan-phase.md is both command and workflow -- there is no separate `workflows/plan-phase.md`)

## Methodology

### Resolution Chain (Same as Execution)

The orchestrator resolves which agent to use for each Task() call via a 3-level priority chain:

```
Priority 1: User prompt override (per-invocation)
    "use python-pro for research"
    -> Orchestrator honors explicit user instruction for that role

Priority 2: .planning/config.json agents section (per-project)
    "agents": { "research": "org:custom-researcher" }
    -> Use the configured agent for that role

Priority 3: Fallback (built-in GSD agent)
    -> gsd-phase-researcher / gsd-planner / gsd-plan-checker
```

### Agent Config Keys

| Key | Type | Role | Fallback |
|-----|------|------|----------|
| `agents.research` | string | Phase researcher | `gsd-phase-researcher` |
| `agents.planning` | string | Plan creator + revision handler | `gsd-planner` |
| `agents.plan_check` | string | Plan verifier | `gsd-plan-checker` |

- All 3 keys are strings (single agent name). Unlike `agents.execution` (array, because multiple plans with different domains run per wave), planning spawns one agent per role per phase -- there is no multi-agent selection to do.
- Users who want different agents per phase can use Priority 1 (user prompt override) at invocation time.
- All keys are optional. Missing = use built-in GSD agent.

### Return Protocol Injection (New -- Not From Execution Pattern)

This concept has no equivalent in the execution pipeline. Execution agents write SUMMARY.md to disk and the orchestrator checks for that file. Planning agents return signal headers in their response text, which the orchestrator parses directly. This is why custom planning agents need explicit signal injection.

**Problem:** Built-in GSD agents have return protocols (signal headers, structured data formats) defined in their agent definition files. Custom agents don't know these signals. The orchestrator needs to parse specific signals like `## PLANNING COMPLETE` or `## ISSUES FOUND` from agent responses.

**Solution:** When spawning a custom agent, the orchestrator appends a `<return_protocol>` XML block to the Task() prompt. This block tells the custom agent exactly what signals to return and what structured data to include.

**When built-in agent is used:** The prompt uses `subagent_type="gsd-planner"` (named agent type). Claude Code loads the agent definition automatically. The agent definition already contains the return protocol. No injection needed.

**When custom agent is used:** The prompt uses `subagent_type="org:custom-planner"`. The custom agent's own definition is loaded. The orchestrator appends the `<return_protocol>` block to the prompt so the custom agent knows what the orchestrator expects.

### Return Protocol Per Role

Only signals that the orchestrator actually parses are included. Signals that exist in the built-in agent definitions but are never parsed by the orchestrator (e.g., `## REVISION COMPLETE`, `## GAP CLOSURE PLANS CREATED`) are excluded from the return_protocol -- including them would mislead custom agents.

| Role | Signals the Orchestrator Parses | Where Parsed | Structured Data |
|------|--------------------------------|--------------|-----------------|
| Researcher | `## RESEARCH COMPLETE`, `## RESEARCH BLOCKED` | Researcher handler (after step 5) | RESEARCH.md sections (confidence, findings, file path) |
| Planner | `## PLANNING COMPLETE`, `## CHECKPOINT REACHED`, `## PLANNING INCONCLUSIVE` | Step 9 (Handle Planner Return) | Plan summary (wave structure, plans created) |
| Checker | `## VERIFICATION PASSED`, `## ISSUES FOUND` | Step 11 (Handle Checker Return) | YAML issues block (plan, dimension, severity, fix_hint) |

**Note on revision spawns (step 12):** The orchestrator does NOT parse a specific signal from the revision planner. After the revision agent returns, the orchestrator routes straight back to the checker (step 10). The revision spawn uses the same 3 planner signals above -- the orchestrator handles them identically to the initial planning spawn.

### Consistency Rule

The same custom agent used for initial planning MUST be used for revision loops. If `org:custom-planner` creates the plans, `org:custom-planner` handles revision after checker feedback. The orchestrator stores the resolved planner agent and reuses it in step 12 (revision loop).

### Default Path Changes (Affects All Projects)

Even without custom agents configured, the default spawn behavior changes for ALL projects:

| Spawn Point | Before | After |
|-------------|--------|-------|
| Researcher | `subagent_type="general-purpose"` + "First, read ~/.claude/agents/gsd-phase-researcher.md..." prompt prefix | `subagent_type="gsd-phase-researcher"` (agent def loaded automatically) |
| Planner | `subagent_type="general-purpose"` + "First, read ~/.claude/agents/gsd-planner.md..." prompt prefix | `subagent_type="gsd-planner"` (agent def loaded automatically) |
| Checker | `subagent_type="gsd-plan-checker"` (already correct) | No change |

The researcher and planner currently use `subagent_type="general-purpose"` and manually instruct the agent to read the GSD agent definition file. This was a workaround. Named agent types (`subagent_type="gsd-planner"`) cause Claude Code to load the agent definition automatically, which is cleaner and matches how the checker already works. The execution pipeline made this same change (from "general-purpose" to "gsd-executor").

This is a behavioral change for all projects, not just those with custom agents. It should be functionally equivalent (the agent gets the same instructions either way) but worth calling out explicitly.

### Workflow Toggle Interaction

The existing `workflow.plan_check` boolean (controls whether the checker runs at all) is separate from `agents.plan_check` (controls which agent runs as checker). When `workflow.plan_check: false`, the entire verification step (step 10) is skipped -- the `agents.plan_check` config is silently ignored. The routing section in plan-phase.md should document this: agent resolution for the checker only happens when `workflow.plan_check` is not disabled.

Similarly, `workflow.research: false` skips the research step entirely, making `agents.research` irrelevant.

### Model Profile Still Applies

Custom agent name overrides which agent definition runs. The model profile (`quality`/`balanced`/`budget`) still controls which AI model powers the agent. This is the same behavior as execution routing.

## Key Milestones

1. Config template updated -- `config-agents-example.json` has planning, research, plan_check examples
2. plan-phase.md routing complete -- all 5 spawn points use dynamic resolution with return_protocol injection
3. new-project.md extended -- Round 3 agent pool questions cover planning agents
4. Integration validated -- static review confirms routing works with and without config

## Risk Assessment

| Risk | Impact | Mitigation |
|------|--------|------------|
| Return protocol signals don't match what orchestrator parses | High - orchestrator can't detect plan completion | Return protocol blocks include ONLY the signals the orchestrator actually parses (verified against plan-phase.md steps 9, 11, and researcher handler). Signals that exist in agent definitions but are never parsed by the orchestrator (REVISION COMPLETE, GAP CLOSURE PLANS CREATED) are excluded. |
| Custom agent ignores return_protocol, returns free-form text | Medium - orchestrator confused | Return protocol block explicitly states signals are REQUIRED. Orchestrator already handles unexpected returns (PLANNING INCONCLUSIVE path). |
| Config key naming conflicts with future features | Low - naming convention established | Keys follow execution routing pattern (`planning` parallels `execution`, `plan_check` matches `workflow.plan_check`). |
| Default path behavior changes for all projects (general-purpose to named agent type) | Medium - could surface latent issues | Execution routing already made this exact change (general-purpose to gsd-executor) without issues. The agent receives the same instructions either way. Checker already uses named type. Test both paths during integration validation. |
| Prompt construction logic adds complexity to plan-phase.md | Medium - context budget for orchestrator | Keep routing to conditional logic per spawn point. No loops, no AI-based selection. Deterministic: check config, apply, done. |
| workflow.plan_check: false silently ignores agents.plan_check | Low - confusing but correct | Document the interaction explicitly in the routing section. Same pattern: workflow.research: false ignores agents.research. |

## Dependencies

### Internal Dependencies (within this change set)

- Config template (Phase 1) must be updated before plan-phase.md changes (Phase 2) -- the routing section references the config schema
- plan-phase.md changes (Phase 2) are independent of new-project.md changes (Phase 3)
- Integration validation (Phase 4) depends on all other phases

### External Dependencies

- Execution pipeline routing must be complete (it IS complete -- `gsd-agents` spec is done)
- Agent definitions must exist for custom agents to work (user's responsibility)
- Claude Code Task tool accepts custom `subagent_type` values (already verified in execution routing pre-flight)

## Success Criteria

- All 5 Task() spawn points in plan-phase.md use dynamic agent resolution instead of hardcoded types
- `<return_protocol>` blocks are injected only for custom agents, not for built-in GSD agents
- Return protocol blocks include ONLY the signals the orchestrator actually parses -- no extra signals from agent definitions that the orchestrator never checks
- Planner return_protocol has exactly 3 signals: `PLANNING COMPLETE`, `CHECKPOINT REACHED`, `PLANNING INCONCLUSIVE` (NOT `REVISION COMPLETE` or `GAP CLOSURE PLANS CREATED`)
- A project with NO planning agent keys in config.json behaves identically to today (backwards compatible)
- Default path change (general-purpose to named agent type) is functionally equivalent for researcher and planner
- A project WITH planning agents configured routes to the correct agent for each spawn point
- The consistency rule works: same resolved planner handles both initial planning (step 8) and revision (step 12)
- The resolution chain (user override > config > fallback) works correctly at each priority level for all 3 roles
- config-agents-example.json is valid JSON with all 5 agent keys documented (all strings, no arrays for planning keys)
- new-project.md asks about planning agents in Round 3 (when user chooses "Configure agents")
- workflow.plan_check: false and workflow.research: false interactions documented in routing section
- No increase in orchestrator context budget beyond ~5% for routing logic

## Files Changed

| # | File | Type | Change Summary |
|---|------|------|----------------|
| 1 | `get-shit-done/templates/config-agents-example.json` | Modify | Add `planning`, `research`, `plan_check` keys with examples and comments |
| 2 | `commands/gsd/plan-phase.md` | Modify | Add `<agent_routing>` section, modify 5 Task() spawn points, inject return_protocol for custom agents |
| 3 | `commands/gsd/new-project.md` | Modify | Extend Round 3 agent pool questions to cover planning, research, plan_check agents |
