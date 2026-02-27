# GSD Agent Routing — Session Report

**Date:** 2026-02-27
**Branch:** main (implementation work should start on a feature branch)
**Status:** Planning complete, ready for execution

---

## 1. Executive Summary

This session analyzed the GSD meta-prompting system to understand how it hardcodes `subagent_type="general-purpose"` in 10 locations across 6 files, preventing users from routing tasks to specialized agents. The session produced a full strategic plan, execution task breakdown, devils-advocate review, planner debate response, and a verifier's 15-point checklist — all of which are stored in `specs/`. The final plan passed verification with all 15 checks at PASS. One outstanding issue was identified post-verification: the agent selection logic in the plan still says "use first in list" for multiple agents, but the agreed behavior is that the orchestrator (Claude) picks the right agent per task based on task content and agent descriptions. This must be corrected before execution begins.

---

## 2. The Problem

GSD's orchestrator commands hardcode `subagent_type="general-purpose"` in every Task tool call. When users instruct GSD to use a specific agent — for example, "use typescript-pro for this phase" — the 600+ lines of structured command instructions in `execute-phase.md` and `execute-plan.md` override the user's prompt. The user's intent is technically registered but the spawned agent is always `"general-purpose"`.

### Hardcoded Reference Inventory (10 total)

**Commands and templates (6 references):**

| File | Lines | Count |
|------|-------|-------|
| `~/.claude/commands/gsd/execute-phase.md` | 77, 78, 79 | 3 |
| `~/.claude/commands/gsd/execute-plan.md` | 53 | 1 |
| `~/.claude/get-shit-done/templates/subagent-task-prompt.md` | 87 | 1 |
| `~/.claude/get-shit-done/templates/continuation-prompt.md` | 231 | 1 |

**Workflows (4 references):**

| File | Lines | Count |
|------|-------|-------|
| `~/.claude/get-shit-done/workflows/execute-phase.md` | 191, 230 | 2 |
| `~/.claude/get-shit-done/workflows/execute-plan.md` | 209, 361 | 2 |

All 10 references are confirmed by the verifier's grep of every file.

---

## 3. Key Finding: @-Reference Behavior in Task Tool Prompts

During analysis, a secondary question arose: do `@path/to/file` references in Task tool prompt strings actually expand, or are they passed as literal text?

**Finding (empirically tested):** `@` references do NOT expand in Task tool prompts. The prompt string is passed as-is to the subagent. However, Claude (as the subagent) recognizes the `@path/to/file` pattern and reads the file via the Read tool on its own.

**Implication:** GSD currently works by accident, not by design. The subagent-task-prompt.md contains lines like `@~/.claude/get-shit-done/workflows/execute-plan.md`, and the spawned agent reads that file on its own initiative. This behavior is:
- Model-dependent (a less capable model might not recognize or act on the pattern)
- Fragile (the agent might skip some references if context gets tight)
- Costs extra tool turns compared to true reference expansion

This finding is informational for this session. It does not block the agent routing work but is worth noting for a future session on GSD prompt architecture.

---

## 4. The Real Blocker: Hardcoded Agent Types

The actual problem preventing agent customization is structural. The orchestrator commands (`execute-phase.md`, `execute-plan.md`) contain explicit `subagent_type="general-purpose"` strings inside their `<wave_execution>` and process sections. No amount of user instruction can override what is baked into the command's execution logic.

Current state in `~/.claude/commands/gsd/execute-phase.md` (lines 76-82):

```
Task(prompt=filled_template_for_plan_01, subagent_type="general-purpose")
Task(prompt=filled_template_for_plan_02, subagent_type="general-purpose")
Task(prompt=filled_template_for_plan_03, subagent_type="general-purpose")
```

Current state in `~/.claude/commands/gsd/execute-plan.md` (line 53):

```
Spawn: `Task(prompt=filled_template, subagent_type="general-purpose")`
```

Current state in `~/.claude/get-shit-done/templates/subagent-task-prompt.md` (lines 85-88):

```python
Task(
    prompt=filled_template,
    subagent_type="general-purpose"
)
```

---

## 5. The Solution (Agreed and Verified)

### Core Change

Remove all hardcoded `"general-purpose"` strings from commands and templates. Replace with a dynamic resolution chain that the orchestrator follows for every Task call.

### Resolution Chain (4 levels)

