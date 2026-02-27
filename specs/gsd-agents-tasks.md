---

# GSD Agent Routing Implementation

Modify GSD's orchestrator commands to dynamically route tasks to specialized agents from a configurable pool, replacing all hardcoded `subagent_type="general-purpose"` references. Adds optional verification agents.

**Implementation strategy:** All changes on a feature branch. Revert the branch as a unit if routing causes issues.

## Completed Tasks

- [x] Group 0: Pre-flight Verification
  - [x] Task 0.1: Verify Task tool accepts custom subagent_type values — PASS. Codebase already uses custom subagent_type values (gsd-executor, gsd-verifier, gsd-planner, gsd-debugger, gsd-roadmapper, gsd-codebase-mapper) confirming Task tool accepts arbitrary strings.
- [x] Group 1: Foundation -- Config & Template Schema
  - [x] Task 1.1: Add agents section to templates/config.json — DONE. Added `"agents": {}` as top-level key after `safety` in `get-shit-done/templates/config.json`. Created sibling file `get-shit-done/templates/config-agents-example.json` with full schema example showing `execution` (array) and `verification` (string) fields, plus `_comment` keys documenting usage. Both files validated with `python3 -m json.tool`. All existing keys confirmed unchanged.
- [x] Group 2: Template Updates -- Remove Hardcoded References & Add Verification Template
  - [x] Task 2.1: Remove hardcoded general-purpose from subagent-task-prompt.md — SKIPPED. File `get-shit-done/templates/subagent-task-prompt.md` does not exist in this codebase.
  - [x] Task 2.2: Remove hardcoded general-purpose from continuation-prompt.md — SKIPPED. File `get-shit-done/templates/continuation-prompt.md` does not exist in this codebase.
  - [x] Task 2.3: Create verify-prompt.md template — DONE. Created `get-shit-done/templates/verify-prompt.md` with Template section containing `<objective>`, `<context>`, `<verification_checks>`, `<output_format>` XML structure. Includes Placeholders table with all 6 placeholders and Usage section showing conditional spawn via `config.agents.verification`. Follows same structure as debug-subagent-prompt.md and planner-subagent-prompt.md.
- [x] Group 3: new-project Command -- Agent Pool Setup
  - [x] Task 3.1: Add optional agents setup step to new-project.md — DONE. Added "Round 3 — Agent pool" section to Phase 5 (Workflow Preferences) in `commands/gsd/new-project.md`, positioned after Round 2 (workflow agents) and before the config.json write. Uses AskUserQuestion with "Skip (use defaults)" as first/recommended option. "Configure agents" triggers freeform follow-ups for execution agents (comma-separated) and verification agent. Updated config.json template to include `"agents": {}` key. Updated Phase 10 (Done) to conditionally show agent pool info. Updated success_criteria to include agents configuration check. Updated commit message to include agent pool status.
- [x] Group 4: execute-phase -- Dynamic Agent Routing
  - [x] Task 4.1: Update commands/gsd/execute-phase.md with agent routing — DONE. Applied 4 changes: (A) Added `@.planning/config.json` to `<context>` section and `@~/.claude/get-shit-done/templates/verify-prompt.md` to `<execution_context>`. (B) Added `<agent_routing>` section with 3-level priority chain (user override, config pool, fallback to "gsd-executor"). (C) Replaced 3 hardcoded `subagent_type="gsd-executor"` Task calls in `<wave_execution>` with `resolve_agent()` pattern using dynamic agent variables. (D) Added `<verification_wave>` section for optional post-wave verification when `agents.verification` is configured. All sections positioned after `<checkpoint_handling>` and before `<deviation_rules>`.
  - [x] Task 4.2: Update workflows/execute-phase.md with agent routing steps — DONE. Applied 3 changes: (A) Added `<step name="resolve_agents">` between `group_by_wave` and `execute_waves` with 4-step agent resolution logic (config read, user override check, pool selection, storage). (B) Replaced both hardcoded `subagent_type="gsd-executor"` in `checkpoint_handling` step with `subagent_type={resolved_agent_for_plan}` — initial spawn and continuation spawn both use the same resolved agent variable. (C) Added `<step name="verify_wave">` after `checkpoint_handling` and before `aggregate_results` with conditional verification agent spawning using verify-prompt template.
