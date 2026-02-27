# GSD (Get Shit Done) — Technical Overview

> A Claude Code plugin that orchestrates AI-driven software development through slash commands, subagents, workflows, and templates. All logic lives in markdown files — no runtime code beyond the installer and two hooks.

**Last Updated:** 2026-02-25

---

## Table of Contents

- [1. What GSD Is](#1-what-gsd-is)
- [2. Plugin Directory Structure](#2-plugin-directory-structure)
- [3. The Four Layers](#3-the-four-layers)
- [4. Command Anatomy](#4-command-anatomy)
- [5. Project Lifecycle](#5-project-lifecycle)
- [6. The Phase Cycle](#6-the-phase-cycle)
- [7. Agent System](#7-agent-system)
- [8. Model Profile System](#8-model-profile-system)
- [9. Configuration](#9-configuration)
- [10. The .planning/ Directory](#10-the-planning-directory)
- [11. Data Flow Between Agents](#11-data-flow-between-agents)
- [12. Templates](#12-templates)
- [13. References](#13-references)
- [14. Hooks](#14-hooks)
- [15. Key Design Principles](#15-key-design-principles)
- [16. Complete Command Reference](#16-complete-command-reference)

---

## 1. What GSD Is

GSD is a **Claude Code plugin** — a collection of markdown files that extend Claude Code with slash commands, subagents, workflows, templates, references, and hooks. There is no runtime code beyond the installer (`bin/install.js`) and two hooks (`hooks/`).

It is distributed as an npm package (`get-shit-done-cc`) and installed via `npx get-shit-done-cc`. The installer copies files into `~/.claude/`:
- `commands/gsd/` → `~/.claude/commands/gsd/`
- `agents/` → `~/.claude/agents/`
- `get-shit-done/` → `~/.claude/get-shit-done/` (workflows, templates, references)

---

## 2. Plugin Directory Structure

```
get-shit-done/
├── package.json              # NPM package metadata (get-shit-done-cc)
├── bin/install.js            # Installer script
├── commands/gsd/             # 27 slash commands (user-invocable)
├── agents/                   # 11 subagent definitions
├── hooks/                    # 2 JS hooks (statusline, update checker)
├── get-shit-done/            # Plugin's installed content
│   ├── workflows/            # 12 workflow definitions (internal logic)
│   ├── templates/            # 31 file templates
│   │   ├── codebase/         # 7 codebase analysis templates
│   │   └── research-project/ # 5 project research templates
│   └── references/           # 9 reference documents
├── scripts/build-hooks.js    # Hook build script
└── assets/                   # Logo/branding
```

---

## 3. The Four Layers

GSD has a strict layered architecture. Each layer has a distinct role:

```
┌─────────────────────────────────────────────────────────────┐
│  COMMANDS  (commands/gsd/*.md)                              │
│  User-invocable via /gsd:<name>                             │
│  Parse args, load context, orchestrate agents               │
│  27 commands                                                │
├─────────────────────────────────────────────────────────────┤
│  WORKFLOWS  (get-shit-done/workflows/*.md)                  │
│  Internal logic loaded by commands via @reference           │
│  NOT user-invocable — loaded as execution context           │
│  12 workflows                                               │
├─────────────────────────────────────────────────────────────┤
│  AGENTS  (agents/gsd-*.md)                                  │
│  Subagents spawned via Task tool                            │
│  Each gets fresh 200k context window                        │
│  11 agents with restricted tool access                      │
├─────────────────────────────────────────────────────────────┤
│  TEMPLATES  (get-shit-done/templates/*.md)                  │
│  File structure patterns for output documents               │
│  Loaded by agents/workflows when creating files             │
│  31 templates                                               │
├─────────────────────────────────────────────────────────────┤
│  REFERENCES  (get-shit-done/references/*.md)                │
│  Shared knowledge (verification patterns, git rules, etc.)  │
│  9 reference docs                                           │
└─────────────────────────────────────────────────────────────┘
```

### How Layers Connect

- **Commands** load **workflows** and **templates** via `@` references in `<execution_context>`
- **Commands** spawn **agents** via the `Task` tool with inlined file contents
- **Agents** use **templates** to structure their output files
- **Agents** follow **references** for shared patterns (verification, git, etc.)
- **Workflows** contain the procedural logic that commands follow step-by-step

---

## 4. Command Anatomy

Every command is a markdown file with YAML frontmatter + XML-structured body:

### Frontmatter

```yaml
---
name: gsd:<command-name>       # Slash command identifier
description: <one-liner>       # Shows in /gsd:help listing
argument-hint: "<args>"        # Usage hint shown to user
allowed-tools:                 # Tool whitelist for this command
  - Read
  - Write
  - Task                       # Required if spawning agents
  - AskUserQuestion            # Required if interactive
---
```

### Body Structure

```xml
<objective>What this command achieves</objective>

<execution_context>
@~/.claude/get-shit-done/workflows/some-workflow.md   <!-- loads workflow -->
@~/.claude/get-shit-done/templates/some-template.md   <!-- loads template -->
</execution_context>

<context>
@.planning/STATE.md            <!-- loads project files -->
@.planning/ROADMAP.md
</context>

<process>
  <step name="step_name">
    Step instructions...
  </step>
  <step name="next_step">
    More instructions...
  </step>
</process>

<anti_patterns>
  What NOT to do
</anti_patterns>

<success_criteria>
  - [ ] Checklist item 1
  - [ ] Checklist item 2
</success_criteria>
```

The `@` syntax tells Claude to load file contents inline. The `<execution_context>` loads workflows/templates into Claude's context so it can follow them.

---

## 5. Project Lifecycle

```
┌──────────────────────────────────────────────────────────────────────────┐
│                          GSD PROJECT LIFECYCLE                           │
│                                                                          │
│  ┌─────────────┐   ┌───────────────┐   ┌──────────────┐                │
│  │ NEW PROJECT  │──▶│  MAP CODEBASE │──▶│ REQUIREMENTS │                │
│  │  /gsd:new-   │   │  /gsd:map-    │   │  (inline in   │                │
│  │   project    │   │   codebase    │   │  new-project) │                │
│  └──────┬───────┘   └───────────────┘   └──────┬───────┘                │
│         │           (brownfield only)           │                        │
│         ▼                                       ▼                        │
│  ┌──────────────────────────────────────────────────────┐               │
│  │              ROADMAP CREATION                         │               │
│  │  Spawns: gsd-project-researcher ×4 (parallel)        │               │
│  │        → gsd-research-synthesizer                     │               │
│  │        → gsd-roadmapper                               │               │
│  │  Creates: PROJECT.md, REQUIREMENTS.md, ROADMAP.md,    │               │
│  │           STATE.md, config.json, research/*            │               │
│  └──────────────────────────┬───────────────────────────┘               │
│                             ▼                                            │
│  ┌ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ PHASE CYCLE ─ ─ ─ ─ ─ ─ ─ ─ ─ ┐              │
│  │                                                        │              │
│  │  ┌─────────────┐  ┌──────────────┐  ┌──────────────┐ │              │
│  │  │   DISCUSS    │─▶│    PLAN      │─▶│   EXECUTE    │ │              │
│  │  │  /gsd:       │  │  /gsd:       │  │  /gsd:       │ │              │
│  │  │ discuss-phase│  │ plan-phase   │  │ execute-phase│ │              │
│  │  └─────────────┘  └──────────────┘  └──────┬───────┘ │              │
│  │        ▲                                    │         │              │
│  │        │           ┌──────────────┐         │         │              │
│  │        │           │   VERIFY     │◀────────┘         │              │
│  │        │           │  /gsd:       │                   │              │
│  │        │           │ verify-work  │                   │              │
│  │        │           └──────┬───────┘                   │              │
│  │        │                  │                            │              │
│  │        │    ┌─────────────┴─────────────┐             │              │
│  │        │    │ gaps?  ──▶ plan-phase     │             │              │
│  │        │    │            --gaps          │             │              │
│  │        │    │ pass?  ──▶ next phase ────┼──┐          │              │
│  │        │    └───────────────────────────┘  │          │              │
│  │        └───────────────────────────────────┘          │              │
│  └ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─┘              │
│                             │                                            │
│                    all phases done                                        │
│                             ▼                                            │
│  ┌──────────────────────────────────────────────────────┐               │
│  │           MILESTONE COMPLETION                        │               │
│  │  /gsd:audit-milestone → /gsd:complete-milestone       │               │
│  │  Archives to milestones/, tags git, cleans up          │               │
│  │  → /gsd:new-milestone for next cycle                   │               │
│  └──────────────────────────────────────────────────────┘               │
└──────────────────────────────────────────────────────────────────────────┘
```

### Lifecycle Stages

1. **`/gsd:new-project`** — 10-phase process: setup → brownfield detection → deep questioning → PROJECT.md → config.json → research (4 parallel researchers + synthesizer) → requirements → roadmap (roadmapper agent) → STATE.md → done
2. **Phase Cycle** — Repeats for each phase: discuss → plan → execute → verify
3. **`/gsd:complete-milestone`** — Archives milestone, tags git, offers `/gsd:new-milestone`

---

## 6. The Phase Cycle — Detailed Data Flow

### 6.1 DISCUSS → CONTEXT.md

```
/gsd:discuss-phase 3
        │
        ▼
┌─────────────────────────────────┐
│  COMMAND: discuss-phase.md      │
│  Loads: discuss-phase workflow  │
│  Loads: context.md template     │
│  Reads: ROADMAP.md, STATE.md   │
├─────────────────────────────────┤
│  1. Parse phase number          │
│  2. Analyze phase → gray areas  │
│  3. AskUserQuestion (multiSel)  │     User picks areas to discuss
│  4. Deep-dive each area (4 Q's) │     AskUserQuestion per question
│  5. Write CONTEXT.md            │
│  6. Git commit                  │
└────────────┬────────────────────┘
             │
             ▼
  .planning/phases/03-name/03-CONTEXT.md
  ┌──────────────────────────┐
  │ Phase Boundary            │  ← from ROADMAP.md (FIXED, not negotiable)
  │ Implementation Decisions  │
  │   - Locked decisions      │  ← downstream agents MUST follow these
  │   - Claude's Discretion   │  ← agent decides
  │ Specific Ideas            │
  │ Deferred Ideas            │  ← explicitly OUT of scope
  └──────────────────────────┘
```

**Key design**: The phase boundary from ROADMAP.md is FIXED. Discussion clarifies HOW to implement what's scoped, never adds new capabilities. Scope creep is captured in "Deferred Ideas" and redirected.

### 6.2 PLAN → PLAN.md files

```
/gsd:plan-phase 3
        │
        ▼
┌────────────────────────────────────────────────────────────────────┐
│  COMMAND: plan-phase.md                                            │
│  Reads: ROADMAP, STATE, CONTEXT.md, config.json                   │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  Step 1: Resolve model profile from config.json                    │
│                                                                    │
│  Step 2: Spawn gsd-phase-researcher (if config.workflow.research)  │
│          ├── Constrained by CONTEXT.md (locked decisions)          │
│          ├── Uses: Context7 first, WebFetch, WebSearch last        │
│          ├── Writes: RESEARCH.md directly                          │
│          └── Returns: COMPLETE / CHECKPOINT / INCONCLUSIVE         │
│                                                                    │
│  Step 3: Spawn gsd-planner                                         │
│          ├── Receives INLINED: STATE, ROADMAP, REQUIREMENTS,       │
│          │   CONTEXT.md, RESEARCH.md                               │
│          ├── Goal-backward methodology:                             │
│          │   Goal → Observable Truths → Artifacts → Wiring         │
│          ├── Breaks into tasks (2-3 per plan)                      │
│          ├── Assigns waves (parallel execution groups)             │
│          ├── Writes: PLAN.md files with rich frontmatter           │
│          └── Returns: COMPLETE / CHECKPOINT / INCONCLUSIVE         │
│                                                                    │
│  Step 4: Spawn gsd-plan-checker (if config.workflow.plan_check)    │
│          ├── 7 verification dimensions:                             │
│          │   1. Requirement coverage                                │
│          │   2. Task completeness (files/action/verify/done)       │
│          │   3. Dependency correctness                              │
│          │   4. Key links planned                                   │
│          │   5. Scope sanity (2-3 tasks/plan)                      │
│          │   6. Verification derivation                             │
│          │   7. Context compliance (honors CONTEXT.md)             │
│          └── Returns: PASSED / ISSUES FOUND                        │
│                                                                    │
│  Step 5: Revision loop (max 3)                                     │
│          planner revises → checker re-checks → repeat if needed    │
└────────────────────────────────────────────────────────────────────┘
```

**Output: PLAN.md files**

```yaml
---
phase: 3
plan: 1
type: feature
wave: 1                         # Parallel execution group
depends_on: []                  # Plans this depends on
files_modified: [src/auth.ts]
autonomous: true                # false = has checkpoints requiring user
user_setup: false               # true = needs USER-SETUP.md
must_haves:
  truths: ["User can log in"]   # Goal-backward observable truths
  artifacts: [src/auth.ts]      # Required files
  key_links: ["Form → API"]     # Required wiring
---
```

```xml
<objective>What this plan delivers</objective>
<context>Referenced files</context>
<tasks>
  <task id="1" type="auto" tdd="false">
    <files>src/auth.ts</files>
    <action>Create authentication middleware...</action>
    <verify>Run tests, check imports</verify>
    <done>Auth middleware exports and is imported</done>
  </task>
  <task id="2" type="checkpoint:human-verify">
    <files>src/login.tsx</files>
    <action>Build login form...</action>
    <verify>Visual check of form layout</verify>
    <done>Login form renders correctly</done>
  </task>
</tasks>
```

Task types:
- `auto` — fully autonomous, no user interaction
- `checkpoint:human-verify` (90% of checkpoints) — user confirms visually
- `checkpoint:decision` (9%) — user makes a choice
- `checkpoint:human-action` (1%) — user must do something Claude can't (browser auth, etc.)

### 6.3 EXECUTE → SUMMARY.md files

```
/gsd:execute-phase 3
        │
        ▼
┌──────────────────────────────────────────────────────────────────────┐
│  COMMAND: execute-phase.md                                           │
│  Loads: execute-phase.md workflow, ui-brand.md                       │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. Discover plans → extract wave numbers from frontmatter           │
│  2. Group by wave:                                                   │
│                                                                      │
│     Wave 1: [PLAN-01, PLAN-02]  ← parallel (no deps)               │
│     Wave 2: [PLAN-03]           ← sequential (depends on wave 1)   │
│                                                                      │
│  3. Execute waves:                                                   │
│                                                                      │
│     ┌─── Wave 1 ───────────────────────────────────┐               │
│     │                                               │               │
│     │  ┌──────────────┐    ┌──────────────┐        │               │
│     │  │ Task: spawn   │    │ Task: spawn   │        │               │
│     │  │ gsd-executor  │    │ gsd-executor  │        │               │
│     │  │ for PLAN-01   │    │ for PLAN-02   │        │               │
│     │  │ (parallel)    │    │ (parallel)    │        │               │
│     │  └──────┬───────┘    └──────┬───────┘        │               │
│     │         │                    │                 │               │
│     │         ▼                    ▼                 │               │
│     │  03-01-SUMMARY.md    03-02-SUMMARY.md         │               │
│     └───────────────────────────────────────────────┘               │
│                        │                                             │
│                        ▼                                             │
│     ┌─── Wave 2 ───────────────────────────────────┐               │
│     │  ┌──────────────┐                             │               │
│     │  │ Task: spawn   │                             │               │
│     │  │ gsd-executor  │                             │               │
│     │  │ for PLAN-03   │                             │               │
│     │  └──────┬───────┘                             │               │
│     │         ▼                                      │               │
│     │  03-03-SUMMARY.md                              │               │
│     └───────────────────────────────────────────────┘               │
│                                                                      │
│  4. Spawn gsd-verifier (if config.workflow.verifier)                 │
│     Goal-backward: Truths → Artifacts → Wiring                      │
│     Creates: VERIFICATION.md                                         │
│     Returns: passed / gaps_found / human_needed                      │
│                                                                      │
│  5. Update ROADMAP.md, STATE.md, REQUIREMENTS.md                     │
└──────────────────────────────────────────────────────────────────────┘
```

**What gsd-executor does per plan** (in a fresh 200k context window):

```
┌─ gsd-executor ──────────────────────────────────────────────┐
│                                                               │
│  For each task in PLAN.md:                                    │
│                                                               │
│  1. If type="auto":                                           │
│     ├── Execute <action> (write/edit code)                   │
│     ├── Run <verify> (tests, checks)                         │
│     ├── Git commit per task:                                  │
│     │   feat(03-01): create auth middleware                    │
│     │   (stage specific files, NEVER git add .)               │
│     └── Apply deviation rules:                                │
│         Rule 1: Auto-fix bugs (no ask)                        │
│         Rule 2: Auto-add missing critical (no ask)            │
│         Rule 3: Auto-fix blockers (no ask)                    │
│         Rule 4: Ask about architectural changes (STOP)        │
│                                                               │
│  2. If type="checkpoint:*":                                   │
│     └── STOP, return state to orchestrator, wait for user    │
│                                                               │
│  3. After all tasks: Write SUMMARY.md                         │
│     ├── YAML frontmatter: dependency_graph, tech_tracking,   │
│     │   file_tracking, decisions, metrics                     │
│     └── Body: accomplishments, commits, deviations, issues   │
└───────────────────────────────────────────────────────────────┘
```

### 6.4 VERIFY → UAT.md + gap closure

```
/gsd:verify-work 3
        │
        ▼
┌──────────────────────────────────────────────────────────────────┐
│  1. Extract testable deliverables from SUMMARY.md files          │
│  2. Create UAT.md with test list                                 │
│  3. Present tests ONE AT A TIME:                                 │
│     "Expected: User can log in with valid credentials"           │
│     "Type pass or describe what's wrong"                         │
│  4. User types "yes" = pass, anything else = issue               │
│  5. Severity inferred from description (never asked)             │
│                                                                  │
│  If issues found:                                                │
│  ┌────────────────────────────────────────────────────────┐     │
│  │  Spawn parallel gsd-debugger agents (one per gap)      │     │
│  │  Collect root causes                                    │     │
│  │  Spawn gsd-planner --gaps (creates fix plans)           │     │
│  │  Spawn gsd-plan-checker (verify fix plans)              │     │
│  │  Revision loop (max 3)                                  │     │
│  │  → Ready for /gsd:execute-phase --gaps-only             │     │
│  └────────────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────────────┘
```

---

## 7. Agent System

### 7.1 Agent Frontmatter Structure

```yaml
---
name: gsd-executor
description: Executes GSD plans with atomic commits...
tools: Read, Write, Edit, Bash, Grep, Glob    # Tool whitelist
color: yellow                                    # Status line color
---
```

The body is the agent's **system prompt** — detailed behavioral instructions.

### 7.2 All 11 Agents

| Agent | Tools | Spawned By | Writes | Returns |
|-------|-------|------------|--------|---------|
| `gsd-project-researcher` | Read, Write, Bash, Grep, Glob, WebSearch, WebFetch, Context7 | `new-project`, `new-milestone` (×4 parallel) | `research/STACK.md`, `FEATURES.md`, `ARCHITECTURE.md`, or `PITFALLS.md` | Structured result (does NOT commit) |
| `gsd-research-synthesizer` | Read, Write, Bash | `new-project`, `new-milestone` | `research/SUMMARY.md` | Summary + commits all 5 research files |
| `gsd-roadmapper` | Read, Write, Bash, Glob, Grep | `new-project`, `new-milestone` | `ROADMAP.md`, `STATE.md` | Summary with phase count |
| `gsd-codebase-mapper` | Read, Bash, Grep, Glob, Write | `map-codebase` (×4 parallel) | 1-2 files in `.planning/codebase/` per agent | Brief confirmation (does NOT commit) |
| `gsd-phase-researcher` | Read, Write, Bash, Grep, Glob, WebSearch, WebFetch, Context7 | `plan-phase`, `research-phase` | `RESEARCH.md` in phase dir | COMPLETE / CHECKPOINT / INCONCLUSIVE |
| `gsd-planner` | Read, Write, Bash, Glob, Grep, WebFetch, Context7 | `plan-phase`, `quick`, `verify-work` | `PLAN.md` files in phase dir | COMPLETE / CHECKPOINT / INCONCLUSIVE |
| `gsd-plan-checker` | Read, Bash, Glob, Grep | `plan-phase`, `verify-work` | Nothing (read-only analysis) | PASSED / ISSUES FOUND |
| `gsd-executor` | Read, Write, Edit, Bash, Grep, Glob | `execute-phase` (per plan) | Code files + `SUMMARY.md` | COMPLETE / CHECKPOINT / INCONCLUSIVE |
| `gsd-verifier` | Read, Bash, Grep, Glob | `execute-phase` | `VERIFICATION.md` | passed / gaps_found / human_needed |
| `gsd-integration-checker` | Read, Bash, Grep, Glob | `audit-milestone` | Nothing (returns report) | Wiring status + flow status |
| `gsd-debugger` | Read, Write, Edit, Bash, Grep, Glob, WebSearch | `debug`, `verify-work` | `.planning/debug/{slug}.md` | ROOT CAUSE FOUND / CHECKPOINT / INCONCLUSIVE |

### 7.3 Agent Return Protocol

Every agent uses one of three structured return types:

```
COMPLETE
- Summary of what was done
- File paths created/modified
- Key metrics

CHECKPOINT REACHED
- Completed: [task table]
- Current task: N
- Checkpoint type: human-verify | decision | human-action
- Details: "What the user needs to do"

INVESTIGATION INCONCLUSIVE
- What was checked
- What was eliminated
- Remaining hypotheses
```

### 7.4 Context Passing

Commands **inline file contents** when spawning agents because `@` references don't work across Task tool boundaries:

```
┌─────────────────────────────┐
│  COMMAND (orchestrator)     │
│                              │
│  Reads files:                │
│  - STATE.md                  │
│  - ROADMAP.md                │
│  - CONTEXT.md                │
│  - RESEARCH.md               │
│                              │
│  INLINES content into        │
│  Task prompt string          │
└──────────┬──────────────────┘
           │
           ▼
  ┌────────────────┐
  │ gsd-agent      │
  │ (fresh 200k    │
  │  context)      │
  │                │
  │ Receives full  │
  │ text of files  │
  │ in the prompt  │
  └────────────────┘
```

---

## 8. Model Profile System

Commands resolve agent models from `config.json`'s `model_profile` field:

| Agent | `quality` | `balanced` | `budget` |
|-------|-----------|------------|----------|
| `gsd-project-researcher` | opus | sonnet | haiku |
| `gsd-phase-researcher` | opus | sonnet | haiku |
| `gsd-research-synthesizer` | opus | sonnet | haiku |
| `gsd-planner` | opus | opus | sonnet |
| `gsd-plan-checker` | sonnet | sonnet | haiku |
| `gsd-executor` | opus | sonnet | sonnet |
| `gsd-verifier` | sonnet | sonnet | haiku |
| `gsd-codebase-mapper` | sonnet | haiku | haiku |
| `gsd-roadmapper` | opus | sonnet | sonnet |
| `gsd-debugger` | opus | sonnet | sonnet |
| `gsd-integration-checker` | sonnet | sonnet | haiku |

Set via `/gsd:set-profile <quality|balanced|budget>` or `/gsd:settings`.

**Design principle**: Writing agents (planner, executor) get the highest-quality models. Read-only agents (checker, verifier) can use cheaper models.

---

## 9. Configuration

Created during `/gsd:new-project` Phase 5 (Workflow Preferences). Stored at `.planning/config.json`:

```json
{
  "mode": "interactive",
  "depth": "standard",
  "model_profile": "balanced",
  "workflow": {
    "research": true,
    "plan_check": true,
    "verifier": true
  },
  "planning": {
    "commit_docs": true,
    "search_gitignored": false
  },
  "git": {
    "branching_strategy": "none"
  },
  "parallelization": {
    "enabled": true,
    "plan_level": true,
    "task_level": false,
    "max_concurrent_agents": 3,
    "min_plans_for_parallel": 2
  },
  "gates": {
    "confirm_project": true,
    "confirm_phases": true,
    "confirm_roadmap": true,
    "confirm_breakdown": true,
    "confirm_plan": true,
    "execute_next_plan": true,
    "issues_review": true,
    "confirm_transition": true
  },
  "safety": {
    "always_confirm_destructive": true,
    "always_confirm_external_services": true
  }
}
```

### Key Fields

| Field | Values | Effect |
|-------|--------|--------|
| `mode` | `interactive` / `yolo` | `yolo` auto-approves most confirmation gates |
| `depth` | `quick` / `standard` / `comprehensive` | Controls roadmap phase count (3-5 / 5-8 / 8-12) |
| `model_profile` | `quality` / `balanced` / `budget` | Model selection per agent (see table above) |
| `workflow.research` | `true` / `false` | Spawn researcher before planning |
| `workflow.plan_check` | `true` / `false` | Spawn plan-checker after planning |
| `workflow.verifier` | `true` / `false` | Spawn verifier after execution |
| `planning.commit_docs` | `true` / `false` | Git commit planning documents |
| `git.branching_strategy` | `none` / `phase` / `milestone` | Branch creation per phase or milestone |
| `safety.always_confirm_destructive` | `true` / `false` | Overrides yolo mode for destructive actions |

---

## 10. The .planning/ Directory

GSD creates and manages this directory tree in the user's project:

```
.planning/
├── PROJECT.md                    # Living project description
├── REQUIREMENTS.md               # Checkable requirements (REQ-IDs)
├── ROADMAP.md                    # Phase structure + progress
├── STATE.md                      # Current position (< 100 lines)
├── MILESTONES.md                 # Shipped milestone history
├── config.json                   # Workflow configuration
│
├── research/                     # Project-level research (new-project)
│   ├── STACK.md                  #   Recommended technologies
│   ├── FEATURES.md               #   Feature landscape
│   ├── ARCHITECTURE.md           #   System structure patterns
│   ├── PITFALLS.md               #   Common mistakes to avoid
│   └── SUMMARY.md                #   Synthesized executive summary
│
├── codebase/                     # Codebase analysis (map-codebase)
│   ├── STACK.md                  #   Technology foundation
│   ├── INTEGRATIONS.md           #   External service dependencies
│   ├── ARCHITECTURE.md           #   Conceptual code organization
│   ├── STRUCTURE.md              #   Physical file organization
│   ├── CONVENTIONS.md            #   Coding style and patterns
│   ├── TESTING.md                #   Test framework and patterns
│   └── CONCERNS.md               #   Known issues, tech debt
│
├── phases/
│   ├── 01-auth/
│   │   ├── 01-CONTEXT.md         # User decisions (from discuss-phase)
│   │   ├── 01-RESEARCH.md        # Technical research (from researcher)
│   │   ├── 01-01-PLAN.md         # Execution plan (from planner)
│   │   ├── 01-01-SUMMARY.md      # Completion record (from executor)
│   │   ├── 01-02-PLAN.md         # Second plan
│   │   ├── 01-02-SUMMARY.md      # Second completion record
│   │   ├── 01-VERIFICATION.md    # Goal verification (from verifier)
│   │   ├── 01-UAT.md             # User acceptance tests (from verify-work)
│   │   ├── 01-USER-SETUP.md      # Manual setup steps (if needed)
│   │   └── .continue-here.md     # Session handoff (from pause-work, temp)
│   ├── 02-profiles/
│   │   └── ...
│   └── 02.1-hotfix/              # Decimal = inserted urgent phase
│       └── ...
│
├── quick/                        # Quick tasks (outside roadmap)
│   ├── 001-fix-typo/
│   └── 002-add-env-var/
│
├── debug/                        # Debug sessions (persistent across resets)
│   └── auth-token-expired.md
│
├── todos/
│   ├── pending/                  # Captured ideas/tasks
│   │   └── 2026-02-25-add-caching.md
│   └── done/                     # Completed todos
│
└── milestones/                   # Archived milestones
    ├── v1.0-ROADMAP.md           # Frozen roadmap snapshot
    └── v1.0-REQUIREMENTS.md      # Frozen requirements snapshot
```

### Key Files

| File | Purpose | Size Constraint |
|------|---------|-----------------|
| `STATE.md` | Current position, decisions, blockers, session continuity | < 100 lines (digest, not archive) |
| `PROJECT.md` | Living project description, evolves at phase transitions | Evolved at each transition |
| `REQUIREMENTS.md` | Checkable requirements with REQ-IDs (e.g., AUTH-01) | Fresh per milestone |
| `ROADMAP.md` | Phase structure, progress tracking, success criteria | 100% requirement coverage |
| `config.json` | All workflow toggles and preferences | Created once, updated via settings |

---

## 11. Data Flow Between Agents

### 11.1 New Project — Research Pipeline

```
/gsd:new-project
    │
    ├── Spawns 4 parallel gsd-project-researcher agents:
    │   ├── Agent 1: focus=stack      → writes research/STACK.md
    │   ├── Agent 2: focus=features   → writes research/FEATURES.md
    │   ├── Agent 3: focus=architecture → writes research/ARCHITECTURE.md
    │   └── Agent 4: focus=pitfalls   → writes research/PITFALLS.md
    │
    ├── Waits for all 4 to complete
    │
    ├── Spawns gsd-research-synthesizer
    │   ├── Reads all 4 research files
    │   ├── Writes research/SUMMARY.md
    │   └── Commits all 5 files together
    │
    └── Spawns gsd-roadmapper
        ├── Reads SUMMARY.md + REQUIREMENTS.md
        ├── Maps every requirement to a phase (100% coverage)
        ├── Writes ROADMAP.md + STATE.md
        └── Returns for user approval loop
```

### 11.2 Plan Phase — Research → Plan → Check Pipeline

```
/gsd:plan-phase 3
    │
    ├── Spawns gsd-phase-researcher (if enabled)
    │   ├── Constrained by CONTEXT.md
    │   └── Writes RESEARCH.md
    │
    ├── Reads RESEARCH.md, CONTEXT.md, STATE.md, ROADMAP.md, REQUIREMENTS.md
    │
    ├── Spawns gsd-planner (with ALL files inlined)
    │   └── Writes PLAN.md files
    │
    ├── Spawns gsd-plan-checker
    │   └── Returns PASSED or ISSUES FOUND
    │
    └── If ISSUES FOUND: revision loop (max 3)
        ├── Spawn planner (revision mode)
        └── Spawn checker (re-check)
```

### 11.3 Execute Phase — Wave Execution Pipeline

```
/gsd:execute-phase 3
    │
    ├── Groups plans by wave number
    │
    ├── Wave 1: Spawn gsd-executor per autonomous plan (parallel)
    │   └── Each writes code + SUMMARY.md
    │
    ├── Wave 2: Spawn gsd-executor per plan (parallel within wave)
    │   └── Each writes code + SUMMARY.md
    │
    └── Spawn gsd-verifier (if enabled)
        └── Writes VERIFICATION.md
```

### 11.4 Verify Work — Diagnose → Plan → Check Pipeline

```
/gsd:verify-work 3
    │
    ├── Extracts tests from SUMMARY.md files
    ├── User tests one at a time
    │
    ├── If issues found:
    │   ├── Spawn parallel gsd-debugger agents (one per gap)
    │   │   └── Each writes .planning/debug/{slug}.md
    │   │
    │   ├── Spawn gsd-planner --gaps (with root causes from debuggers)
    │   │   └── Writes fix PLAN.md files
    │   │
    │   └── Spawn gsd-plan-checker (verify fix plans)
    │       └── Revision loop if needed (max 3)
    │
    └── Ready for /gsd:execute-phase --gaps-only
```

---

## 12. Templates

31 templates that define the structure of every file GSD creates:

### Core Project Templates

| Template | Creates | Used By |
|----------|---------|---------|
| `project.md` | `.planning/PROJECT.md` | `new-project`, `new-milestone` |
| `requirements.md` | `.planning/REQUIREMENTS.md` | `new-project`, `new-milestone` |
| `roadmap.md` | `.planning/ROADMAP.md` | `gsd-roadmapper` |
| `state.md` | `.planning/STATE.md` | `gsd-roadmapper`, multiple commands |
| `config.json` | `.planning/config.json` | `new-project` |
| `milestone.md` | `.planning/MILESTONES.md` entries | `complete-milestone` |
| `milestone-archive.md` | `.planning/milestones/v*-ROADMAP.md` | `complete-milestone` |

### Phase Templates

| Template | Creates | Used By |
|----------|---------|---------|
| `context.md` | `{phase}-CONTEXT.md` | `discuss-phase` |
| `research.md` | `{phase}-RESEARCH.md` | `gsd-phase-researcher` |
| `phase-prompt.md` | `{phase}-{plan}-PLAN.md` | `gsd-planner` |
| `summary.md` | `{phase}-{plan}-SUMMARY.md` | `gsd-executor` |
| `verification-report.md` | `{phase}-VERIFICATION.md` | `gsd-verifier` |
| `UAT.md` | `{phase}-UAT.md` | `verify-work` |
| `user-setup.md` | `{phase}-USER-SETUP.md` | `gsd-executor` |
| `discovery.md` | `DISCOVERY.md` | `plan-phase` (mandatory discovery) |
| `continue-here.md` | `.continue-here.md` | `pause-work` |
| `DEBUG.md` | `.planning/debug/{slug}.md` | `gsd-debugger` |

### Agent Prompt Templates

| Template | Purpose | Used By |
|----------|---------|---------|
| `planner-subagent-prompt.md` | Prompt for spawning `gsd-planner` | `plan-phase`, `verify-work` |
| `debug-subagent-prompt.md` | Prompt for spawning `gsd-debugger` | `debug`, `verify-work` |

### Codebase Analysis Templates (7 files in `codebase/`)

| Template | Creates | Focus |
|----------|---------|-------|
| `codebase/stack.md` | `STACK.md` | Languages, runtime, frameworks, key deps |
| `codebase/integrations.md` | `INTEGRATIONS.md` | External services, APIs, storage |
| `codebase/architecture.md` | `ARCHITECTURE.md` | Conceptual organization, data flow |
| `codebase/structure.md` | `STRUCTURE.md` | Physical file layout, "where to put X" |
| `codebase/conventions.md` | `CONVENTIONS.md` | Naming, style, patterns |
| `codebase/testing.md` | `TESTING.md` | Test framework, patterns, coverage |
| `codebase/concerns.md` | `CONCERNS.md` | Tech debt, bugs, security, fragile areas |

### Research Project Templates (5 files in `research-project/`)

| Template | Creates | Focus |
|----------|---------|-------|
| `research-project/STACK.md` | `research/STACK.md` | Recommended technologies |
| `research-project/FEATURES.md` | `research/FEATURES.md` | Feature landscape, MVP definition |
| `research-project/ARCHITECTURE.md` | `research/ARCHITECTURE.md` | System patterns, data flow |
| `research-project/PITFALLS.md` | `research/PITFALLS.md` | Common mistakes, recovery strategies |
| `research-project/SUMMARY.md` | `research/SUMMARY.md` | Executive summary, roadmap implications |

---

## 13. References

9 reference documents providing shared knowledge:

| Reference | Purpose |
|-----------|---------|
| `verification-patterns.md` | Patterns for verifying code (stub detection, wiring checks) |
| `continuation-format.md` | Standard "Next Up" formatting for command outputs |
| `planning-config.md` | How to read and use config.json fields |
| `model-profiles.md` | Model profile lookup tables |
| `git-integration.md` | Git commit conventions, branching, tagging |
| `checkpoints.md` | Checkpoint protocol (human-verify, decision, human-action) |
| `tdd.md` | TDD execution patterns (RED-GREEN-REFACTOR) |
| `questioning.md` | Adaptive questioning techniques for discuss-phase |
| `ui-brand.md` | ASCII box formatting, status line, visual conventions |

---

## 14. Hooks

Two JavaScript hooks:

### `gsd-statusline.js`

Updates the Claude Code status line with current GSD state by reading `.planning/STATE.md`. Shows phase number, plan status, and progress.

### `gsd-check-update.js`

Checks for newer GSD versions on npm and notifies the user when updates are available.

---

## 15. Key Design Principles

These principles are embedded throughout the codebase:

1. **Plans ARE prompts** — PLAN.md is directly consumed by the executor agent as its instructions, not a document that gets translated
2. **Goal-backward everywhere** — Goal → Observable Truths → Required Artifacts → Required Wiring (used in planning, verification, roadmapping)
3. **Context budget awareness** — Plans target ~50% of context window; agents get fresh 200k windows to avoid degradation
4. **Orchestrator stays lean** — Commands coordinate and pass data; agents do the heavy work in isolated contexts
5. **Atomic commits** — One commit per task, stage specific files (NEVER `git add .` or `git add -A`)
6. **Three return types** — COMPLETE, CHECKPOINT, INCONCLUSIVE — every agent uses these structured formats
7. **Deviation rules** — 4 priority-ordered rules: auto-fix bugs/critical/blockers (no ask), ASK about architectural changes
8. **Coverage is non-negotiable** — Every requirement maps to exactly one phase (enforced by roadmapper)
9. **Don't trust claims** — Verifier checks actual code existence/implementation/wiring, not SUMMARY.md text
10. **Interactive vs YOLO** — `config.json` mode controls all confirmation gates; `always_confirm_destructive` overrides YOLO for safety
11. **Scope creep prevention** — Phase boundaries from ROADMAP.md are FIXED; discuss-phase captures deferred ideas separately
12. **Parallel by default** — Plans in the same wave execute in parallel; researcher agents spawn in parallel

---

## 16. Complete Command Reference

### Project Initialization

| Command | Purpose | Agents Spawned |
|---------|---------|----------------|
| `/gsd:new-project` | Init project from scratch | 4× researcher, synthesizer, roadmapper |
| `/gsd:new-milestone` | Start new milestone cycle | 4× researcher, synthesizer, roadmapper |
| `/gsd:map-codebase [area]` | Analyze existing codebase | 4× codebase-mapper |

### Phase Planning

| Command | Purpose | Agents Spawned |
|---------|---------|----------------|
| `/gsd:discuss-phase N` | Gather user decisions → CONTEXT.md | None (interactive) |
| `/gsd:list-phase-assumptions N` | Surface Claude's assumptions | None (conversational) |
| `/gsd:research-phase N` | Standalone research → RESEARCH.md | phase-researcher |
| `/gsd:plan-phase N [flags]` | Create PLAN.md files | researcher, planner, plan-checker |

`plan-phase` flags: `--research` (force research), `--skip-research`, `--gaps` (from verification), `--skip-verify`

### Execution

| Command | Purpose | Agents Spawned |
|---------|---------|----------------|
| `/gsd:execute-phase N [--gaps-only]` | Execute all plans in wave order | executor (per plan), verifier |
| `/gsd:quick` | Small ad-hoc task | planner, executor |

### Verification

| Command | Purpose | Agents Spawned |
|---------|---------|----------------|
| `/gsd:verify-work N` | User acceptance testing | debugger (per gap), planner, plan-checker |
| `/gsd:audit-milestone [version]` | Cross-phase integration check | integration-checker |

### Roadmap Management

| Command | Purpose | Agents Spawned |
|---------|---------|----------------|
| `/gsd:add-phase <description>` | Append phase to end of milestone | None |
| `/gsd:insert-phase <after> <desc>` | Insert decimal phase (e.g., 3.1) | None |
| `/gsd:remove-phase N` | Remove future phase + renumber | None |
| `/gsd:plan-milestone-gaps` | Create phases from audit gaps | None |

### Milestone Management

| Command | Purpose | Agents Spawned |
|---------|---------|----------------|
| `/gsd:complete-milestone <version>` | Archive + git tag | None |

### Session Management

| Command | Purpose | Agents Spawned |
|---------|---------|----------------|
| `/gsd:progress` | Check status + smart routing | None |
| `/gsd:resume-work` | Restore session from STATE.md | None |
| `/gsd:pause-work` | Create .continue-here.md handoff | None |

### Debugging

| Command | Purpose | Agents Spawned |
|---------|---------|----------------|
| `/gsd:debug [description]` | Scientific method debugging | debugger |

### Todo Management

| Command | Purpose | Agents Spawned |
|---------|---------|----------------|
| `/gsd:add-todo [description]` | Capture idea/task | None |
| `/gsd:check-todos [area]` | List and act on todos | None |

### Configuration

| Command | Purpose | Agents Spawned |
|---------|---------|----------------|
| `/gsd:settings` | Configure all toggles | None |
| `/gsd:set-profile <profile>` | Switch model profile | None |

### Utility

| Command | Purpose | Agents Spawned |
|---------|---------|----------------|
| `/gsd:help` | Show command reference | None |
| `/gsd:update` | Update GSD to latest version | None |
| `/gsd:join-discord` | Show Discord invite link | None |