```
Priority 1: User prompt override (per-invocation)
    "use typescript-pro for this phase"
    -> Orchestrator honors explicit user instruction

Priority 2: Plan frontmatter agents field (per-plan)
    ---
    agents: ["org:typescript-pro"]
    ---
    -> Set by planner agent or user, specific to one plan

Priority 3: .planning/config.json agents section (per-project)
    "agents": { "execution": ["org:typescript-pro"] }
    -> If one agent: use it for all plans
    -> If multiple: orchestrator picks based on task content and agent descriptions

Priority 4: Fallback
    -> "general-purpose" (current behavior, always available)
```

### Config Schema (Final Agreed Version)

```json
{
  "agents": {
    "execution": ["org:agent-a", "org:agent-b"],
    "verification": "org:verifier-agent"
  }
}
```

Two keys. Both are agent names. `execution` is an array (supports multiple agents). `verification` is a single string. No nested objects. No shell commands. Empty `{}` or missing key = fallback to `"general-purpose"` (full backwards compatibility).

### Verification Wave (Optional)

When `agents.verification` is configured, after each execution wave completes, the orchestrator spawns a verification agent using the new `verify-prompt.md` template. The agent checks compilation, linting, test suite, and plan success criteria. If PASS: proceed to next wave. If FAIL: report to user and ask how to proceed. Skipped entirely when not configured.

### Monorepo Behavior

Config resolution is always CWD-relative. GSD already scopes to CWD, so `.planning/config.json` always resolves to the correct sub-project config automatically.

---

## 6. What Was Removed During Review

The devils-advocate-developer review identified and the strategic planner conceded the following changes to the original plan:

### Removed

**Planning delegation (original Group 6):** The plan originally included a group that added planner/reviewer subagent spawning to `plan-phase.md`. This was removed because `plan-phase.md` has zero hardcoded `"general-purpose"` references — it does not spawn subagents at all. Adding new subagent spawning is a new feature, not a routing fix. It deserves its own plan, templates, and testing.

**Fuzzy domain matching:** The original plan included logic to match a plan's `domain` frontmatter field against agent names (e.g., `domain: "frontend"` matches `"org:frontend-dev"`). This was removed because it is fragile (agent `org:react-specialist` does not substring-match domain `frontend`), undefined ("clear match" had no definition), and unnecessary given the `agents:` frontmatter field already handles per-plan overrides explicitly.

**Complex config schema:** The original schema had 4 different shapes in one object: an array (`execution`), a nested object (`planning`), a string (`verification`), and a mixed nested object (`post_phase`) that contained shell commands. Shell commands in an `agents` config key is a category error. Flattened to 2 shapes.

### Added

**Group 0 — Pre-flight verification:** A 30-second test that spawns a Task with a custom `subagent_type` value before any file changes are made. If the Task tool rejects custom values, the entire plan needs redesign. This is a hard gate.

**Group 6 — Integration smoke test:** 5-scenario end-to-end validation covering: no agents configured, single execution agent, plan frontmatter override, verification wave on/off, missing agents key graceful handling.

---

## 7. Outstanding Issue: Agent Selection Logic (Must Fix Before Execution)

The plan and tasks currently say:

> "If `agents.execution` has multiple agents: the plan's `agents:` frontmatter field or user override MUST specify which one. If neither is present, use the first agent in the execution list."

This was the conceded replacement for domain matching — deterministic, not fuzzy. However, during the session's final review, a better behavior was identified that was not reflected in the plan:

**The agreed behavior:** When the orchestrator has a pool of multiple execution agents, it reads the task content and the available agent descriptions, and picks the most appropriate agent for each task. Claude (as the orchestrator) is capable of this without any heuristic code. This is not domain matching — it is the orchestrator doing what orchestrators do: understanding context and making routing decisions.

**What the plan currently says:** "Use first in list" as fallback when frontmatter/user override are absent.

**What it should say:**

```
Use agents from the user's prompt or config.json agents.execution pool.
Pick the appropriate agent for each task based on task content
and agent descriptions. If no agents available, use "general-purpose".
```

This change must be made to the plan and tasks files before execution begins. It affects:
- `specs/gsd-agents-plan.md` — "Agent Selection Logic" section (lines 67-75)
- `specs/gsd-agents-tasks.md` — Task 4.1 Change B `<agent_routing>` section
- `specs/gsd-agents-tasks.md` — Task 4.2 Change A `resolve_agents` step
- `specs/gsd-agents-tasks.md` — Task 5.1 Change A `<agent_routing>` section

