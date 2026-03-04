---

# Planning Pipeline Agent Routing Implementation

Extend the GSD planning pipeline (`/gsd:plan-phase`) to support custom agent routing via `config.json`, using the same 3-level resolution chain as the execution pipeline (user override > config > fallback). Adds `<return_protocol>` injection -- a new concept not present in execution routing -- so custom planning agents know what signal headers the orchestrator expects in their responses.

## Completed Tasks

- [x] Phase 1: Foundation -- Config Template Update
  - [x] Task 1.1: Add planning agent keys to config-agents-example.json
    - Source: Read `/mnt/3b92ea25-2e45-41c8-97d3-58aa8141755e/Videos/Projects/NIIT/gsd-agents/get-shit-done/templates/config-agents-example.json`
    - Current state: Has `execution` (array) and `verification` (string) keys, plus `_comment` explanations. The `_comment_defaults` says fallback is "default agent types (general-purpose)" which will be wrong after this change. No planning-related keys.
    - Changes (2 modifications):
    - **Change A -- Add 3 new keys** to the `agents` object and corresponding `_comment` keys. All 3 planning keys are strings (not arrays). Unlike `execution` (array, multiple plans per wave with different domains), planning spawns one agent per role per phase -- no multi-agent selection needed:
      ```json
      {
        "_comment_planning": "Single agent name for plan creation and revision. Same agent handles revisions after checker feedback. Optional. When omitted, falls back to built-in gsd-planner.",
        "_comment_research": "Single agent name for phase research before planning. Optional. When omitted, falls back to built-in gsd-phase-researcher.",
        "_comment_plan_check": "Single agent name for plan verification before execution. Optional. When omitted, falls back to built-in gsd-plan-checker.",
        "agents": {
          "execution": ["org:agent-name-1", "org:agent-name-2"],
          "verification": "org:verifier-agent",
          "planning": "org:custom-planner",
          "research": "org:custom-researcher",
          "plan_check": "org:custom-checker"
        }
      }
      ```
    - **Change B -- Update `_comment_defaults`** to reflect named agent type fallbacks instead of "general-purpose":
      - Current: `"_comment_defaults": "When the agents key is empty ({}) or missing entirely, all routing falls back to default agent types (general-purpose)."`
      - New: `"_comment_defaults": "When the agents key is empty ({}) or missing entirely, all routing falls back to built-in GSD agents (gsd-executor, gsd-verifier, gsd-phase-researcher, gsd-planner, gsd-plan-checker)."`
    - Preserve all existing `_comment` keys (except `_comment_defaults` which gets updated) and the existing `execution` and `verification` entries.
    - Completion Criteria: File is valid JSON. Contains all 5 agent keys (`execution`, `verification`, `planning`, `research`, `plan_check`). `planning`, `research`, `plan_check` are strings (not arrays). All `_comment` keys explain usage. `_comment_defaults` references named agent types, not "general-purpose". Existing `execution` and `verification` values unchanged.
    - Tests: Parse with `python3 -m json.tool` to confirm valid JSON. Verify all 5 keys present in `agents` object. Verify `planning`, `research`, `plan_check` are string values (not arrays). Verify `_comment_defaults` says "built-in GSD agents" not "general-purpose". Verify existing `execution` and `verification` values unchanged.

## Future Tasks

