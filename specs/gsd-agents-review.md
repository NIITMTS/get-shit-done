# GSD Agent Routing -- Plan & Tasks Review

Reviewer: Devils Advocate Developer
Date: 2026-02-26
Files reviewed: `specs/gsd-agents-plan.md`, `specs/gsd-agents-tasks.md`
Cross-referenced against: 10 GSD source files (commands, templates, workflows)

---

## APPROVED

**Hardcoded reference inventory is accurate.** The plan claims 6 hardcoded `"general-purpose"` in commands/templates and 4 in workflows. I grepped every file. The counts are exact:

- Commands/templates: `execute-phase.md` (3), `execute-plan.md` (1), `subagent-task-prompt.md` (1), `continuation-prompt.md` (1) = 6
- Workflows: `execute-phase.md` (2), `execute-plan.md` (2) = 4

**File coverage is complete.** 11 files in the plan table, 11 tasks in the tasks file. Every file that contains `"general-purpose"` has a corresponding task. No files missed.

**Backwards compatibility design is sound.** Empty `"agents": {}` as default, with every routing path falling back to `"general-purpose"` when absent, is the right approach. The resolution chain (user override > plan frontmatter > config pool > fallback) is clean and has clear precedence.

**Task dependency ordering is correct.** Group 1 (config schema) before everything else, Group 2 (templates) before Groups 4/5 (commands that reference templates), Group 3 independent after Group 1, Groups 4/5 independent of each other. The execution order 1 > 2 > 3 > 4 > 5 > 6 works.

**Scope exclusions are well-chosen.** Leaving `map-codebase.md` alone (uses `Explore` agents, different purpose) and not touching checkpoint handling are correct calls that prevent scope creep.

**The note about `resolve_agent()` being pseudocode, not a real function, is important and correctly called out** in the tasks file notes section. This prevents an implementer from trying to create an actual function.

---

## CONCERNS

### CONCERN 1: The "domain matching" logic is under-specified and risky (HIGH)

The plan says the orchestrator does "simple keyword/domain match" -- matching the plan's `domain` frontmatter field against agent names. But this is vague enough to cause inconsistent implementations.

**Evidence from plan (line 69-75):**
```
1. Read the plan's `domain` frontmatter field (e.g., "frontend", "api", "database")
2. Match against agent names/descriptions in the pool
3. If clear match: use that agent
4. If ambiguous or no match: use the first agent in the execution list
```

**Problems:**
- "Match against agent names/descriptions" -- where do descriptions come from? The config schema only stores names like `"org:agent-name-1"`. Agent descriptions live in `.claude/agents/` definition files. Is the orchestrator supposed to read those files? That would blow up the 5-line routing budget.
- "If clear match" -- what is "clear"? Does `domain: "frontend"` match `"org:react-specialist"` or not? The plan says keyword match but the example domains (`frontend`, `api`, `database`) won't substring-match typical agent names (`org:typescript-pro`, `org:frontend-dev`).
- The task pseudocode (Task 4.2) says "Find agent in `agents.execution[]` whose name contains domain keyword." This is more specific than the plan but still fragile. Agent `org:frontend-dev` contains "frontend" but `org:react-specialist` does not, even though React is frontend.

**Suggested fix:** Drop the domain-matching entirely. Simplify to: if `agents.execution` has one agent, use it for everything. If it has multiple agents, the plan frontmatter `agents:` field or user override must specify which one. No guessing. This removes an entire class of bugs and keeps the orchestrator truly lean. If the user configures 3 execution agents, they should explicitly map them per-plan via frontmatter, not rely on fuzzy name matching. This is also more honest -- the plan already has the `agents:` frontmatter field (Task 1.2) which is the right mechanism for per-plan agent selection.

### CONCERN 2: Task 5.1 makes a false claim about execute-plan.md (MEDIUM)

Task 5.1 says:
> "Context section references STATE.md and config.json but not for agent routing."

This is accurate -- `execute-plan.md` line 31 already has `@.planning/config.json (if exists)` in its context. But then Task 5.1 Change A says to "Add agent_routing section" and Change B says to add "Read config: `@.planning/config.json` (already in context)".

The problem is the task **does not list adding config.json to the context section as a change**, because it is already there. Good -- this is actually correct. But the task description's phrasing makes it sound like the config reference is being added, when it is not. Minor, but an implementer could waste time looking for where to add it.

**Suggested fix:** Change the task description to explicitly state: "config.json is already in the context section (line 31). No change needed to the context section."

### CONCERN 3: Task 5.2 line references may be fragile (LOW)

Task 5.2 claims hardcoded references at lines 209 and 361 of `workflows/execute-plan.md`. I verified both exist:

- Line 209: `Use Task tool with subagent_type="general-purpose":`
- Line 361: `Spawn Task tool with subagent_type="general-purpose":`

Both confirmed. However, `execute-plan.md` is a 394-line file with complex structure. The task says "Replace both hardcoded references" but does not provide enough context about the surrounding text to find them unambiguously. The line numbers might shift if any prior task modifies this file (though none should in this plan).

**Suggested fix:** Include 2-3 lines of surrounding context for each replacement point, not just line numbers. Example: "In the 'For fully autonomous plans' subsection of segment_routing, replace..." and "In the 'If routing = Subagent' block of the segment loop, replace..."

### CONCERN 4: The config schema has unnecessary structural complexity (MEDIUM)

The proposed config agents schema:
```json
"agents": {
    "execution": ["org:agent-a", "org:agent-b"],
    "planning": {
        "planner": "org:planner-agent",
        "reviewer": "org:reviewer-agent"
    },
    "verification": "org:verifier-agent",
    "post_phase": {
        "compile_lint": "org:compile-agent",
        "external": ["command --flag"]
    }
}
```

This is four different shapes in one object: an array (`execution`), a nested object (`planning`), a string (`verification`), and a mixed nested object (`post_phase` with string + array). Four different access patterns for one config section.

**Why this matters:** Every orchestrator command that reads this config must handle each shape differently. The routing logic in execute-phase alone needs to check `agents.execution` (array), `agents.verification` (string), and `agents.post_phase` (nested object). That is not "5 lines of pseudocode."

**Suggested fix:** Consider a flatter or more uniform structure. For example:
```json
"agents": {
    "execution": ["org:agent-a"],
    "planner": "org:planner-agent",
    "reviewer": "org:reviewer-agent",
    "verifier": "org:verifier-agent"
}
```
Drop `post_phase.external` from the agents config entirely -- external shell commands are not agents. They belong in a separate `hooks` config key or should be manually configured. Mixing shell commands and agent names in one config section is a category error.

### CONCERN 5: Planning delegation (Group 6) is a scope expansion disguised as a feature (HIGH)

The original goal is clear: "route tasks to specialized agents from a configurable pool, instead of hardcoding `subagent_type="general-purpose"`."

But Task 6.1 (planning delegation) is not routing an existing `subagent_type="general-purpose"` call to a different agent. `plan-phase.md` has ZERO hardcoded `"general-purpose"` references -- I verified this with grep. It currently does all planning work in the main context, not via subagents at all.

**What Task 6.1 actually does:** It introduces an entirely new subagent spawning capability that does not exist today. It adds:
- A new delegation decision branch in the planning workflow
- A planner agent spawn (new Task call that did not exist)
- A reviewer agent spawn (another new Task call that did not exist)
- An iteration loop between planner and reviewer (up to 2 iterations)
- New prompt construction for both planner and reviewer agents

This is not "replacing hardcoded general-purpose with dynamic routing." This is adding a new feature (planning delegation) that happens to use the same agent routing mechanism. It is architecturally different from Tasks 2.1-5.2 which are all find-and-replace operations on existing code paths.

**Risks specific to this task:**
- The planner agent needs a prompt template. Where is it? Task 6.1 describes what the planner prompt "includes" but there is no template file for it. Every other agent spawn in GSD uses a template (subagent-task-prompt.md, continuation-prompt.md, verify-prompt.md). This one would be ad-hoc.
- The reviewer iteration loop ("Max 2 iterations, then present to user") adds control flow complexity to an orchestrator that currently has none in plan-phase.
- Context budget: plan-phase currently uses ~100% of context for planning. Spawning a planner subagent means the planner gets ~100% context but the orchestrator must still gather all context (steps 1-5) to pass to it. If context gathering alone approaches 15%, the value of delegation is marginal because the planner agent gets the same 200k window anyway.

**Suggested fix:** Remove Group 6 from this plan entirely. Ship Groups 1-5 (the actual routing work) first. Planning delegation is a separate feature that deserves its own plan, its own template, and its own testing. It can reference the config schema established by this plan without being bundled into it.

### CONCERN 6: Verification wave prompt has no error handling for missing build/test commands (LOW)

The verify-prompt template (Task 2.3) instructs the verification agent to:
```
1. Compile check: Run the project's build/compile command.
2. Lint check: Run the project's lint command.
3. Test suite: Run the project's test command.
```

But there is no mechanism for the verification agent to discover what these commands are. Different projects use `npm run build`, `cargo build`, `go build`, `make`, etc. The verify-prompt does not reference `PROJECT.md` or `config.json` for this information, and neither of those files currently stores build/test/lint commands.