---

## 8. Files to Modify (10 files)

All paths are relative to `~/.claude/` (GSD installation root).

| # | File | Change |
|---|------|--------|
| 1 | `get-shit-done/templates/config.json` | Add `"agents": {}` top-level key. Create sibling `config-agents-example.json` with full populated example. |
| 2 | `get-shit-done/templates/phase-prompt.md` | Add optional `agents: []` frontmatter field in template, fields table, and at least one example. |
| 3 | `get-shit-done/templates/subagent-task-prompt.md` | Replace `subagent_type="general-purpose"` with `subagent_type=resolved_agent` in Usage section. Add resolution chain note. Do NOT change Template section. |
| 4 | `get-shit-done/templates/continuation-prompt.md` | Same as #3. Target: line 231. |
| 5 | `get-shit-done/templates/verify-prompt.md` | NEW FILE. Template for verification agents. Contains `<objective>`, `<context>` (including `@.planning/PROJECT.md`), `<verification_checks>`, `<output_format>` sections. |
| 6 | `commands/gsd/new-project.md` | Add `<step name="agents">` between `parallelization` and `config` steps. Skippable. Update config step to write agents. Update done step to show agents if non-empty. |
| 7 | `commands/gsd/execute-phase.md` | Add `@.planning/config.json` to context. Add `<agent_routing>` section. Replace 3 hardcoded Task calls with `resolve_agent()` pattern. Add `<verification_wave>` section. |
| 8 | `get-shit-done/workflows/execute-phase.md` | Add `<step name="resolve_agents">` between `group_by_wave` and `execute_waves`. Update both hardcoded references (lines 191, 230) with resolved agent variables. Add `<step name="verify_wave">`. |
| 9 | `commands/gsd/execute-plan.md` | Add `<agent_routing>` section. Replace line 53 hardcoded Task call with `resolved_agent`. Note: config.json already in context at line 31 — no context change needed. |
| 10 | `get-shit-done/workflows/execute-plan.md` | Replace both hardcoded references (lines 209, 361) with `resolved_agent`. Add brief routing note at each location. |

---

## 9. User's Workflow Examples

The user shared two example prompts demonstrating how they currently work with specialized agents manually (before this routing feature exists).

**Execution prompt (with specialized agents):**

```
/gsd:execute-phase 03

Use the following agents for this phase:
- typescript-pro: for all TypeScript implementation tasks
- frontend-dev: for all React/UI tasks
- test-specialist: for test writing tasks
```

**Planning prompt (with specialized agents):**

```
/gsd:plan-phase 04

Use these agents:
- planner-task-decomposer: to decompose phase objectives into plans
- devils-advocate-developer: to review the plans
- task-completion-verifier: to verify the plans are complete
```

These examples illustrate why routing matters. The user is specifying agents that live in their `.claude/agents/` directory and have specialized capabilities. Today, GSD ignores these instructions because the command overwrites with `"general-purpose"`. After this work, Priority 1 (user prompt override) handles them correctly.

---

## 10. Artifacts Produced

All artifacts are in `/mnt/3b92ea25-2e45-41c8-97d3-58aa8141755e/Videos/Projects/NIIT/get-shit-done/specs/`.

| File | Description | Status |
|------|-------------|--------|
| `specs/gsd-agents-plan.md` | Strategic plan (post-debate revision) | Needs routing logic fix (Section 7) |
| `specs/gsd-agents-tasks.md` | 6 groups, 11 tasks (post-debate revision) | Needs routing logic fix (Section 7) |
| `specs/gsd-agents-review.md` | Devils-advocate review — 7 concerns, 5 missing items | Complete |
| `specs/gsd-agents-debate-r1.md` | Planner response — 12/12 concessions | Complete |
| `specs/gsd-agents-verification.md` | Verifier's 15-point checklist — 15/15 PASS | Complete |

---

## 11. Next Steps for Future Session

Execute in this order:

**Step 0 — Fix routing logic in specs (before running GSD on the plan)**

Update `specs/gsd-agents-plan.md` and `specs/gsd-agents-tasks.md` to replace "use first in list" with "orchestrator picks based on task content and agent descriptions." Specifically:
- `gsd-agents-plan.md` lines 67-75: rewrite Agent Selection Logic section
- `gsd-agents-tasks.md` Task 4.1 Change B: rewrite `<agent_routing>` section instructions
- `gsd-agents-tasks.md` Task 4.2 Change A: rewrite `resolve_agents` step logic
- `gsd-agents-tasks.md` Task 5.1 Change A: rewrite `<agent_routing>` section instructions

