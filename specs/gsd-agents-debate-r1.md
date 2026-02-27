# GSD Agent Routing -- Debate Round 1

Respondent: Strategic Planner
Date: 2026-02-26
Responding to: `specs/gsd-agents-review.md` by Devils Advocate Developer

---

## CONCERNS

### CONCERN 1: The "domain matching" logic is under-specified and risky

**Response:** CONCEDE

**Reasoning:** The reviewer is correct on all points. Domain matching ("match domain keyword against agent names") is vague, fragile, and introduces an entire class of bugs for marginal value. Agent name `org:react-specialist` does not substring-match domain `frontend`. The word "clear" in "if clear match" is undefined. The plan already has the `agents:` frontmatter field (Task 1.2) which is the proper mechanism for per-plan agent selection. If a user configures multiple execution agents, they should explicitly map them per-plan via frontmatter or user override, not rely on fuzzy heuristics.

**Changes made:**
- **Plan:** Rewrote "Agent Selection Logic" section to be deterministic: one agent = use it; multiple = require frontmatter/user override, else use first in list. Removed all domain matching language. Updated Priority 3 in the resolution chain.
- **Tasks:** Updated Task 4.1 Change B (`agent_routing` section) to remove domain matching and use deterministic selection. Updated Task 4.2 Change A (`resolve_agents` step) to remove domain matching, replace with deterministic pool selection. Updated Task 5.1 Change A similarly.

---

### CONCERN 2: Task 5.1 makes a false claim about execute-plan.md

**Response:** CONCEDE

**Reasoning:** Verified that `execute-plan.md` line 31 already contains `@.planning/config.json (if exists)`. The original task description's phrasing ("Context section references STATE.md and config.json but not for agent routing") is technically accurate but could mislead an implementer into thinking a context section change is needed. Making it explicit saves time.

**Changes made:**
- **Tasks:** Updated Task 5.1 current state description to: "Context section already includes `@.planning/config.json (if exists)` at line 31 -- no change needed to the context section." Updated Change A to include explicit note: "config.json is already in the context section (line 31). No change needed to context."

---

### CONCERN 3: Task 5.2 line references may be fragile

**Response:** CONCEDE

**Reasoning:** Line numbers alone are insufficient for a 394-line file with complex structure. Surrounding context makes replacements unambiguous even if line numbers shift.

**Changes made:**
- **Tasks:** Updated Task 5.2 to describe both replacement points with surrounding context:
  1. "In the 'For fully autonomous plans' subsection of segment_routing... under the '5. Implementation:' heading"
  2. "In the 'If routing = Subagent' block of the segment loop... under '2. For each segment in order:', under 'B. If routing = Subagent:' subheading"

---

### CONCERN 4: The config schema has unnecessary structural complexity

**Response:** CONCEDE

**Reasoning:** Four different shapes (array, nested object, string, mixed nested object with shell commands) in one config section is over-engineered. The reviewer correctly identifies that shell commands in an `agents` config key is a category error. With Group 6 (planning delegation) removed (see Concern 5), the `planning` nested object is no longer needed either. The flattened schema has two shapes: an array (`execution`) and a string (`verification`), both agent names.

**Changes made:**
- **Plan:** Updated methodology to show the flat schema. Removed post-phase hooks from verification flow. Added "Post-phase external tool hooks" to Out of Scope.
- **Tasks:** Updated Task 1.1 schema to flat structure with only `execution` (array) and `verification` (string). Added schema rationale note. Updated completion criteria to include `config-agents-example.json`. Removed `post_phase_hooks` from Tasks 4.1 and 4.2. Updated Task 3.1 to remove planning agent setup question.

---

### CONCERN 5: Planning delegation (Group 6) is a scope expansion disguised as a feature

**Response:** CONCEDE

**Reasoning:** The reviewer is right, and this is the most important correction. I verified: `plan-phase.md` has ZERO hardcoded `"general-purpose"` references. Group 6 is not replacing existing hardcoded references -- it is adding an entirely new subagent spawning capability that does not exist today. This is architecturally different from Groups 1-5 which are all find-and-replace operations on existing code paths. The reviewer's specific sub-points are all valid:
- No prompt template exists for the planner agent (every other agent spawn in GSD uses a template)
- The reviewer iteration loop adds control flow complexity to an orchestrator that currently has none
- Context budget concerns are real -- if context gathering approaches 15%, delegation value is marginal

This feature deserves its own plan, its own template(s), and its own testing.

**Changes made:**
- **Plan:** Removed planning delegation from: Overview, Objectives, Scope (added to Out of Scope), Methodology (removed Planning Delegation Flow entirely), Milestones, Risk Assessment, Success Criteria, Files Changed table (removed plan-phase.md row, count goes from 11 to 10).
- **Tasks:** Replaced Group 6 (planning delegation) with Group 6 (integration validation / smoke test). Added note in Notes section explaining why planning delegation was removed and that it should be a separate plan.

---

### CONCERN 6: Verification wave prompt has no error handling for missing build/test commands

**Response:** CONCEDE

**Reasoning:** The verify-prompt instructs the agent to "Run the project's build/compile command" but provides no mechanism to discover what that command is. Adding `@.planning/PROJECT.md` to the verify-prompt context section is simple, effective, and avoids config expansion. The verification agent can infer build/test/lint commands from the project description and tech stack documented in PROJECT.md.