- [ ] Phase 2: Core Routing -- plan-phase.md Modifications
  - [x] Task 2.1: Add `<agent_routing>` section to plan-phase.md
    - Source: Read `/mnt/3b92ea25-2e45-41c8-97d3-58aa8141755e/Videos/Projects/NIIT/gsd-agents/commands/gsd/plan-phase.md` (the orchestrator to modify) and `/mnt/3b92ea25-2e45-41c8-97d3-58aa8141755e/Videos/Projects/NIIT/gsd-agents/commands/gsd/execute-phase.md` lines 295-308 (the `<agent_routing>` section to use as structural reference for layout)
    - Change: Add a new `<agent_routing>` section to plan-phase.md. Position it between `</context>` (line 41) and `<process>` (line 43). This section defines:
      - **Resolution chain per role** -- 3-level priority (user override > config > fallback) for each of the 3 planning roles. All config values are strings (single agent name per role):
        - Researcher: `agents.research` config key (string), fallback `gsd-phase-researcher`
        - Planner: `agents.planning` config key (string), fallback `gsd-planner`
        - Checker: `agents.plan_check` config key (string), fallback `gsd-plan-checker`
      - **Consistency rule** -- Resolved planner agent is stored and reused for revision loops (step 12). Same agent creates plans and handles revisions.
      - **Workflow toggle interaction** -- Agent resolution only happens when the corresponding workflow toggle is enabled. If `workflow.research: false`, skip research entirely (agents.research is irrelevant). If `workflow.plan_check: false`, skip verification entirely (agents.plan_check is irrelevant). The `planning` role has no workflow toggle -- planner always runs.
      - **Prompt construction logic** -- When to use built-in vs custom agent prompts:
        - Built-in (fallback): Use `subagent_type="gsd-phase-researcher"` (named agent type, Claude Code loads agent def automatically). Prompt = context only. No "First, read ~/.claude/agents/..." prefix needed.
        - Custom: Use `subagent_type=resolved_agent` (config/override value). Prompt = context + appended `<return_protocol>` block.
      - **Return protocol blocks** -- Define the exact `<return_protocol>` XML for each role. These are appended to custom agent prompts only. CRITICAL: Include ONLY signals the orchestrator actually parses. Do NOT include signals from agent definitions that the orchestrator never checks.
        - **Researcher return_protocol** (2 signals): Must return `## RESEARCH COMPLETE` (with Phase, Confidence, Key Findings, File Created) on success, or `## RESEARCH BLOCKED` (with Blocked by, Attempted, Options) on failure. Research output goes to `{phase_dir}/{phase}-RESEARCH.md`. Source: verified against orchestrator's researcher handler after step 5.
        - **Planner return_protocol** (3 signals): Must return one of: `## PLANNING COMPLETE` (with Phase, Plans, Wave Structure), `## CHECKPOINT REACHED` (with Type, Decision Needed), or `## PLANNING INCONCLUSIVE` (with what was attempted). Source: verified against orchestrator step 9 parsing logic. NOTE: Do NOT include `## REVISION COMPLETE` or `## GAP CLOSURE PLANS CREATED` -- these exist in the gsd-planner agent definition but are never parsed by the orchestrator. The same 3 signals apply for both initial planning (step 8) and revision (step 12) spawns.
        - **Checker return_protocol** (2 signals): Must return `## VERIFICATION PASSED` (with Coverage Summary, Plan Summary) or `## ISSUES FOUND` (with severity counts, Blockers list, Warnings list, Structured Issues YAML block). Source: verified against orchestrator step 11 parsing logic.
    - Follow the structural layout of execute-phase.md's `<agent_routing>` section. Note: execute-phase does zero return_protocol injection (execution agents write files to disk). The return_protocol concept is new to the planning pipeline.
    - Completion Criteria: `<agent_routing>` section exists in plan-phase.md between `</context>` and `<process>`. Contains resolution chains for all 3 roles (all strings, no arrays). Contains return_protocol blocks for all 3 roles. Planner return_protocol has exactly 3 signals (NOT 5). Contains prompt construction logic (built-in vs custom). Contains consistency rule. Contains workflow toggle interaction note.
    - Tests: Grep for `<agent_routing>` in plan-phase.md -- at least 1 match. Grep for `return_protocol` -- at least 3 matches (one per role). Grep for `gsd-phase-researcher` as fallback -- present. Grep for `gsd-planner` as fallback -- present. Grep for `gsd-plan-checker` as fallback -- present. Grep for `consistency` or `revision` rule -- present. Grep for `REVISION COMPLETE` inside the return_protocol section -- zero matches (this signal must NOT appear). Grep for `workflow.plan_check` or `workflow.research` -- present (toggle interaction documented).

  - [x] Task 2.2: Modify researcher spawn point (step 5) to use resolved agent
    - Source: Read `/mnt/3b92ea25-2e45-41c8-97d3-58aa8141755e/Videos/Projects/NIIT/gsd-agents/commands/gsd/plan-phase.md` lines 172-231 (step 5, "Spawn gsd-phase-researcher" through "Handle Researcher Return")
    - Current state (lines 225-230):
      ```
      Task(
        prompt="First, read ~/.claude/agents/gsd-phase-researcher.md for your role and instructions.\n\n" + research_prompt,
        subagent_type="general-purpose",
        model="{researcher_model}",
        description="Research Phase {phase}"
      )
      ```
    - Changes (2 modifications at this spawn point):
    - **Change A -- Add agent resolution before spawn:** Before the Task() call, add:
      ```
      # Resolve researcher agent (from <agent_routing> resolution chain)
      resolved_researcher = resolve_agent("research", config, user_override)
      # Fallback: "gsd-phase-researcher"
      ```
    - **Change B -- Replace Task() call with conditional prompt construction:**
      ```
      # If custom agent (resolved_researcher != "gsd-phase-researcher"):
      Task(
        prompt=research_prompt + "\n\n" + researcher_return_protocol,
        subagent_type=resolved_researcher,
        model="{researcher_model}",
        description="Research Phase {phase}"
      )

      # If built-in (resolved_researcher == "gsd-phase-researcher"):
      Task(
        prompt=research_prompt,
        subagent_type="gsd-phase-researcher",
        model="{researcher_model}",
        description="Research Phase {phase}"
      )
      ```
      Note: Built-in path no longer needs "First, read ~/.claude/agents/gsd-phase-researcher.md..." because using the named agent type (`subagent_type="gsd-phase-researcher"`) causes Claude Code to load the agent definition automatically.
    - The "Handle Researcher Return" section (lines 233-242) requires NO changes -- it already parses `## RESEARCH COMPLETE` and `## RESEARCH BLOCKED`, which are the same signals defined in the return_protocol.
    - Completion Criteria: No `subagent_type="general-purpose"` at this spawn point. No "First, read ~/.claude/agents/gsd-phase-researcher.md" instruction. Conditional prompt construction (custom vs built-in). return_protocol appended for custom agents only. Built-in path uses `subagent_type="gsd-phase-researcher"`.
    - Tests: Read lines around the researcher spawn. Confirm `resolved_researcher` variable used. Confirm conditional prompt construction present. Confirm no "general-purpose" at this location. Confirm return_protocol referenced for custom path.

  - [x] Task 2.3: Modify planner + revision planner spawn points (steps 8 + 12) with consistency rule
    - Source: Read `/mnt/3b92ea25-2e45-41c8-97d3-58aa8141755e/Videos/Projects/NIIT/gsd-agents/commands/gsd/plan-phase.md` lines 271-346 (step 8, planner spawn) and lines 443-495 (step 12, revision planner spawn)
    - Current state -- Planner spawn (lines 340-345):
      ```
      Task(
        prompt="First, read ~/.claude/agents/gsd-planner.md for your role and instructions.\n\n" + filled_prompt,
        subagent_type="general-purpose",
        model="{planner_model}",
        description="Plan Phase {phase}"
      )
      ```
    - Current state -- Revision planner spawn (lines 489-493):
      ```
      Task(
        prompt="First, read ~/.claude/agents/gsd-planner.md for your role and instructions.\n\n" + revision_prompt,
        subagent_type="general-purpose",
        model="{planner_model}",
        description="Revise Phase {phase} plans"
      )
      ```
    - Changes (3 modifications):
    - **Change A -- Add planner resolution before step 8 spawn:** Before the Task() call in step 8, add:
      ```
      # Resolve planner agent (from <agent_routing> resolution chain)
      resolved_planner = resolve_agent("planning", config, user_override)
      # Fallback: "gsd-planner"
      # IMPORTANT: Store resolved_planner — same agent handles revisions in step 12
      ```
    - **Change B -- Replace step 8 Task() with conditional prompt:**
      ```
      # If custom agent (resolved_planner != "gsd-planner"):
      Task(
        prompt=filled_prompt + "\n\n" + planner_return_protocol,
        subagent_type=resolved_planner,
        model="{planner_model}",
        description="Plan Phase {phase}"
      )

      # If built-in (resolved_planner == "gsd-planner"):
      Task(
        prompt=filled_prompt,
        subagent_type="gsd-planner",
        model="{planner_model}",
        description="Plan Phase {phase}"
      )
      ```
    - **Change C -- Replace step 12 Task() with same resolved agent (consistency rule):**
      ```
      # CONSISTENCY RULE: Use the SAME resolved_planner from step 8
      # If custom agent:
      Task(
        prompt=revision_prompt + "\n\n" + planner_return_protocol,
        subagent_type=resolved_planner,
        model="{planner_model}",
        description="Revise Phase {phase} plans"
      )

      # If built-in:
      Task(
        prompt=revision_prompt,
        subagent_type="gsd-planner",
        model="{planner_model}",
        description="Revise Phase {phase} plans"
      )
      ```
      The resolved_planner variable is resolved once (step 8) and reused (step 12). This ensures the same custom agent that created the plans handles revisions.
    - The "Handle Planner Return" section (step 9, lines 349-366) requires NO changes -- it already parses `## PLANNING COMPLETE`, `## CHECKPOINT REACHED`, and `## PLANNING INCONCLUSIVE`, which are the same 3 signals in the return_protocol. The revision loop (step 12, lines 443-509) does NOT parse a specific return signal -- after the revision agent returns, it routes straight back to the checker (step 10). The revision spawn uses the same 3 planner signals.
    - Completion Criteria: No `subagent_type="general-purpose"` at either spawn point. No "First, read ~/.claude/agents/gsd-planner.md" instruction at either spawn point. Both spawns use `resolved_planner` variable. Step 12 explicitly reuses the resolved_planner from step 8 (consistency rule documented). return_protocol appended for custom agents only.
    - Tests: Grep for `resolved_planner` -- at least 2 matches (step 8 and step 12). Confirm consistency rule comment present at step 12 spawn. Confirm no "general-purpose" at either planner spawn. Confirm planner return_protocol referenced for custom paths.

  - [x] Task 2.4: Modify checker spawn point (step 10) to use resolved agent
    - Source: Read `/mnt/3b92ea25-2e45-41c8-97d3-58aa8141755e/Videos/Projects/NIIT/gsd-agents/commands/gsd/plan-phase.md` lines 367-429 (step 10, checker spawn)
    - Current state (lines 423-428):
      ```
      Task(
        prompt=checker_prompt,
        subagent_type="gsd-plan-checker",
        model="{checker_model}",
        description="Verify Phase {phase} plans"
      )
      ```
      Note: The checker already uses the named agent type `"gsd-plan-checker"` (not "general-purpose"). This is closer to the target pattern but still hardcoded -- it doesn't check config for a custom checker.
    - Changes (3 modifications):
    - **Change A -- Add checker resolution before step 10 spawn:** Before the Task() call in step 10, add:
      ```
      # Resolve checker agent (from <agent_routing> resolution chain)
      resolved_checker = resolve_agent("plan_check", config, user_override)
      # Fallback: "gsd-plan-checker"
      # IMPORTANT: Store resolved_checker — same agent used for re-verification in step 12
      ```
    - **Change B -- Replace step 10 Task() with conditional prompt:**
      ```
      # If custom agent (resolved_checker != "gsd-plan-checker"):
      Task(
        prompt=checker_prompt + "\n\n" + checker_return_protocol,
        subagent_type=resolved_checker,
        model="{checker_model}",
        description="Verify Phase {phase} plans"
      )

      # If built-in (resolved_checker == "gsd-plan-checker"):
      Task(
        prompt=checker_prompt,
        subagent_type="gsd-plan-checker",
        model="{checker_model}",
        description="Verify Phase {phase} plans"
      )
      ```
    - **Change C -- Update step 12 checker re-verification to use resolved_checker:** In step 12 (revision loop), after the revision planner returns, the orchestrator spawns the checker again (currently says "spawn checker again (step 10)"). This re-verification spawn is a 5th Task() call that must also use the same `resolved_checker` from Change A. Apply the same conditional prompt construction as Change B. This ensures that if a custom checker verified the initial plans, the same custom checker re-verifies revised plans.
    - The "Handle Checker Return" section (step 11, lines 431-440) requires NO changes -- it already parses `## VERIFICATION PASSED` and `## ISSUES FOUND`, which match the return_protocol signals.
    - Completion Criteria: No hardcoded `"gsd-plan-checker"` as the only path at either spawn point. Conditional prompt construction present at both step 10 and step 12 checker spawns. return_protocol appended for custom checkers only. Step 12 re-verification explicitly reuses resolved_checker from step 10.
    - Tests: Grep for `resolved_checker` -- at least 2 matches (step 10 and step 12). Confirm conditional prompt construction at both checker spawn locations. Confirm checker return_protocol referenced for custom path at both locations. Confirm step 12 re-verification comment documents reuse of resolved_checker.