**Step 1 — Create feature branch**

```bash
git checkout -b feature/agent-routing
```

**Step 2 — Execute Group 0 (pre-flight)**

Verify the Task tool accepts custom `subagent_type` values by spawning a test agent with a non-standard value. If PASS: proceed. If FAIL: the plan needs fundamental redesign — do not proceed.

**Step 3 — Execute Groups 1 through 5 in order**

```
Group 1: Foundation (config.json + phase-prompt.md agents field)
Group 2: Template cleanup (subagent-task-prompt, continuation-prompt, verify-prompt creation)
Group 3: new-project agent setup step
Group 4: execute-phase routing + verification wave
Group 5: execute-plan routing
```

**Step 4 — Execute Group 6 (integration smoke test)**

Run all 5 smoke test scenarios. Document results.

**Step 5 — Consider planning delegation as a separate plan**

The planning delegation feature (spawning planner/reviewer/verifier agents during `plan-phase`) was explicitly removed from this plan as scope creep. It is a legitimate feature but needs its own plan. Key requirements when that session begins:
- `plan-phase.md` currently has zero hardcoded `"general-purpose"` references
- Adding planning delegation means adding entirely new Task calls that do not exist today
- A new prompt template for the planner agent is required (every GSD agent spawn uses a template)
- The reviewer iteration loop adds control flow complexity that does not currently exist in plan-phase

---

## 12. Key Design Decisions and Rationale

| Decision | Rationale |
|----------|-----------|
| Flat config schema (2 shapes only) | 4 shapes require 4 different access patterns in every orchestrator that reads config. Two shapes (array + string) keep the routing logic to ~5 lines of pseudocode. |
| No domain matching | "If clear match" is undefined. `org:react-specialist` does not substring-match `frontend`. The `agents:` frontmatter field is the explicit per-plan mechanism. |
| Orchestrator picks agent from pool | Claude reads task content + agent descriptions and selects. No heuristic code needed. This is what orchestrators do. |
| Shell commands excluded from agents config | Shell commands are not agents. Mixing them in the `agents` key is a category error. If needed later, they belong in a separate `hooks` config key. |
| Planning delegation excluded | `plan-phase.md` has zero hardcoded references. Adding new subagent spawning is a new feature, not a routing change. Ships separately. |
| Pre-flight gate (Group 0) | The fundamental assumption (Task tool accepts custom `subagent_type` values) has never been formally verified. If it fails, 10+ files of work is wasted. 30 seconds of testing is cheap insurance. |
| Feature branch | All changes are to markdown prompt files. `git revert` on a feature branch is a one-step rollback if routing causes issues in production. |
| Continuation uses same resolved agent | When a checkpoint plan pauses and resumes, the continuation spawn must use the same resolved agent as the initial spawn. Re-resolving could pick a different agent if config changed. |
| verify-prompt includes PROJECT.md | Verification agent needs to know build/test/lint commands. Adding `@.planning/PROJECT.md` to context lets it infer commands from the project description without adding a new `commands` config key. |

---

## 13. Risk Register

| Risk | Impact | Mitigation | Status |
|------|--------|-----------|--------|
| Task tool rejects custom `subagent_type` | Critical — entire plan fails | Group 0 pre-flight gate before any file changes | Unverified |
| Config.json parsing fails on missing agents key | High — breaks all execution | Defensive access `config.agents \|\| {}` in every routing section | Design |
| Agent name typo in config | Medium — plan execution fails | Orchestrator does not validate. Task tool handles unknown types. Document in config template. | Design |
| Routing logic exceeds context budget | Medium — context budget exceeded | ~5 lines pseudocode per command. No AI routing code. Deterministic selection only. | Design |
| Backwards compatibility regression | High — breaks existing projects | Every path has explicit fallback to `"general-purpose"`. No agents key = identical behavior. | Design |
| Verification wave adds unwanted overhead | Low — slows execution | Verification only runs when `agents.verification` is configured. | Design |

---

*Report generated: 2026-02-27*
*Session type: Analysis and planning (no code executed)*
*Next session: Fix routing logic in specs, then begin execution on feature branch*
