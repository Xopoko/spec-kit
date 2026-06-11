---
description: Execute the implementation planning workflow using the plan template to generate design artifacts.
handoffs: 
  - label: Create Tasks
    agent: speckit.tasks
    prompt: Break the plan into tasks
    send: true
  - label: Create Checklist
    agent: speckit.checklist
    prompt: Create a checklist for the following domain...
scripts:
  sh: scripts/bash/setup-plan.sh --json
  ps: scripts/powershell/setup-plan.ps1 -Json
---

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty).

## Pre-Execution Checks

- Missing `.specify/extensions.yml` (project root), no `hooks.before_plan` entries, or invalid/unparsable YAML → skip silently and continue.
- Skip hooks with `enabled` explicitly `false`; absent `enabled` = enabled.
- Do **not** interpret or evaluate hook `condition` expressions: absent/null/empty `condition` = executable; non-empty `condition` = skip (left to the HookExecutor implementation).
- Output per executable hook by `optional` flag:
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

## Outline

1. **Setup**: Run `{SCRIPT}` from repo root and parse JSON for FEATURE_SPEC, IMPL_PLAN, SPECS_DIR, BRANCH. Escape single quotes in args: 'I'\''m Groot' (or double-quote if possible: "I'm Groot").

2. **Load context**: Read FEATURE_SPEC and `/memory/constitution.md`. Load IMPL_PLAN template (already copied).

3. **Execute plan workflow** per the IMPL_PLAN template: fill Technical Context (mark unknowns as "NEEDS CLARIFICATION"); fill Constitution Check from constitution; evaluate gates (ERROR if violations unjustified); run Phases 0–1 (below), including the agent-script context update; re-evaluate Constitution Check post-design.

## Mandatory Post-Execution Hooks

**You MUST complete this section before reporting completion to the user.**

Apply the Pre-Execution Checks hook rules to `hooks.after_plan`; if skipped, skip to the Completion Report. Output per executable hook by `optional` flag:
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

Command ends after Phase 2 planning. Report branch, IMPL_PLAN path, and generated artifacts.

## Phases

### Phase 0: Outline & Research

1. **Extract unknowns from Technical Context**: NEEDS CLARIFICATION → research task; dependency → best practices task; integration → patterns task.

2. **Generate and dispatch research agents**:

   ```text
   For each unknown in Technical Context:
     Task: "Research {unknown} for {feature context}"
   For each technology choice:
     Task: "Find best practices for {tech} in {domain}"
   ```

3. **Consolidate findings** in `research.md`:
   - Decision: [what was chosen]
   - Rationale: [why chosen]
   - Alternatives considered: [what else evaluated]

**Output**: research.md with all NEEDS CLARIFICATION resolved

### Phase 1: Design & Contracts

**Prerequisites:** `research.md` complete

1. **Extract entities from feature spec** → `data-model.md`: name, fields, relationships; validation rules from requirements; state transitions if applicable.

2. **Define interface contracts** (if project has external interfaces) → `/contracts/`: identify interfaces exposed to users or other systems; document a contract format fitting the project type (e.g. public APIs for libraries). Skip if purely internal (build scripts, one-off tools, etc.).

3. **Create quickstart validation guide** → `quickstart.md`: runnable end-to-end validation scenarios — prerequisites, setup, test/run commands, expected outcomes. Link contracts/data model details, don't duplicate. No full implementation code, model/service/controller bodies, migrations, or complete test suites; validation/run guide only — implementation belongs in `tasks.md` and the implementation phase.

4. **Agent context update**: point the `<!-- SPECKIT START -->`/`<!-- SPECKIT END -->` plan reference in `__CONTEXT_FILE__` at the IMPL_PLAN path from step 1.

**Output**: data-model.md, /contracts/*, quickstart.md, updated agent context file

## Key rules

- Absolute paths for filesystem operations; project-relative paths in documentation and agent context references
- ERROR on gate failures or unresolved clarifications

## Done When

- [ ] Plan workflow executed; design artifacts generated
- [ ] Extension hooks dispatched or skipped per Mandatory Post-Execution Hooks rules above
- [ ] Branch, plan path, and generated artifacts reported to user