- [ ] Phase 3: Setup Flow -- new-project.md Extension
  - [x] Task 3.1: Extend Round 3 agent pool questions to cover planning agents
    - Source: Read `/mnt/3b92ea25-2e45-41c8-97d3-58aa8141755e/Videos/Projects/NIIT/gsd-agents/commands/gsd/new-project.md` lines 338-401 (Round 3 -- Agent pool section)
    - Current state: Round 3 asks "Configure specialized agents?" and if "Configure agents" is chosen, asks 2 follow-up questions inline:
      1. "Execution agents (comma-separated agent names, or 'skip'):"
      2. "Verification agent? (agent name, or 'skip'):"
      Then builds the agents object from responses.
    - Changes (3 modifications):
    - **Change A -- Add 3 more follow-up questions:** After the existing 2 questions, add. All 3 planning keys are single agent names (strings), not arrays:
      ```
      3. "Planning agent? (agent name, or 'skip'):"
         - If user provides a name: store as string in agents.planning
         - If "skip": omit planning key
      4. "Research agent? (agent name, or 'skip'):"
         - If user provides a name: store as string in agents.research
         - If "skip": omit research key
      5. "Plan checker agent? (agent name, or 'skip'):"
         - If user provides a name: store as string in agents.plan_check
         - If "skip": omit plan_check key
      ```
    - **Change B -- Update agents object examples:** After the existing examples showing `execution` and `verification`, add examples with planning keys:
      ```json
      // If all configured:
      {
        "execution": ["org:agent-1"],
        "verification": "org:verifier",
        "planning": "org:custom-planner",
        "research": "org:custom-researcher",
        "plan_check": "org:custom-checker"
      }

      // If only planning configured:
      { "planning": "org:custom-planner" }
      ```
    - **Change C -- Update Phase 10 done section:** The conditional agent pool display currently shows `execution=[agent list], verification=[agent name]`. Extend to also show planning, research, plan_check when configured:
      ```
      **Agent pool:** execution=[list], verification=[name], planning=[name], research=[name], plan_check=[name]
      ```
      Only show keys that are actually configured (non-empty).
    - Completion Criteria: Round 3 "Configure agents" path asks about planning, research, and plan_check agents. All 3 new keys use single agent name input (not comma-separated). Agents object examples include all 5 keys. Phase 10 done section shows all configured agent keys.
    - Tests: Read new-project.md. Confirm "Planning agent?" question exists after "Verification agent" question. Confirm "Research agent?" question exists. Confirm "Plan checker agent?" question exists. Confirm agents object examples show `planning`, `research`, `plan_check` as strings (not arrays). Confirm Phase 10 done section references planning agent keys.