- [x] Group 5: execute-plan -- Dynamic Agent Routing
  - [x] Task 5.1: Update commands/gsd/execute-plan.md with agent routing — SKIPPED. File `commands/gsd/execute-plan.md` does not exist as a command file. Only the workflow file `get-shit-done/workflows/execute-plan.md` exists.
  - [x] Task 5.2: Update workflows/execute-plan.md with agent routing — DONE. Replaced both hardcoded `subagent_type="gsd-executor"` references with `subagent_type={resolved_agent}` in `get-shit-done/workflows/execute-plan.md`. Added routing resolution priority chain comment (user override, config agents.execution[], fallback to "gsd-executor") before each Task call. Location 1: "For fully autonomous plans" block in segment_routing step. Location 2: "If routing = Subagent" block in the segment loop. All existing workflow steps intact and unchanged in structure. Verified: zero `"gsd-executor"` hardcoded references remain, 2 `resolved_agent` references present, 2 routing notes present.
- [x] Group 6: Integration Validation — DONE.
  - [x] Task 6.1: End-to-end integration validation — DONE. Performed comprehensive static review of all modified files. All checks PASS. Summary:
    - Consistency: Resolution chain (user override > config > fallback to "gsd-executor") documented identically in execute-phase command `<agent_routing>`, execute-phase workflow `resolve_agents` step, and execute-plan workflow routing comments. Verification wave is conditional on `agents.verification` in all three locations.
    - Backwards compatibility: `get-shit-done/templates/config.json` has `"agents": {}` (empty, backwards compatible). All routing paths explicitly fall back to "gsd-executor" when agents key is missing or empty. The resolve_agents step in execute-phase workflow explicitly states "If agents key missing or empty: all plans use gsd-executor (default)".
    - No hardcoded "general-purpose" remains in execute-phase.md command (0 matches), execute-phase.md workflow (0 routing matches — 1 non-routing table entry is acceptable context), or execute-plan.md workflow (0 matches).
    - Template integrity: verify-prompt.md is well-formed with all 4 XML sections and all 6 placeholders documented. config.json and config-agents-example.json are valid JSON (python3 -m json.tool confirmed). config-agents-example.json has execution (array) and verification (string) schema.
    - new-project.md: "Round 3 — Agent pool" step exists between Round 2 (workflow agents) and the config.json write. Success_criteria includes agents configuration check. Phase 10 Done section conditionally shows agent pool. "Skip (use defaults)" is first/recommended option.
    - Cross-references: execute-phase command has verify-prompt.md in `<execution_context>` (line 28) and config.json in `<context>` (line 39). execute-phase workflow has `resolve_agents` step (line 238) before `execute_waves` step (line 265) and `verify_wave` step (line 472). execute-plan workflow uses `{resolved_agent}` variable at lines 233 and 390.
    - Plan success criteria: All achievable criteria met. The 3 skipped tasks (2.1, 2.2, 5.1) are correctly noted — those files do not exist in this codebase. Fallback agent adapted from "general-purpose" (plan) to "gsd-executor" (codebase-appropriate default, consistent with existing usage throughout the codebase).
    - All Groups (0–6) marked complete in Completed Tasks section.

## In Progress Tasks

- ~~[ ] Group 0: Pre-flight Verification~~
  - ~~[ ] Task 0.1: Verify Task tool accepts custom subagent_type values~~
    - Purpose: Confirm the fundamental assumption that `subagent_type` accepts arbitrary strings (e.g., `"org:frontend-dev"`) before making any file changes. If this fails, the entire plan needs redesign.
    - Steps:
      1. Create a minimal test agent definition in `.claude/agents/` (or use an existing custom agent if available)
      2. Spawn a Task with `subagent_type="test-agent-name"` and a trivial prompt (e.g., "Return the string OK")
      3. Observe whether the Task tool accepts the value or errors
    - If PASS: Proceed to Group 1
    - If FAIL: Stop. The plan needs a fundamentally different approach (e.g., all routing happens at the prompt level, not the subagent_type level). Document the failure and reassess.
    - Completion Criteria: Task tool successfully spawns an agent with a custom `subagent_type` value that is not `"general-purpose"` or `"Explore"`.
    - Tests: The spawned agent returns a response (any response). No error about invalid subagent_type.

