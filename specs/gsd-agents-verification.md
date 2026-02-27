# GSD Agent Routing -- Plan & Tasks Verification Report

**Verifier:** Task Completion Verifier
**Date:** 2026-02-26
**Files verified:**
- `specs/gsd-agents-plan.md` (strategic plan, post-debate revision)
- `specs/gsd-agents-tasks.md` (execution tasks, post-debate revision)
- `specs/gsd-agents-review.md` (advocate's review)
- `specs/gsd-agents-debate-r1.md` (planner's response to review)

**Cross-referenced against:** 7 GSD source files (commands, templates, workflows)

---

## CHECKLIST RESULTS

### [ PASS ] All files containing hardcoded "general-purpose" are covered by tasks

**Evidence:**

Grepped all source files. Confirmed hardcoded reference inventory:

**Commands/templates (6 total):**
- `commands/gsd/execute-phase.md` lines 77, 78, 79 -- 3 references -- covered by Task 4.1 Change C
- `commands/gsd/execute-plan.md` line 53 -- 1 reference -- covered by Task 5.1 Change B
- `get-shit-done/templates/subagent-task-prompt.md` line 87 -- 1 reference -- covered by Task 2.1
- `get-shit-done/templates/continuation-prompt.md` line 231 -- 1 reference -- covered by Task 2.2

**Workflows (4 total):**
- `get-shit-done/workflows/execute-phase.md` lines 191 and 230 -- 2 references -- covered by Task 4.2 Change B
- `get-shit-done/workflows/execute-plan.md` lines 209 and 361 -- 2 references -- covered by Task 5.2

**Verified zero references in:** `commands/gsd/plan-phase.md` (confirmed no hardcoded references -- consistent with scope exclusion of planning delegation).

All 10 hardcoded references across 6 files have explicit corresponding tasks.

---

### [ PASS ] Config.json schema is defined and backwards compatible (empty = current behavior)

**Evidence:**

Task 1.1 specifies:
- Add `"agents": {}` as a new top-level key to `get-shit-done/templates/config.json`
- Flat schema: `execution` (array of strings) and `verification` (single string)
- Backwards compatibility invariant explicitly stated: "When agents key is missing or empty, all routing falls back to `general-purpose`. This MUST be the invariant."
- A sibling `config-agents-example.json` file documents the full populated schema
- Current `config.json` has keys: `mode`, `depth`, `parallelization`, `gates`, `safety` -- Task 1.1 preserves all existing keys

The schema is intentionally flat (two shapes: array and string, both agent names). The original over-engineered schema (4 shapes including nested objects and shell commands) was conceded and removed in the debate response. The simplified schema is directly implementable with defensive access (`config.agents || {}`), matching the plan's Risk Assessment.

---

### [ PASS ] Resolution chain (prompt > frontmatter > config > fallback) is implemented in routing tasks

**Evidence:**

The 4-level resolution chain appears in:
- **Plan methodology:** "Resolution Chain" section with explicit Priority 1-4 labeling
- **Task 4.1 Change B:** `<agent_routing>` section in `execute-phase.md` command with all 4 priorities
- **Task 4.2 Change A:** `resolve_agents` workflow step with identical 4-level logic
- **Task 5.1 Change A:** `<agent_routing>` section in `execute-plan.md` command
- **Task 2.1 and 2.2:** Resolution chain note added to template Usage sections

The deterministic selection logic (post-debate concession on CONCERN 1) is clear: one agent = use it; multiple agents = use frontmatter/user override or first in list. No domain matching. No fuzzy heuristics.

---

### [ PASS ] execute-phase changes include: agent routing, verification wave, post-phase hooks

**Partial -- "post-phase hooks" correctly excluded from scope.**

**Evidence:**

Task 4.1 specifies 4 changes to `commands/gsd/execute-phase.md`:
- **Change A:** Add `@.planning/config.json` to context section
- **Change B:** Add `<agent_routing>` section with resolution chain
- **Change C:** Replace 3 hardcoded Task calls with `resolve_agent()` pattern
- **Change D:** Add `<verification_wave>` section (conditional on `agents.verification` config)

Post-phase external tool hooks were explicitly removed from scope in the debate response (CONCERN 4 and CONCERN 5 concessions). The plan's Out of Scope section states: "Post-phase external tool hooks (shell commands are not agents -- belongs in a separate `hooks` feature)." This is a deliberate and correct design decision, not a gap.

Verdict for this item: PASS with clarification -- agent routing and verification wave are covered; post-phase hooks are intentionally out of scope.

---

### [ PASS ] execute-plan changes mirror execute-phase routing appropriately

**Evidence:**

Task 5.1 adds routing to `commands/gsd/execute-plan.md` with the same resolution chain as execute-phase, simplified for single-plan context:
- `<agent_routing>` section with all 4 priority levels
- Step 4 replaced: `Task(prompt=filled_template, subagent_type=resolved_agent)`
- Explicit note that `config.json` is already in the context section (line 31) -- no redundant change needed

Task 5.2 updates `get-shit-done/workflows/execute-plan.md`:
- Both hardcoded references replaced with `resolved_agent`
- Surrounding context provided for both replacement locations (debate concession on CONCERN 3)

Execute-plan does NOT get a verification wave -- this is appropriate because verification runs at the phase level (after each wave), not at the single-plan level. The asymmetry is correct.

---

### [ PASS ] Templates (subagent-task-prompt.md, continuation-prompt.md) remove hardcoded type

**Evidence:**

- **Task 2.1:** Targets `subagent-task-prompt.md` line 87. Replaces `subagent_type="general-purpose"` with `subagent_type=resolved_agent`. Adds resolution chain note. Completion criteria: no `"general-purpose"` in file.
- **Task 2.2:** Targets `continuation-prompt.md` line 231. Same pattern. Same criteria.

Both tasks explicitly instruct: "Do NOT change the Template section (the actual prompt content). Only change the Usage/documentation section." This is the correct minimal change.

---

### [ PASS ] new-project has agent setup step (skippable)

**Evidence:**

Task 3.1 adds `<step name="agents">` to `commands/gsd/new-project.md` between `parallelization` and `config` steps. The step:
- Offers "Skip (use defaults)" as the first/recommended option
- Offers "Configure agents" as the alternative
- Skip path: sets `agents: {}` and proceeds
- Configure path: collects execution agents (comma-separated) and verification agent via freeform input

The current `new-project.md` step sequence is: `setup > brownfield_offer > question > project > mode > depth > parallelization > config > commit > done`. The agents step inserts correctly between `parallelization` and `config`.

The task also requires updating the `config` step (to write agents to config.json), `done` step (to show agents config if non-empty), and `success_criteria` section.

---

### [ PASS ] No agent config references in CLAUDE.md

**Evidence:**

Grepped `~/.claude/CLAUDE.md` for "agent". The only matches are:
- Line 30: "ALWAYS delegate tasks file updates to agents" (refers to AI agents generically, not GSD agent config)
- Line 31: "ALWAYS delegate verification to agents" (same -- generic delegation instruction)

Neither reference is about GSD agent configuration. No GSD-specific agent config instructions exist in CLAUDE.md. This checklist item passes.

---

### [ PASS ] Tasks are ordered correctly (dependencies respected)

**Evidence:**

The tasks file states the dependency chain:

```
Group 0 (pre-flight) MUST pass before ANY other work begins
Group 1 (foundation) must be completed before any other group
Group 2 (templates) should be completed before Groups 4/5
Task 2.3 (verify-prompt.md) must exist before Task 4.1
Group 3 (new-project) depends on Group 1
Groups 4 and 5 are independent of each other
Group 6 (integration validation) depends on all other groups
Recommended order: 0 > 1 > 2 > 3 > 4 > 5 > 6
```

This ordering is internally consistent. Specifically:
- Group 0 gates everything (prevents wasted work if Task tool rejects custom types)
- Config schema (Task 1.1) exists before any routing references it
- `phase-prompt.md` agents field (Task 1.2) exists before commands reference it
- `verify-prompt.md` (Task 2.3) exists before Task 4.1 references it in execution_context
- Template cleanup (Tasks 2.1-2.2) before command rewrites (Groups 4-5) for consistency
- Integration test (Group 6) is the final gate

The tasks file groups tasks into "In Progress Tasks" (Group 0 and Group 1) and "Future Tasks" (Groups 2-6), which is a GSD structural artifact -- the dependency ordering is preserved.

---

### [ PASS ] No over-engineering (minimum changes for maximum effect)

**Evidence:**

The plan achieves the routing goal with:
- 2 new files (verify-prompt.md + config-agents-example.json)
- 8 modified files (all pre-existing GSD files)
- Routing logic is ~5 lines of pseudocode per command (explicitly called out in Risk Assessment)
- No new runtime infrastructure, no agent health checking, no marketplace
- `resolve_agent()` is pseudocode in command text, not a real function to implement
- Planning delegation (the largest scope expansion) was removed entirely after review
- Post-phase hooks removed (category error -- shell commands are not agents)
- Domain matching removed (fuzzy heuristics replaced with deterministic selection)

The debate process reduced the plan from 11 files + planning delegation to 10 files + 1 new file, with significantly reduced complexity. This represents a lean implementation.

---

### [ PASS ] Advocate's concerns (CONCERN 1-7) are all addressed in the revised plan

**Evidence per concern:**

- **CONCERN 1 (domain matching):** CONCEDED. Plan rewrote Agent Selection Logic to be deterministic. Domain matching removed from Tasks 4.1, 4.2, 5.1.
- **CONCERN 2 (Task 5.1 false claim):** CONCEDED. Task 5.1 updated with explicit: "config.json is already in the context section (line 31). No change needed to context."
- **CONCERN 3 (fragile line references):** CONCEDED. Task 5.2 updated with surrounding context for both replacement locations.
- **CONCERN 4 (config schema complexity):** CONCEDED. Schema flattened to 2 shapes. Post-phase hooks removed. Nested planning object removed (planning delegation out of scope).
- **CONCERN 5 (planning delegation scope creep):** CONCEDED. Group 6 (planning delegation) removed entirely from plan and tasks. Replaced by Group 6 (integration validation).
- **CONCERN 6 (verify-prompt missing build commands):** CONCEDED. Task 2.3 updated to add `@.planning/PROJECT.md` to verify-prompt context.
- **CONCERN 7 (unverified Task tool assumption):** CONCEDED. Pre-flight verification added as Group 0. Risk Assessment updated with Critical-impact row. External Dependencies updated.

All 7 concerns addressed with concrete plan and task changes.

---

### [ PASS ] Advocate's MISSING items (1-5) are all addressed in the revised plan

**Evidence per missing item:**

- **MISSING 1 (example config file):** Task 1.1 completion criteria and tests updated to include `config-agents-example.json`. File must exist, be valid JSON, contain `execution` and `verification` examples.
- **MISSING 2 (no rollback plan):** Plan Overview section adds: "All changes should be implemented on a feature branch. If agent routing causes issues, the branch can be reverted as a unit." Tasks file header adds matching note.
- **MISSING 3 (no integration test):** Group 6 (replacing removed planning delegation) contains Task 6.1: 5-scenario end-to-end smoke test covering no-agents, single agent, frontmatter override, verification wave, and missing-key graceful handling.
- **MISSING 4 (continuation routing gap):** Task 4.2 Change A updated with "IMPORTANT: The resolved agent MUST be used for both initial spawn AND any continuation spawns for the same plan." Change B explicitly states continuation must use same agent as initial spawn, not re-resolve.
- **MISSING 5 (monorepo behavior):** Plan Methodology adds "Config Resolution Note" subsection explaining CWD-relative resolution. Tasks Notes section adds explicit statement.

All 5 missing items addressed.

---

### [ PASS ] Pre-flight verification task exists (confirm Task tool accepts custom subagent_type)

**Evidence:**

Group 0 (Pre-flight Verification) is the first group in the tasks file, listed under "In Progress Tasks" alongside Group 1. Task 0.1 specifies:
- Create a minimal test agent definition
- Spawn a Task with `subagent_type="test-agent-name"` and trivial prompt
- Observe whether Task tool accepts the value or errors
- **If FAIL: Stop. The entire plan needs redesign.**
- Completion criteria: Task tool successfully spawns agent with custom subagent_type

The dependency section states: "Group 0 (pre-flight) MUST pass before ANY other work begins." This is a hard gate, not a suggestion.

---

### [ PASS ] Integration smoke test task exists

**Evidence:**

Task 6.1 (Group 6) defines 5 explicit test scenarios:
1. No agents configured -- verify fallback to `"general-purpose"`
2. Single execution agent -- verify config-sourced routing
3. Plan frontmatter override -- verify priority 2 over priority 3
4. Verification wave -- verify agent spawned when configured, skipped when not
5. Missing agents key -- verify no errors and graceful fallback

Note: The smoke test uses `"general-purpose"` as the test agent name throughout, which avoids dependency on custom agent definitions. This is explicitly called out in the task notes. The routing logic is what's being tested, not custom agents.

---

### [ PASS ] Plan and tasks are congruent (no gaps between what plan says and what tasks do)

**Evidence:**

Cross-reference of plan's Files Changed table against tasks:

| Plan File | Plan Change Summary | Corresponding Task |
|-----------|--------------------|--------------------|
| `config.json` | Add agents section | Task 1.1 |
| `phase-prompt.md` | Add agents frontmatter | Task 1.2 |
| `subagent-task-prompt.md` | Remove hardcoded type | Task 2.1 |
| `continuation-prompt.md` | Remove hardcoded type | Task 2.2 |
| `verify-prompt.md` (NEW) | Create verification template | Task 2.3 |
| `new-project.md` | Add optional agents step | Task 3.1 |
| `commands/execute-phase.md` | Add routing + verification wave | Task 4.1 |
| `workflows/execute-phase.md` | Add routing steps | Task 4.2 |
| `commands/execute-plan.md` | Add routing to spawn step | Task 5.1 |
| `workflows/execute-plan.md` | Update spawn examples | Task 5.2 |

10 files in plan table, 10 tasks in tasks file. One-to-one correspondence. No files in the plan without a task, no tasks without a corresponding plan entry.

Plan milestones also map to task groups:
- Milestone 0 (Pre-flight verified) = Group 0
- Milestone 1 (Config schema defined) = Group 1
- Milestone 2 (Templates cleaned up) = Group 2
- Milestone 3 (new-project agents step) = Group 3
- Milestone 4 (execute-phase/plan route dynamically) = Groups 4 + 5
- Milestone 5 (Integration validated) = Group 6

Success Criteria in the plan are all testable and map to specific task completion criteria.

---

## GAPS AND RISKS

### Gap 1: Task 1.1 has an ambiguity about config-agents-example.json location

**Severity: Low**

Task 1.1 mentions two options for documenting the schema: (a) create `config-agents-example.json`, or (b) "document the schema inline in the config template using a pattern consistent with how GSD already documents config." The task concludes with "Alternatively, document the schema..." -- this is an unresolved fork. The completion criteria and tests reference `config-agents-example.json` specifically, so the implementer should create that file. But the "alternatively" language could cause hesitation.

**Recommendation:** The implementer should resolve this as: create `config-agents-example.json`. The completion criteria already require it. The "alternatively" clause is an artifact of pre-revision drafting.

---

### Gap 2: Task 3.1 is underdetermined on agents step placement in the `config` step write

**Severity: Low**

The `config` step in `new-project.md` currently reads: "Create `.planning/config.json` with chosen mode, depth, and parallelization using `templates/config.json` structure." Task 3.1 says to update this step to include agents. But it does not specify exactly what the write looks like when agents is empty `{}` vs populated. An implementer could reasonably write either:

```json
{ "mode": "...", "depth": "...", ..., "agents": {} }
```

or omit the key entirely when skipped. The backwards compatibility invariant requires the key to be present (so `config.agents || {}` defensive access works), but an absent key also falls back correctly. This is a minor implementation detail with no wrong answer.

**Recommendation:** The implementer should always write `"agents": {}` (empty object) when user skips, not omit the key. This is consistent with Task 1.1's schema definition.

---

### Gap 3: No task covers `get-shit-done/workflows/execute-plan.md` being a separate file from `commands/gsd/execute-plan.md`

**Severity: Not a gap -- this is already covered.**

Verification confirmed: Task 5.1 covers `commands/gsd/execute-plan.md`, Task 5.2 covers `get-shit-done/workflows/execute-plan.md`. Both are in the plan. The concern is noted only to confirm both were checked independently.

---

### Risk 1: The Task tool pre-flight (Group 0) has no fallback plan articulated

**Severity: Medium**

Task 0.1 correctly gates all implementation on the pre-flight result. However, the "If FAIL" path says only: "Stop. The plan needs a fundamentally different approach. Document the failure and reassess." There is no sketch of what the alternative approach would be.

This is acceptable for a pre-flight gate (the alternative depends on what exactly fails), but it means if the Task tool rejects custom `subagent_type` values, the team has no fallback ready. The entire plan would need to be redesigned from scratch.

**Recommendation:** This risk is acknowledged in the plan's Risk Assessment table ("Task tool rejects custom subagent_type values" is rated Critical). The pre-flight gate is the correct mitigation. Accept this as a known risk.

---

### Risk 2: Verification wave cost is not bounded

**Severity: Low**

The verification wave spawns one agent per wave. For a phase with 10 waves, this is 10 additional Task calls. The plan correctly notes verification is optional (only runs when `agents.verification` is configured) and this risk is managed by user opt-in. No change needed.

---

## OVERALL ASSESSMENT

**VERDICT: READY FOR EXECUTION**

**Confidence: HIGH**

The plan and tasks are internally consistent, complete, and well-ordered. All 15 checklist items pass. The two gaps identified are minor implementation details (one ambiguous "alternatively" clause, one unspecified empty-vs-omit decision) that any competent implementer will resolve correctly.

The debate process (review + response) meaningfully improved the plan by:
1. Removing the largest scope expansion (planning delegation)
2. Eliminating fragile fuzzy matching logic
3. Flattening the config schema
4. Adding the pre-flight gate that protects against fundamental assumption failure
5. Adding the integration smoke test that validates end-to-end routing

The resulting plan is lean, deterministic, and backwards-compatible. It modifies 9 existing files and creates 2 new files. The implementation complexity is appropriate for the scope of change.

**Execution prerequisites:**
- Feature branch created before any file changes
- Group 0 (Task 0.1) must pass before any other work begins
- All task completion criteria and tests are specific enough to verify without ambiguity
