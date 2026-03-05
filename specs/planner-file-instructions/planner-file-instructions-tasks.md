# Planner Output Contract Extraction Implementation

Extract the planner's output contract (file paths, ROADMAP update rules, git commit convention) into a shared reference file. Custom planning agents receive this contract via prompt injection in plan-phase.md. The built-in gsd-planner.md is NOT modified (Alternative B).

## Completed Tasks

[Empty]

## In Progress Tasks

- [ ] Phase 1: Output Contract for Custom Planner Agents
  - [ ] Task 1: Create `get-shit-done/references/planner-output-contract.md`
    - Source material:
      - `/mnt/3b92ea25-2e45-41c8-97d3-58aa8141755e/Videos/Projects/NIIT/gsd-agents/agents/gsd-planner.md` lines 1189-1240 -- the 3 initial-mode output steps to extract (`write_phase_prompt`, `update_roadmap`, `git_commit`)
      - `/mnt/3b92ea25-2e45-41c8-97d3-58aa8141755e/Videos/Projects/NIIT/gsd-agents/agents/gsd-planner.md` lines 954-963 -- the revision-mode git commit step (different commit format: `fix($PHASE)` vs `docs($PHASE)`)
      - `/mnt/3b92ea25-2e45-41c8-97d3-58aa8141755e/Videos/Projects/NIIT/gsd-agents/get-shit-done/references/git-integration.md` -- structural pattern for reference files (uses XML-style sections like `<commit_formats>`, `<anti_patterns>`)
      - `/mnt/3b92ea25-2e45-41c8-97d3-58aa8141755e/Videos/Projects/NIIT/gsd-agents/get-shit-done/references/planning-config.md` -- structural pattern and the `COMMIT_PLANNING_DOCS` config check snippet
      - `/mnt/3b92ea25-2e45-41c8-97d3-58aa8141755e/Videos/Projects/NIIT/gsd-agents/get-shit-done/templates/phase-prompt.md` -- the PLAN.md format template (referenced by pointer, NOT duplicated)
    - What to create: A reference file with 3 sections, using XML-style structural tags consistent with other files in the `references/` directory. **Line budget: 45 lines maximum.** Content must be dense and operational, not explanatory.
      - **Preamble (2-3 lines):** State this is the output contract for custom planner agents. Note that gsd-planner has its own inline steps -- this file is for custom agents only. Note the intentional duplication.
      - **Section 1: `<file_output>` -- Write PLAN.md files**
        - File path convention: `.planning/phases/XX-name/{phase}-{NN}-PLAN.md`
        - Pointer to the full template: `@~/.claude/get-shit-done/templates/phase-prompt.md`
        - Required frontmatter fields listed (phase, plan, type, wave, depends_on, files_modified, autonomous, must_haves) -- field names only, NOT full descriptions
      - **Section 2: `<update_roadmap>` -- Update ROADMAP.md after planning**
        - Extract verbatim from gsd-planner.md lines 1197-1222 but compress to essentials
        - Must include: finding phase entry, updating Goal placeholder (only if placeholder), updating Plans count (always), updating Plan list checkboxes (always)
        - Must preserve the exact placeholder strings: `[To be planned]`, `[Urgent work - to be planned]`, `(created by /gsd:plan-phase)`
        - Must preserve the exact checkbox format: `- [ ] {phase}-01-PLAN.md -- {brief objective}`
      - **Section 3: `<git_commit>` -- Commit planning artifacts**
        - Include the `COMMIT_PLANNING_DOCS` config check bash snippet
        - **BOTH commit formats, keyed by mode:**
          - Initial mode: `docs($PHASE): create phase plan` (from gsd-planner.md line 1233)
          - Revision mode: `fix($PHASE): revise plans based on checker feedback` (from gsd-planner.md line 962)
        - Explicit instruction: "If mode is revision, use fix() format. Otherwise use docs() format."
        - If COMMIT_PLANNING_DOCS=false: skip git, log message
    - Completion Criteria:
      - File exists at `get-shit-done/references/planner-output-contract.md`
      - Contains all 3 sections (file_output, update_roadmap, git_commit)
      - References `phase-prompt.md` via `@` syntax (pointer, not duplication)
      - Does NOT duplicate the PLAN.md template content from phase-prompt.md
      - Contains the exact placeholder strings from gsd-planner.md
      - Contains BOTH commit formats: `docs($PHASE)` for initial AND `fix($PHASE)` for revision
      - Contains the COMMIT_PLANNING_DOCS config check snippet
      - File is 45 lines or fewer (`wc -l` check)
      - Follows XML-style section tags consistent with other reference files
    - Tests:
      - `test -f get-shit-done/references/planner-output-contract.md` exits 0
      - `grep -c "phase-prompt.md" get-shit-done/references/planner-output-contract.md` returns >= 1
      - `grep -c "<objective>" get-shit-done/references/planner-output-contract.md` returns 0 (template tag NOT reproduced)
      - `grep -c "To be planned" get-shit-done/references/planner-output-contract.md` returns >= 1
      - `grep -c 'docs(\$PHASE)' get-shit-done/references/planner-output-contract.md` returns >= 1
      - `grep -c 'fix(\$PHASE)' get-shit-done/references/planner-output-contract.md` returns >= 1
      - `grep -c "COMMIT_PLANNING_DOCS" get-shit-done/references/planner-output-contract.md` returns >= 1
      - `grep -c "<file_output>" get-shit-done/references/planner-output-contract.md` returns 1
      - `wc -l < get-shit-done/references/planner-output-contract.md` returns <= 45

  - [ ] Task 2: Modify `commands/gsd/plan-phase.md` to inject output contract for custom agents
    - Source material:
      - `/mnt/3b92ea25-2e45-41c8-97d3-58aa8141755e/Videos/Projects/NIIT/gsd-agents/commands/gsd/plan-phase.md` lines 376-393 (step 7 "Read Context Files" -- where context files are read and stored)
      - `/mnt/3b92ea25-2e45-41c8-97d3-58aa8141755e/Videos/Projects/NIIT/gsd-agents/commands/gsd/plan-phase.md` lines 463-484 (step 8 custom vs built-in branching for initial planning)
      - `/mnt/3b92ea25-2e45-41c8-97d3-58aa8141755e/Videos/Projects/NIIT/gsd-agents/commands/gsd/plan-phase.md` lines 640-657 (step 12 custom vs built-in branching for revision loop)
    - What to change (3 adjacent locations in the same file):
      - **Step 7**: Add `OUTPUT_CONTRACT=$(cat ~/.claude/get-shit-done/references/planner-output-contract.md 2>/dev/null)` alongside the other context file reads. Consistent with how STATE_CONTENT, ROADMAP_CONTENT, etc. are read.
      - **Step 8 custom branch** (line 471): Change `prompt=filled_prompt + "\n\n" + planner_return_protocol` to `prompt=filled_prompt + "\n\n" + output_contract + "\n\n" + planner_return_protocol`. The output contract comes BEFORE return_protocol because return_protocol defines signal headers that must be the last thing the agent sees.
      - **Step 12 custom branch** (line 644): Same pattern -- change `prompt=revision_prompt + "\n\n" + planner_return_protocol` to `prompt=revision_prompt + "\n\n" + output_contract + "\n\n" + planner_return_protocol`.
      - **Built-in branches in steps 8 and 12 are NOT changed.** gsd-planner already has its own inline steps.
    - Completion Criteria:
      - Step 7 reads the output contract file into OUTPUT_CONTRACT variable
      - Step 8 custom agent Task() call includes output_contract in prompt concatenation
      - Step 12 custom agent Task() call includes output_contract in prompt concatenation
      - Both built-in (fallback) Task() calls are unchanged (no output_contract)
      - output_contract appears BEFORE planner_return_protocol in concatenation order
      - Total `output_contract` references in plan-phase.md >= 3 (1 read + 2 uses)
      - All 13 process steps still present and intact
      - Agent routing section unchanged
      - Return protocol blocks unchanged
    - Tests:
      - `grep -c 'OUTPUT_CONTRACT\|output_contract' commands/gsd/plan-phase.md` returns >= 3
      - `grep -c 'planner-output-contract.md' commands/gsd/plan-phase.md` returns >= 1
      - `grep -c '## [0-9]' commands/gsd/plan-phase.md` returns >= 13 (all steps present)
      - `grep -c '<agent_routing>' commands/gsd/plan-phase.md` returns 1 (unchanged)
      - Verify built-in Task() blocks (subagent_type="gsd-planner") do NOT include output_contract -- manual inspection of the 2 built-in blocks in steps 8 and 12
      - Verify output_contract appears before planner_return_protocol in the custom branch prompt= lines

  - [ ] Task 3: Integration smoke test
    - Source material: N/A (this is a verification task, not a creation task)
    - What to verify: End-to-end correctness of the contract injection path. This is NOT a standalone task in the traditional sense -- it is the final validation that Tasks 1 and 2 produced correct, compatible results.
    - Steps:
      1. Verify the contract file can be read by the cat command used in plan-phase.md step 7: `cat ~/.claude/get-shit-done/references/planner-output-contract.md` succeeds and outputs content
      2. Verify gsd-planner.md is UNCHANGED: `git diff agents/gsd-planner.md` produces no output
      3. Verify the contract contains both commit formats: grep for `docs($PHASE)` AND `fix($PHASE)`
      4. Verify the contract references phase-prompt.md but does NOT reproduce its content
      5. Verify plan-phase.md injects contract only in custom agent branches (not built-in)
    - Completion Criteria: All 5 verification steps pass. No regressions in gsd-planner.md. The custom agent prompt injection path is complete and correct for both initial and revision modes.
    - Tests:
      - `cat ~/.claude/get-shit-done/references/planner-output-contract.md | wc -l` returns > 0 and <= 45
      - `git diff agents/gsd-planner.md | wc -l` returns 0 (untouched)
      - `grep -c 'docs(\$PHASE)' get-shit-done/references/planner-output-contract.md` returns >= 1
      - `grep -c 'fix(\$PHASE)' get-shit-done/references/planner-output-contract.md` returns >= 1
      - `grep -c '<objective>' get-shit-done/references/planner-output-contract.md` returns 0
      - Manual: read the custom branch in step 8 and step 12 of plan-phase.md, confirm output_contract is present and in correct order