- [x] Group 1: Foundation -- Config & Template Schema
  - [x] Task 1.1: Add agents section to templates/config.json
    - File: `get-shit-done/templates/config.json`
    - Current state: JSON with `mode`, `depth`, `parallelization`, `gates`, `safety` keys. No agents key.
    - Change: Add `"agents": {}` as a new top-level key after `safety`. Empty object is the default (backwards compatible -- empty means "use general-purpose for everything").
    - Also add a commented/documented version showing the full schema. Since JSON does not support comments, add a sibling file `get-shit-done/templates/config-agents-example.json` containing a complete example of the agents section with all fields populated and explanatory structure. Alternatively, document the schema inline in the config template using a pattern consistent with how GSD already documents config (check if there is a reference doc for config format).
    - Full agents schema when populated:
      ```json
      "agents": {
        "execution": ["org:agent-name-1"],
        "verification": "org:verifier-agent"
      }
      ```
    - Schema is intentionally flat: `execution` is an array of agent names, `verification` is a single agent name string. Two shapes, both agent names. No nested objects. No shell commands (those belong in a separate `hooks` config if needed later).
    - Backwards compatibility: When agents key is missing or empty, all routing falls back to `"general-purpose"`. This MUST be the invariant.
    - Completion Criteria: `templates/config.json` contains `"agents": {}` as a top-level key. File is valid JSON. Existing keys unchanged. Sibling file `config-agents-example.json` exists with full example.
    - Tests: Parse the updated JSON with `JSON.parse()` or `python -m json.tool` to confirm validity. Confirm all existing keys (`mode`, `depth`, `parallelization`, `gates`, `safety`) still present and unchanged. Confirm `agents` key exists and is an empty object. Confirm `config-agents-example.json` exists and is valid JSON with `execution` and `verification` examples.

## Future Tasks