- [x] Phase 4: Integration Validation
  - [x] Task 4.1: Static verification of all changes
    - Purpose: Comprehensive review of all modified files to confirm correctness, consistency, and backwards compatibility.
    - Verification checklist:
      1. **config-agents-example.json:** Valid JSON. All 5 agent keys present. `planning`, `research`, `plan_check` are strings (not arrays). All _comment keys present. `_comment_defaults` references named agent types (not "general-purpose"). Existing `execution` and `verification` entries unchanged.
      2. **plan-phase.md routing consistency:** `<agent_routing>` section exists. Resolution chain documented for all 3 roles (all strings, no array handling). Return protocol blocks defined for all 3 roles. Workflow toggle interaction documented (workflow.plan_check: false and workflow.research: false make their respective agent configs irrelevant).
      3. **plan-phase.md spawn points:** All 5 spawn points use `resolved_*` agent variables (researcher step 5, planner step 8, checker step 10, revision planner step 12, checker re-verification step 12). No hardcoded `"general-purpose"` at any spawn point. Conditional prompt construction at each spawn point (custom vs built-in). return_protocol appended only for custom agents.
      4. **plan-phase.md consistency rule:** resolved_planner stored at step 8, reused at step 12. Comment explicitly documents this.
      5. **plan-phase.md fallback agents:** Researcher fallback is `gsd-phase-researcher`. Planner fallback is `gsd-planner`. Checker fallback is `gsd-plan-checker`. These match the named agent types in the agent definition files.
      6. **Return protocol signal accuracy (CRITICAL):** Only signals the orchestrator actually parses are included. Researcher: `## RESEARCH COMPLETE`, `## RESEARCH BLOCKED` (verified against researcher handler). Planner: exactly 3 signals -- `## PLANNING COMPLETE`, `## CHECKPOINT REACHED`, `## PLANNING INCONCLUSIVE` (verified against step 9). NOT `## REVISION COMPLETE`, NOT `## GAP CLOSURE PLANS CREATED` (these exist in the gsd-planner agent def but are never parsed by the orchestrator). Checker: `## VERIFICATION PASSED`, `## ISSUES FOUND` (verified against step 11).
      7. **new-project.md:** Round 3 asks about all 5 agent types (execution, verification, planning, research, plan_check). All 3 new keys use single agent name input (not comma-separated). Config write includes all configured agents. Phase 10 displays all configured agents.
      8. **Backwards compatibility:** Empty `"agents": {}` in config = all 3 planning roles use built-in agents. No planning agent keys = same behavior as today. All routing paths have explicit fallbacks. Default path change (general-purpose to named agent type) is functionally equivalent for researcher and planner.
      9. **No unintended changes:** Verify the file sections outside the modification areas are unchanged. Verify no accidental changes to the planner return parsing (steps 9, 11), researcher return parsing, or revision loop logic.
      10. **Cross-file consistency:** config-agents-example.json key names (`planning`, `research`, `plan_check`) match plan-phase.md routing keys match new-project.md question labels. All are strings everywhere (no mismatched types).
    - Completion Criteria: All 10 checklist items pass. Any failures documented with specific issue descriptions.
    - Tests: Read each modified file. For each checklist item, grep or visually confirm. Document pass/fail for each item. If any fail, document what needs fixing.

