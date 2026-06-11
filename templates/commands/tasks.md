---
description: Generate an actionable, dependency-ordered tasks.md for the feature based on available design artifacts.
handoffs: 
  - label: Analyze For Consistency
    agent: speckit.analyze
    prompt: Run a project analysis for consistency
    send: true
  - label: Implement Project
    agent: speckit.implement
    prompt: Start the implementation in phases
    send: true
scripts:
  sh: scripts/bash/setup-tasks.sh --json
  ps: scripts/powershell/setup-tasks.ps1 -Json
---

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty).

## Pre-Execution Checks

**Check for extension hooks (before tasks generation)**:
- If `.specify/extensions.yml` exists in the project root, read `hooks.before_tasks`; invalid/unparseable YAML → skip hook checking silently, continue normally
- `enabled: false` (explicit) → skip hook; missing `enabled` → enabled by default
- Do **not** interpret or evaluate hook `condition` expressions: no/null/empty `condition` → executable; non-empty → skip (HookExecutor evaluates)
- Per executable hook, output by `optional` flag:
  - **Optional hook** (`optional: true`):
    ```
    ## Extension Hooks

    **Optional Pre-Hook**: {extension}
    Command: `/{command}`
    Description: {description}

    Prompt: {prompt}
    To execute: `/{command}`
    ```
  - **Mandatory hook** (`optional: false`):
    ```
    ## Extension Hooks

    **Automatic Pre-Hook**: {extension}
    Executing: `/{command}`
    EXECUTE_COMMAND: {command}
    
    Wait for the result of the hook command before proceeding to the Outline.
    ```
- If no hooks are registered or `.specify/extensions.yml` does not exist, skip silently

## Outline

1. **Setup**: Run `{SCRIPT}` from repo root; parse FEATURE_DIR, TASKS_TEMPLATE, AVAILABLE_DOCS (document names/relative paths under `FEATURE_DIR`, e.g. `research.md` or `contracts/`). `FEATURE_DIR` and `TASKS_TEMPLATE` must be absolute paths when provided. For single quotes in args like "I'm Groot", use escape syntax: e.g 'I'\''m Groot' (or double-quote if possible: "I'm Groot").

2. **Load from FEATURE_DIR** — required: plan.md (tech stack/libraries/structure), spec.md (prioritized user stories); optional: data-model.md (entities), contracts/ (interface contracts), research.md (decisions), quickstart.md (test scenarios); IF EXISTS: `/memory/constitution.md` (principles/governance constraints). Generate from what's available.

3. **Workflow**: from plan.md extract tech stack/libraries/structure; from spec.md user stories with priorities (P1, P2, P3...); when present, map data-model.md entities and contracts/ interface contracts to stories, and use research.md decisions for setup tasks. Generate story-organized tasks (rules below), a story-completion-order dependency graph, and per-story parallel execution examples; validate each story independently testable with all needed tasks.

4. **Generate tasks.md** from the TASKS_TEMPLATE file (JSON above; empty → `.specify/templates/tasks-template.md`) as structure. Fill: feature name (plan.md); Phase 1 Setup (project initialization); Phase 2 Foundational (blocking prerequisites for all stories); Phase 3+ per-story phases in spec.md priority order — goal, independent test criteria, tests (if requested), implementation tasks; Final Phase Polish & cross-cutting; strict checklist format (rules below), clear file paths; Dependencies (story completion order); per-story parallel examples; implementation strategy (MVP first, incremental delivery).

## Mandatory Post-Execution Hooks

**You MUST complete this section before reporting completion to the user.**

Check `.specify/extensions.yml` (project root) for `hooks.after_tasks`. Missing file, no hooks, or invalid YAML → skip silently to the Completion Report. Otherwise apply the Pre-Execution Checks filtering rules (`enabled`, `condition`); output per executable hook by `optional` flag:
  - **Mandatory hook** (`optional: false`) — **You MUST emit `EXECUTE_COMMAND:` for each mandatory hook**:
    ```
    ## Extension Hooks

    **Automatic Hook**: {extension}
    Executing: `/{command}`
    EXECUTE_COMMAND: {command}
    ```
  - **Optional hook** (`optional: true`):
    ```
    ## Extension Hooks

    **Optional Hook**: {extension}
    Command: `/{command}`
    Description: {description}

    Prompt: {prompt}
    To execute: `/{command}`
    ```

## Completion Report

Output the generated tasks.md path plus: total and per-story task counts; parallel opportunities identified; per-story independent test criteria; suggested MVP scope (typically just User Story 1); format validation: confirm ALL tasks follow the checklist format (checkbox, ID, labels, file paths).

Context for task generation: {ARGS}

The tasks.md should be immediately executable — each task must be specific enough that an LLM can complete it without additional context.

## Task Generation Rules

**CRITICAL**: Tasks MUST be organized by user story to enable independent implementation and testing.

**Tests are OPTIONAL**: Only generate test tasks if explicitly requested in the feature specification or if user requests TDD approach.

### Checklist Format (REQUIRED)

Every task MUST strictly follow this format:

```text
- [ ] [TaskID] [P?] [Story?] Description with file path
```

1. **Checkbox**: ALWAYS start with `- [ ]`
2. **Task ID**: sequential (T001, T002, T003...), execution order
3. **[P] marker**: ONLY if parallelizable (different files, no dependencies on incomplete tasks)
4. **[Story] label** ([US1], [US2]... = spec.md user stories): REQUIRED on user story phase tasks only; NO story label on Setup/Foundational/Polish phases
5. **Description**: clear action + exact file path

Examples:

- ✅ CORRECT: `- [ ] T001 Create project structure per implementation plan`
- ✅ CORRECT: `- [ ] T005 [P] Implement authentication middleware in src/middleware/auth.py`
- ✅ CORRECT: `- [ ] T012 [P] [US1] Create User model in src/models/user.py`
- ✅ CORRECT: `- [ ] T014 [US1] Implement UserService in src/services/user_service.py`
- ❌ WRONG: `- [ ] Create User model` (missing ID and Story label)
- ❌ WRONG: `T001 [US1] Create model` (missing checkbox)
- ❌ WRONG: `- [ ] [US1] Create User model` (missing Task ID)
- ❌ WRONG: `- [ ] T001 [US1] Create model` (missing file path)

### Task Organization

1. **From User Stories (spec.md)** — PRIMARY ORGANIZATION: each user story (P1, P2, P3...) gets its own phase; map all related components to their story (models, services, interfaces/UI, plus story-specific tests if requested); mark story dependencies (most stories should be independent)
2. **From Contracts**: each interface contract → the user story it serves; if tests requested, each interface contract → contract test task [P] before implementation in that story's phase
3. **From Data Model**: each entity → the user story(ies) needing it; multi-story entity → earliest story or Setup phase; relationships → service layer tasks in appropriate story phase
4. **From Setup/Infrastructure**: shared infrastructure → Setup phase (Phase 1); foundational/blocking tasks → Foundational phase (Phase 2); story-specific setup → within that story's phase

### Phase Structure

- **Phase 1**: Setup (project initialization)
- **Phase 2**: Foundational (blocking prerequisites - MUST complete before user stories)
- **Phase 3+**: User Stories in priority order (P1, P2, P3...); within each story: Tests (if requested) → Models → Services → Endpoints → Integration; each phase a complete, independently testable increment
- **Final Phase**: Polish & Cross-Cutting Concerns

## Done When

- [ ] tasks.md generated with all phases, task IDs, and file paths
- [ ] Extension hooks dispatched or skipped per Mandatory Post-Execution Hooks above
- [ ] Completion reported to user with task count, story breakdown, and MVP scope