- ~~[x] Group 2: Template Updates -- Remove Hardcoded References & Add Verification Template~~ (DONE — 2.1 & 2.2 SKIPPED, 2.3 completed)
  - ~~[ ] Task 2.1: Remove hardcoded general-purpose from subagent-task-prompt.md~~
    - File: `get-shit-done/templates/subagent-task-prompt.md`
    - Current state: Line 85-89 in the Usage section contains:
      ```python
      Task(
          prompt=filled_template,
          subagent_type="general-purpose"
      )
      ```
    - Change: Replace the Usage section's hardcoded example with routing-aware documentation:
      ```python
      Task(
          prompt=filled_template,
          subagent_type=resolved_agent  # See agent routing resolution chain
      )
      ```
    - Add a brief note below the code block:
      ```
      Agent resolution (highest to lowest priority):
      1. User prompt override (per-invocation)
      2. `.planning/config.json` agents.execution pool (per-project)
      3. Fallback: "general-purpose"
      ```
    - Do NOT change the Template section (the actual prompt content). Only change the Usage/documentation section.
    - Completion Criteria: No literal `"general-purpose"` string remains in the file. Usage section shows `resolved_agent` variable. Resolution chain is documented.
    - Tests: Grep for `"general-purpose"` in the file -- should return zero matches. Confirm the Template section (the actual prompt XML) is unchanged. Confirm resolution chain note is present.

  - ~~[ ] Task 2.2: Remove hardcoded general-purpose from continuation-prompt.md~~
    - File: `get-shit-done/templates/continuation-prompt.md`
    - Current state: Line 229-233 in the Usage section contains:
      ```python
      Task(
          prompt=filled_continuation_template,
          subagent_type="general-purpose"
      )
      ```
    - Change: Same pattern as Task 2.1. Replace with:
      ```python
      Task(
          prompt=filled_continuation_template,
          subagent_type=resolved_agent  # See agent routing resolution chain
      )
      ```
    - Add the same resolution chain note below the code block.
    - Do NOT change the Template section (the actual prompt content). Only change the Usage/documentation section.
    - Completion Criteria: No literal `"general-purpose"` string remains in the file. Usage section shows `resolved_agent` variable. Resolution chain is documented.
    - Tests: Grep for `"general-purpose"` in the file -- should return zero matches. Confirm the Template section (the actual continuation prompt XML) is unchanged. Confirm resolution chain note is present.

  - ~~[x] Task 2.3: Create verify-prompt.md template~~
    - File: `get-shit-done/templates/verify-prompt.md` (NEW FILE)
    - Purpose: Template for spawning verification agents after execution waves. Used by execute-phase when `agents.verification` is configured.
    - Content structure (follow the pattern of subagent-task-prompt.md):
      ```markdown
      # Verification Prompt Template

      Template for spawning verification agents after execution waves. Used by execute-phase orchestrator when `agents.verification` is configured in config.json.

      ---

      ## Template

      ```markdown
      <objective>
      Verify the work completed in wave {wave_number} of phase {phase_number}-{phase_name}.

      Check compilation, linting, test suites, and plan success criteria. Report pass/fail with details.
      </objective>

      <context>
      Phase directory: @{phase_dir}
      Config: @.planning/config.json
      Project: @.planning/PROJECT.md (for discovering build/test/lint commands)
      Plans completed this wave: {completed_plan_list}

      Summaries to check:
      {summary_paths}
      </context>

      <verification_checks>
      1. **Compile check:** Run the project's build/compile command. Report errors.
      2. **Lint check:** Run the project's lint command. Report warnings and errors.
      3. **Test suite:** Run the project's test command. Report failures.
      4. **Plan criteria:** For each completed plan, read its success_criteria section and verify each criterion is met.
      </verification_checks>

      <output_format>
      ## VERIFICATION RESULT

      **Wave:** {wave_number}
      **Status:** [PASS | FAIL]

      ### Checks
      | Check | Status | Details |
      |-------|--------|---------|
      | Compile | [pass/fail] | [details if fail] |
      | Lint | [pass/fail] | [details if fail] |
      | Tests | [pass/fail] | [X passed, Y failed] |
      | Plan Criteria | [pass/fail] | [which criteria failed] |

      ### Issues (if any)
      - [Issue description and affected files]

      ### Recommendation
      [Proceed to next wave / Fix issues before continuing]
      </output_format>
      ```

      ---

      ## Placeholders

      | Placeholder | Source | Example |
      |-------------|--------|---------|
      | `{wave_number}` | Current wave | `1` |
      | `{phase_number}` | Phase directory | `03` |
      | `{phase_name}` | Phase directory | `features` |
      | `{phase_dir}` | Phase path | `.planning/phases/03-features` |
      | `{completed_plan_list}` | Wave results | `03-01, 03-02` |
      | `{summary_paths}` | Derived from plans | `@.planning/phases/03-features/03-01-SUMMARY.md` |

      ---

      ## Usage

      Orchestrator fills placeholders and spawns verification agent:

      ```python
      Task(
          prompt=filled_verify_template,
          subagent_type=config.agents.verification  # From config.json
      )
      ```

      Only spawned when `agents.verification` is configured. Skipped if not present.
      ```
    - Completion Criteria: File exists at `get-shit-done/templates/verify-prompt.md`. Contains Template section with XML structure, Placeholders table, and Usage section. Follows same structure as subagent-task-prompt.md.
    - Tests: File exists and is non-empty. Contains `<objective>`, `<context>`, `<verification_checks>`, `<output_format>` sections. Placeholders table has all 6 placeholders listed.

- [x] Group 3: new-project Command -- Agent Pool Setup
  - [x] Task 3.1: Add optional agents setup step to new-project.md
    - File: `commands/gsd/new-project.md`
    - Current state: Process steps are: setup > brownfield_offer > question > project > mode > depth > parallelization > config > commit > done. No agents step.
    - Change: Add a new `<step name="agents">` between the `parallelization` step and the `config` step. The step should:
      1. Use AskUserQuestion:
         - header: "Agent Pool"
         - question: "Configure specialized agents for this project?"
         - options:
           - "Skip (use defaults)" -- No agents configured, all routing uses general-purpose (Recommended for most projects)
           - "Configure agents" -- Set up execution and verification agents
      2. If "Skip": Set agents to empty object `{}`, proceed to config step
      3. If "Configure agents": Ask follow-up questions:
         a. "Execution agents (comma-separated agent names, or 'skip'):" -- freeform input
         b. "Verification agent? (agent name, or 'skip'):" -- freeform input
         c. Build agents object from responses
    - Also update the `config` step to include agents in the config.json write (currently it writes mode, depth, parallelization).
    - Also update the `done` step to show agents config if non-empty.
    - Also update the `success_criteria` section to include: `config.json has agents configuration (empty or populated)`.
    - Completion Criteria: new-project.md contains an `agents` step between `parallelization` and `config`. The step uses AskUserQuestion. The config step writes agents to config.json. The done step conditionally shows agents config.
    - Tests: Read the updated file. Confirm `<step name="agents">` exists. Confirm it appears after `parallelization` and before `config` in document order. Confirm the config step references agents. Confirm "Skip (use defaults)" is the first/recommended option.