## Dependencies

- Phase 1 (config template) should be completed before Phase 2 (plan-phase.md) -- the routing section references config key names
- Phase 2 tasks have internal ordering: Task 2.1 (routing section) must be done first, then Tasks 2.2-2.4 can be done in any order
- Phase 3 (new-project.md) is independent of Phase 2 -- both depend only on Phase 1
- Phase 4 (integration) depends on all other phases being complete

Recommended execution order: 1.1 > 2.1 > 2.2 > 2.3 > 2.4 > 3.1 > 4.1

## Notes

- All file paths are absolute paths on this machine. The files live within the GSD plugin installation at `/mnt/3b92ea25-2e45-41c8-97d3-58aa8141755e/Videos/Projects/NIIT/gsd-agents/`.
- The `resolve_agent()` function described in tasks is pseudocode for the orchestrator's inline logic -- it is NOT a separate function to implement. The orchestrator reads config, checks for user override, and picks the agent type directly in the command prompt text.
- The `<return_protocol>` block content should be derived by reading the `<structured_returns>` sections of each built-in agent definition file, BUT only include signals the orchestrator actually parses. The planner agent definition has 5 signals; the orchestrator only parses 3. Including unrecognized signals in the return_protocol would mislead custom agents.
- **Return protocol injection is NOT from the execution pattern.** The execution pipeline does zero return_protocol injection (execution agents write SUMMARY.md to disk). Planning agents return signal headers in response text. This is a new concept specific to the planning pipeline.
- When the built-in agent is used (fallback path), the prompt no longer needs "First, read ~/.claude/agents/gsd-*.md for your role and instructions." This instruction was a workaround for using `subagent_type="general-purpose"`. With named agent types (`subagent_type="gsd-planner"`), Claude Code loads the agent definition automatically. This same change was made in the execution pipeline routing. **This changes default behavior for ALL projects**, not just those with custom agents. It is functionally equivalent but worth noting.
- The `config.json` template file (`get-shit-done/templates/config.json`) does NOT need changes -- it already has `"agents": {}` as an empty default from the execution routing work.
- The existing `workflow.plan_check` config key (boolean, controls whether checker runs at all) is separate from `agents.plan_check` (string, controls which agent runs as checker). Both can coexist: `workflow.plan_check: false` disables the checker entirely and makes `agents.plan_check` irrelevant (silently ignored). Same for `workflow.research: false` making `agents.research` irrelevant. This interaction must be documented in the `<agent_routing>` section.

---