**Suggested fix:** Either add a `commands` section to config.json (`"commands": { "build": "npm run build", "test": "npm test", "lint": "npm run lint" }`) or add `@.planning/PROJECT.md` to the verify-prompt context section so the agent can infer commands from the project description. The second option is simpler and avoids config expansion.

### CONCERN 7: The plan claims Task tool accepts custom subagent_type values but provides no evidence (MEDIUM)

The plan states under External Dependencies:
> "Claude Code Task tool must support custom `subagent_type` values (confirmed: it does)"

The tasks assume `subagent_type="org:frontend-dev"` will work. But the parenthetical "(confirmed: it does)" has no citation or test. If the Task tool only accepts a fixed set of values (`"general-purpose"`, `"Explore"`, etc.), the entire plan fails.

**Suggested fix:** Task 1.1 or a pre-flight task should include an explicit verification step: spawn a test agent with a custom `subagent_type` value and confirm it works. This takes 30 seconds and prevents discovering the fundamental assumption is wrong after implementing 11 files of changes.

---

## MISSING

### MISSING 1: No task for the example config file

Task 1.1 says:
> "add a sibling file `get-shit-done/templates/config-agents-example.json` containing a complete example"

But there is no task that creates this file. Task 1.1's completion criteria only mention `templates/config.json`. Either remove the example file from the task description or add it to completion criteria.

### MISSING 2: No rollback plan

If the Task tool does NOT support custom `subagent_type` values (Concern 7), or if agent routing causes unexpected failures in production use, there is no documented way to revert. Since all changes are to markdown prompt files (not compiled code), "revert" means `git revert`, but the plan does not establish a single commit boundary or feature branch strategy.

**Suggested fix:** Add a note: "All changes should be implemented on a feature branch. If agent routing causes issues, the branch can be reverted as a unit."

### MISSING 3: No integration test strategy

The tasks have per-task verification (grep for strings, parse JSON), but there is no end-to-end test. Nobody verifies that "configure agents in config.json, run execute-phase, and agents actually get routed correctly." The closest thing is the success criteria in the plan, which are observational, not executable.

**Suggested fix:** Add a final validation task (Task 7.1) that describes a manual smoke test:
1. Create a test project with `agents.execution: ["general-purpose"]` in config
2. Create a minimal phase with one plan
3. Run `/gsd:execute-phase` and verify the Task call uses the configured agent
4. Remove agents key, re-run, verify fallback to "general-purpose"

### MISSING 4: The continuation-prompt.md routing gap

When a checkpoint plan pauses and resumes, the orchestrator spawns a fresh agent using `continuation-prompt.md`. Task 2.2 updates the template's Usage section to show `resolved_agent`. But nothing in Task 4.1 or 4.2 (execute-phase) explicitly addresses that the continuation spawn must also use the resolved agent, not just the initial spawn.

Look at `workflows/execute-phase.md` line 228-232:
```python
Task(
    prompt=filled_continuation_template,
    subagent_type="general-purpose"
)
```

This is the continuation spawn -- it is one of the 2 hardcoded references in the execute-phase workflow. Task 4.2 Change B says "Replace both hardcoded references (lines 191 and 230)." So it IS covered. But the task does not explicitly call out that the continuation agent must use the SAME resolved agent as the original spawn. If the resolved agent was `"org:frontend-dev"` for the initial spawn, the continuation must also be `"org:frontend-dev"`. The task should make this explicit to prevent an implementer from accidentally re-resolving to a different agent.

### MISSING 5: No mention of what happens with monorepo sub-projects

The task context says "Monorepo support: Each sub-project has its own config." But neither the plan nor the tasks mention how the orchestrator discovers which `config.json` to read when there are multiple `.planning/config.json` files in a monorepo. Is it always relative to CWD? What if the orchestrator runs from the monorepo root but the phase belongs to a sub-project?

The current behavior (read `.planning/config.json` relative to CWD) likely handles this correctly because GSD already scopes to the current directory. But this assumption should be stated explicitly, not left implicit.

---

## VERDICT

The plan is solid for Groups 1-5 (the core routing work). The hardcoded reference inventory is accurate, the file coverage is complete, and the backwards compatibility design is sound.

**Recommended changes before execution:**

1. **Remove Group 6 (planning delegation)** -- it is a separate feature, not a routing change. Ship it independently.
2. **Drop domain-matching logic** -- simplify to "one execution agent = use it; multiple = require explicit frontmatter or user override." No fuzzy matching.
3. **Flatten the config schema** -- remove `post_phase.external` (not an agent) and consider a flatter structure.
4. **Add a pre-flight verification** -- confirm Task tool accepts custom `subagent_type` values before implementing 11 files.
5. **Add an integration smoke test** as a final task.

With these changes, the plan goes from "ambitious and somewhat risky" to "focused and deliverable."