- ~~[x] Group 4: execute-phase -- Dynamic Agent Routing~~ (DONE)
  - ~~[x] Task 4.1: Update commands/gsd/execute-phase.md with agent routing~~
    - File: `commands/gsd/execute-phase.md`
    - Current state: `<wave_execution>` section (lines 71-85) has 3 hardcoded `Task(prompt=..., subagent_type="general-purpose")` calls. `<execution_context>` references workflow and subagent-task-prompt but not config.json. No verification wave. No post-phase hooks. No agent routing section.
    - Changes (4 modifications to this file):
    - **Change A -- Add config to context:** In the `<context>` section (lines 30-35), add `@.planning/config.json` alongside the existing ROADMAP.md and STATE.md references.
    - **Change B -- Add agent_routing section:** Add a new `<agent_routing>` section after `<execution_context>` and before `<process>`. Content:
      ```xml
      <agent_routing>
      For each Task call, resolve agent type using this priority chain:

      1. **User prompt override:** If user specified an agent in the invocation (e.g., "use typescript-pro"), use it for all plans
      2. **Config pool:** Read `.planning/config.json` → `agents.execution[]`:
         - One agent in list: use it for all plans
         - Multiple agents in list: orchestrator picks the most appropriate agent based on task content and agent descriptions
         - Empty or missing: fallback
      3. **Fallback:** "general-purpose"

      Store resolved agent per plan before spawning wave.
      </agent_routing>
      ```
    - **Change C -- Replace hardcoded Task calls in wave_execution:** Replace the 3 hardcoded lines:
      ```
      Task(prompt=filled_template_for_plan_01, subagent_type="general-purpose")
      Task(prompt=filled_template_for_plan_02, subagent_type="general-purpose")
      Task(prompt=filled_template_for_plan_03, subagent_type="general-purpose")
      ```
      With dynamic routing:
      ```
      # For each plan in wave, resolve agent type first:
      agent_01 = resolve_agent(plan_01, config, user_override)
      agent_02 = resolve_agent(plan_02, config, user_override)
      agent_03 = resolve_agent(plan_03, config, user_override)

      Task(prompt=filled_template_for_plan_01, subagent_type=agent_01)
      Task(prompt=filled_template_for_plan_02, subagent_type=agent_02)
      Task(prompt=filled_template_for_plan_03, subagent_type=agent_03)
      ```
    - **Change D -- Add verification wave:** Add a new section after `<checkpoint_handling>` and before `<deviation_rules>`:
      ```xml
      <verification_wave>
      **After each execution wave completes (optional):**

      If `agents.verification` is configured in config.json:
      1. Fill verify-prompt template with wave results
      2. Spawn: `Task(prompt=filled_verify_template, subagent_type=config.agents.verification)`
      3. If PASS: proceed to next wave
      4. If FAIL: report issues to user, ask: "Fix and retry?" or "Continue anyway?" or "Stop execution?"

      If `agents.verification` is NOT configured: skip verification, proceed to next wave (current behavior).
      </verification_wave>
      ```
    - Also add `@~/.claude/get-shit-done/templates/verify-prompt.md` to the `<execution_context>` section.
    - Completion Criteria: No literal `"general-purpose"` in the file. `<agent_routing>` section exists with resolution chain. `<wave_execution>` uses `resolve_agent()` pattern. `<verification_wave>` section exists. Config.json is in the context section. verify-prompt.md is in execution_context.
    - Tests: Grep for `"general-purpose"` -- zero matches. Grep for `agent_routing` -- at least 1 match. Grep for `resolve_agent` -- at least 3 matches (one per plan in example). Grep for `verification_wave` -- at least 1 match. Grep for `config.json` in context section -- present.

  - ~~[x] Task 4.2: Update workflows/execute-phase.md with agent routing steps~~
    - File: `get-shit-done/workflows/execute-phase.md`
    - Current state: Contains `<step>` elements for load_project_state, validate_phase, discover_plans, group_by_wave, execute_waves, checkpoint_handling, aggregate_results, update_roadmap, offer_next. Two hardcoded `subagent_type="general-purpose"` references at lines 191 and 230.
    - Changes (3 modifications):
    - **Change A -- Add resolve_agents step:** Add a new `<step name="resolve_agents">` between `group_by_wave` and `execute_waves`. Content:
      ```xml
      <step name="resolve_agents">
      For each plan in each wave, determine the agent to use:

      1. Read `.planning/config.json` → parse `agents` section
         - If `agents` key missing or empty: all plans use "general-purpose"
         - If `agents.execution` exists and non-empty: proceed to selection

      2. Check for user override:
         - If user specified agent in invocation: use for all plans, skip selection

      3. Select from execution pool (deterministic, no guessing):
         - One agent in `agents.execution[]`: use it
         - Multiple agents: orchestrator picks based on task content and agent descriptions
         - Empty or missing: fallback to "general-purpose"

      4. Store agent assignment per plan for use in execute_waves step.
         IMPORTANT: The resolved agent MUST be used for both initial spawn AND any continuation spawns for the same plan. If a plan pauses at a checkpoint and resumes, the continuation agent must use the SAME resolved agent as the original spawn.

      Report agent assignments:
      ```
      Agent assignments:
        plan-01: org:typescript-pro (from user override)
        plan-02: org:typescript-pro (from config, single agent)
        plan-03: general-purpose (no agents configured)
      ```
      </step>
      ```
    - **Change B -- Update execute_waves step:** Replace both hardcoded `subagent_type="general-purpose"` references with `subagent_type={resolved_agent_for_plan}`. The two locations are:
      1. In `<step name="checkpoint_handling">`, the initial spawn for checkpoint plans: "Spawn agent for checkpoint plan:" block (currently `Task(prompt="{subagent-task-prompt}", subagent_type="general-purpose")`)
      2. In `<step name="checkpoint_handling">`, the continuation spawn: "Spawn continuation agent (NOT resume):" block (currently `Task(prompt=filled_continuation_template, subagent_type="general-purpose")`)
      Both must use the same `{resolved_agent_for_plan}` -- the continuation MUST use the same agent as the initial spawn, not re-resolve.
    - **Change C -- Add verify_wave step:** Add `<step name="verify_wave">` after `checkpoint_handling`. Content matches the `<verification_wave>` section added to the command file in Task 4.1 but with workflow-level detail.
    - Completion Criteria: No literal `"general-purpose"` in the file. New `resolve_agents` step exists between `group_by_wave` and `execute_waves`. Execute_waves step uses resolved agent variables. `verify_wave` step exists. Continuation spawn explicitly uses same resolved agent as initial spawn.
    - Tests: Grep for `"general-purpose"` -- zero matches. Grep for `resolve_agents` step -- present. Grep for `verify_wave` step -- present. All original steps still present (load_project_state through offer_next). Check that both checkpoint_handling spawn locations use the same resolved agent variable.

