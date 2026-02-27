# Verification Prompt Template

Template for spawning verification agents after execution waves. Used by execute-phase orchestrator when `agents.verification` is configured in config.json.

---

## Template

```xml
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