**Changes made:**
- **Tasks:** Updated Task 2.3 verify-prompt template to add `Project: @.planning/PROJECT.md (for discovering build/test/lint commands)` to the `<context>` section.

---

### CONCERN 7: The plan claims Task tool accepts custom subagent_type values but provides no evidence

**Response:** CONCEDE

**Reasoning:** The parenthetical "(confirmed: it does)" had no citation or test. This is the fundamental load-bearing assumption of the entire plan. If it fails, every other task is wasted work. A 30-second pre-flight verification is trivially cheap insurance.

**Changes made:**
- **Plan:** Changed "(confirmed: it does)" to "(must be verified in pre-flight task before implementation begins)". Added "Task tool rejects custom subagent_type values" to Risk Assessment with Critical impact. Added pre-flight milestone (Milestone 0). Updated Success Criteria.
- **Tasks:** Added Group 0 (Pre-flight Verification) with Task 0.1 as the first task. Updated Dependencies to make Group 0 a hard gate before any other work. Updated execution order to 0 > 1 > 2 > 3 > 4 > 5 > 6.

---

## MISSING

### MISSING 1: No task for the example config file

**Response:** CONCEDE

**Reasoning:** Task 1.1 description mentions creating `config-agents-example.json` but completion criteria only checked `config.json`. The example file was planned but not tracked.

**Changes made:**
- **Tasks:** Added `config-agents-example.json` to Task 1.1 completion criteria and tests. The file must exist, be valid JSON, and contain `execution` and `verification` examples.

---

### MISSING 2: No rollback plan

**Response:** CONCEDE

**Reasoning:** All changes are to markdown prompt files, so `git revert` works mechanically. But establishing a feature branch strategy upfront makes rollback a one-step operation and prevents partial reversions.

**Changes made:**
- **Plan:** Added "Implementation strategy: All changes should be implemented on a feature branch. If agent routing causes issues, the branch can be reverted as a unit." to the Overview section.
- **Tasks:** Added matching implementation strategy note to the tasks file header.

---

### MISSING 3: No integration test strategy

**Response:** CONCEDE

**Reasoning:** Per-task tests (grep for strings, parse JSON) verify individual file changes but do not verify the routing chain works end-to-end. A manual smoke test with 5 scenarios covers the critical paths: no agents, single agent, frontmatter override, verification wave, and missing agents key.

**Changes made:**
- **Tasks:** Replaced Group 6 (removed planning delegation) with Group 6 (Integration Validation) containing Task 6.1: a 5-scenario end-to-end smoke test. Uses `"general-purpose"` as the test agent name to avoid dependency on custom agent definitions.
- **Plan:** Added integration validation milestone. Added smoke test to success criteria.

---

### MISSING 4: The continuation-prompt.md routing gap

**Response:** CONCEDE

**Reasoning:** Task 4.2 Change B does cover both hardcoded references (lines 191 and 230), but it did not explicitly state that the continuation agent must use the SAME resolved agent as the original spawn. An implementer could accidentally re-resolve to a different agent if config or frontmatter changed between spawn and continuation.

**Changes made:**
- **Tasks:** Updated Task 4.2 Change A (resolve_agents step) to include explicit note: "IMPORTANT: The resolved agent MUST be used for both initial spawn AND any continuation spawns for the same plan." Updated Change B to describe both replacement locations with surrounding context and explicitly state: "the continuation MUST use the same agent as the initial spawn, not re-resolve." Updated completion criteria to verify this.

---

### MISSING 5: No mention of what happens with monorepo sub-projects

**Response:** CONCEDE

**Reasoning:** CWD-relative config resolution is the correct behavior since GSD already scopes to CWD. But implicit assumptions should be stated explicitly to prevent confusion.

**Changes made:**
- **Plan:** Added "Config Resolution Note" subsection in Methodology explaining CWD-relative resolution and monorepo behavior.
- **Tasks:** Added note in Notes section: "Config resolution is always CWD-relative. In monorepo setups, GSD already scopes to CWD, so `.planning/config.json` always resolves to the correct sub-project's config."

---

## SUMMARY OF ALL CHANGES

**Concessions: 12 of 12** (7 concerns + 5 missing items)

The reviewer's feedback was uniformly correct. The plan went from "ambitious and somewhat risky" to "focused and deliverable" by:

1. **Removing Group 6 (planning delegation)** -- the single biggest simplification. This was scope creep masquerading as a feature. plan-phase.md has zero hardcoded "general-purpose" references; adding new subagent spawning is a separate feature.

2. **Dropping domain matching** -- replaced with deterministic selection. One agent = use it. Multiple = require explicit specification. No guessing.

3. **Flattening config schema** -- from 4 shapes (array, nested object, string, mixed object with shell commands) to 2 shapes (array and string), both agent names. Removed shell commands from agents config (category error).

4. **Adding pre-flight verification** -- 30 seconds of testing to validate the fundamental assumption before 10+ file changes.

5. **Adding integration smoke test** -- end-to-end validation that the routing chain actually works.

6. **Explicit about implicit assumptions** -- CWD-relative config, continuation uses same agent, config.json already in context, line locations with surrounding context.

**Net effect on scope:** 11 files changed reduced to 10. 6 groups reduced to 7 (but Group 0 is a single 30-second test, Group 6 is validation, and the removed Group 6 was the most complex task in the plan). Total implementation complexity decreased significantly.