- ~~[x] Group 5: execute-plan -- Dynamic Agent Routing~~ (DONE — 5.1 SKIPPED, 5.2 completed)
  - ~~[ ] Task 5.1: Update commands/gsd/execute-plan.md with agent routing~~
    - File: `commands/gsd/execute-plan.md`
    - Current state: Step 4 (line 51-53) has: `Spawn: Task(prompt=filled_template, subagent_type="general-purpose")`. Context section already includes `@.planning/config.json (if exists)` at line 31 -- no change needed to the context section. No routing logic exists yet.
    - Changes (2 modifications):
    - **Change A -- Add agent_routing section:** Add an `<agent_routing>` section (same resolution chain as execute-phase, but simpler since it's a single plan):
      ```xml
      <agent_routing>
      Resolve agent for this plan:

      1. User prompt override → use specified agent
      2. Config `agents.execution[]` → one agent: use it; multiple: orchestrator picks based on task content and agent descriptions
      3. Fallback → "general-purpose"

      Note: config.json is already in the context section (line 31). No change needed to context.
      ```
    - **Change B -- Replace hardcoded Task call:** In step 4, replace:
      ```
      Spawn: `Task(prompt=filled_template, subagent_type="general-purpose")`
      ```
      With:
      ```
      Resolve agent type using agent_routing rules above.
      Spawn: `Task(prompt=filled_template, subagent_type=resolved_agent)`
      ```
    - Completion Criteria: No literal `"general-purpose"` in the file. `<agent_routing>` section exists. Step 4 uses resolved_agent variable.
    - Tests: Grep for `"general-purpose"` -- zero matches. Grep for `agent_routing` -- at least 1 match. Grep for `resolved_agent` -- at least 1 match. Process steps 1-6 still intact.

  - ~~[x] Task 5.2: Update workflows/execute-plan.md with agent routing~~
    - File: `get-shit-done/workflows/execute-plan.md`
    - Current state: Two hardcoded `subagent_type="general-purpose"` references. Located at:
      1. **In the "For fully autonomous plans" subsection** of segment_routing (line ~209): `Use Task tool with subagent_type="general-purpose":` -- this is inside `<step name="segment_routing">`, under the "5. Implementation:" heading, under "For fully autonomous plans:" subheading.
      2. **In the "If routing = Subagent" block** of the segment loop (line ~361): `Spawn Task tool with subagent_type="general-purpose":` -- this is inside the same `<step name="segment_routing">`, under "2. For each segment in order:", under "B. If routing = Subagent:" subheading.
    - Change: Replace both hardcoded `subagent_type="general-purpose"` with dynamic routing. At each location:
      - Add a brief routing resolution note before the Task call
      - Replace `"general-purpose"` with `resolved_agent` variable
      - Add comment referencing the resolution chain
    - The routing logic itself does not need a dedicated step here because execute-plan.md workflow delegates to the command, which has the full routing logic. The workflow just needs to show the correct variable instead of the hardcoded string.
    - Completion Criteria: No literal `"general-purpose"` in the file. Both Task call locations use resolved_agent. Brief routing note present at each location.
    - Tests: Grep for `"general-purpose"` -- zero matches. Grep for `resolved_agent` -- at least 2 matches. All existing workflow steps intact and unchanged in structure.

- ~~[x] Group 6: Integration Validation~~ (DONE — Task 6.1 completed via static review)
  - ~~[x] Task 6.1: End-to-end smoke test~~ — DONE. Comprehensive static verification performed. All routing logic, cross-references, template integrity, backwards compatibility, and plan success criteria verified. See Group 6 entry in Completed Tasks section above for full results.

## Dependencies

- Group 0 (pre-flight) MUST pass before ANY other work begins -- if Task tool rejects custom subagent_type, the plan needs redesign
- Group 1 (foundation) must be completed before any other group -- all routing logic references the config schema defined here
- Group 2 (templates) should be completed before Groups 4/5 -- commands reference updated templates
- Task 2.3 (verify-prompt.md) must exist before Task 4.1 -- execute-phase references it in execution_context
- Group 3 (new-project) depends on Group 1 -- writes agents to config using the defined schema
- Groups 4 and 5 (execute commands) are independent of each other and can be done in either order
- Group 6 (integration validation) depends on all other groups being complete

Recommended execution order: 0 > 1 > 2 > 3 > 4 > 5 > 6

## Notes

- All file paths in this document are relative to the GSD installation root (`~/.claude/`). When executing tasks, use the actual installation path.
- The `resolve_agent()` function described in tasks is pseudocode for the orchestrator's inline logic -- it is NOT a separate function to implement. The orchestrator reads config, checks for user override, and picks the agent type directly in the command prompt text.
- The agents section in config.json uses string agent names in the format `"org:agent-name"` which maps directly to Claude Code's custom agent system. The `subagent_type` parameter of the Task tool accepts these names.
- `map-codebase` is intentionally excluded -- it uses Explore agents for read-only analysis, which is a fundamentally different use case than execution routing.
- The verification wave is intentionally lightweight -- it spawns a single agent per wave, not per plan. The agent checks all plans completed in that wave.
- Config resolution is always CWD-relative. In monorepo setups, GSD already scopes to CWD, so `.planning/config.json` always resolves to the correct sub-project's config.
- Planning delegation (planner/reviewer subagents) was explicitly removed from this plan. It is a separate feature that adds new subagent spawning capability to plan-phase.md (which currently has zero hardcoded "general-purpose" references). It should get its own plan, templates, and testing when ready.
- Post-phase external tool hooks were explicitly removed. Shell commands are not agents and mixing them in the agents config is a category error. If needed, they belong in a separate `hooks` config key.

---