## Future Tasks

[None -- this is a single-phase plan]

## Dependencies

- The `<agent_routing>` section in `plan-phase.md` (from the planning-agents spec) must already be in place. Defines custom vs built-in branching. **Status: Already merged** (commit cc1e965).
- `get-shit-done/templates/phase-prompt.md` must exist at its current path. The output contract references it. **Status: Exists** (568 lines, unchanged).
- `get-shit-done/references/planning-config.md` must exist. Referenced for `commit_docs` config details. **Status: Exists**.

## Notes

- **Intentional duplication:** The output contract duplicates procedural knowledge from gsd-planner.md lines 1189-1240 and 954-963. This is accepted under Alternative B -- modifying gsd-planner.md for refactoring purity creates regression risk that outweighs the duplication cost. Both files are in the same repo and under our control.

- **Prescriptive by design:** The contract is intentionally prescriptive about output format. Custom agents swap planning METHODOLOGY (how to think), not output FORMAT (where files go). The orchestrator and downstream consumers depend on specific file naming, ROADMAP.md structure, and commit message conventions. This is an interface contract.

- **Prompt size impact:** The contract adds ~45 lines to custom agent prompts, bringing total injected content from ~97 lines to ~142 lines. This is acceptable for operational instructions that the agent must follow to produce correct output.
