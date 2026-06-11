---
description: Execute the implementation plan by processing and executing all tasks defined in tasks.md
scripts:
  sh: scripts/bash/check-prerequisites.sh --json --require-tasks --include-tasks
  ps: scripts/powershell/check-prerequisites.ps1 -Json -RequireTasks -IncludeTasks
---

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty).

## Pre-Execution Checks

**Extension hooks (before implementation)**:
- If `.specify/extensions.yml` exists in the project root, read entries under `hooks.before_implement`; if the file is missing, no hooks are registered, or the YAML cannot be parsed or is invalid, skip hook checking silently and continue
- Skip hooks where `enabled` is explicitly `false`; no `enabled` field means enabled
- Do **not** interpret or evaluate hook `condition` expressions: no/null/empty `condition` → executable; non-empty `condition` → skip it, leaving evaluation to the HookExecutor implementation
- For each executable hook, output per its `optional` flag:
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

1. Run `{SCRIPT}` from repo root and parse FEATURE_DIR and AVAILABLE_DOCS list. All paths must be absolute. For single quotes in args, use escape syntax: e.g 'I'\''m Groot' (or double-quote if possible: "I'm Groot").

2. **Check checklists status** (if FEATURE_DIR/checklists/ exists):
   - Scan all checklist files in the checklists/ directory
   - Count per checklist: total items (lines matching `- [ ]`, `- [X]`, or `- [x]`), completed (`- [X]`/`- [x]`), incomplete (`- [ ]`); create a status table:

     ```text
     | Checklist | Total | Completed | Incomplete | Status |
     |-----------|-------|-----------|------------|--------|
     | ux.md     | 12    | 12        | 0          | ✓ PASS |
     | test.md   | 8     | 5         | 3          | ✗ FAIL |
     | security.md | 6   | 6         | 0          | ✓ PASS |
     ```

   - Overall: **PASS** if every checklist has 0 incomplete items, else **FAIL**
   - **If any checklist is incomplete**: display the table, **STOP** and ask: "Some checklists are incomplete. Do you want to proceed with implementation anyway? (yes/no)", and wait for the response. "no"/"wait"/"stop" → halt execution; "yes"/"proceed"/"continue" → step 3
   - **If all complete**: display the table showing all passed and automatically proceed to step 3

3. Load and analyze the implementation context:
   - **REQUIRED**: Read tasks.md (complete task list, execution plan) and plan.md (tech stack, architecture, file structure)
   - **IF EXISTS**: Read data-model.md (entities, relationships), contracts/ (API specs, test requirements), research.md (technical decisions, constraints), /memory/constitution.md (governance constraints), quickstart.md (integration scenarios)

4. **Project Setup Verification** — **REQUIRED**: create/verify ignore files based on actual project setup (trigger → ignore file):
   - Git repo → .gitignore, if this succeeds:

     ```sh
     git rev-parse --git-dir 2>/dev/null
     ```

   - Dockerfile* or Docker in plan.md → .dockerignore
   - .eslintrc* → .eslintignore
   - eslint.config.* → ensure the config's `ignores` entries cover required patterns
   - .prettierrc* → .prettierignore
   - .npmrc or package.json → .npmignore (if publishing)
   - *.tf files → .terraformignore
   - helm charts → .helmignore

   Existing ignore file: verify essential patterns, append missing critical patterns only. Missing: create with full pattern set for detected technology.

   **Common Patterns by Technology** (from plan.md tech stack):
   - Node.js/JavaScript/TypeScript: `node_modules/`, `dist/`, `build/`, `*.log`, `.env*`
   - Python: `__pycache__/`, `*.pyc`, `.venv/`, `venv/`, `dist/`, `*.egg-info/`
   - Java: `target/`, `*.class`, `*.jar`, `.gradle/`, `build/`
   - C#/.NET: `bin/`, `obj/`, `*.user`, `*.suo`, `packages/`
   - Go: `*.exe`, `*.test`, `vendor/`, `*.out`
   - Ruby: `.bundle/`, `log/`, `tmp/`, `*.gem`, `vendor/bundle/`
   - PHP: `vendor/`, `*.log`, `*.cache`, `*.env`
   - Rust: `target/`, `debug/`, `release/`, `*.rs.bk`, `*.rlib`, `*.prof*`, `.idea/`, `*.log`, `.env*`
   - Kotlin: `build/`, `out/`, `.gradle/`, `.idea/`, `*.class`, `*.jar`, `*.iml`, `*.log`, `.env*`
   - C++: `build/`, `bin/`, `obj/`, `out/`, `*.o`, `*.so`, `*.a`, `*.exe`, `*.dll`, `.idea/`, `*.log`, `.env*`
   - C: `build/`, `bin/`, `obj/`, `out/`, `*.o`, `*.a`, `*.so`, `*.exe`, `*.dll`, `autom4te.cache/`, `config.status`, `config.log`, `.idea/`, `*.log`, `.env*`
   - Swift: `.build/`, `DerivedData/`, `*.swiftpm/`, `Packages/`
   - R: `.Rproj.user/`, `.Rhistory`, `.RData`, `.Ruserdata`, `*.Rproj`, `packrat/`, `renv/`
   - Universal: `.DS_Store`, `Thumbs.db`, `*.tmp`, `*.swp`, `.vscode/`, `.idea/`

   **Tool-Specific Patterns**:
   - Docker: `node_modules/`, `.git/`, `Dockerfile*`, `.dockerignore`, `*.log*`, `.env*`, `coverage/`
   - ESLint: `node_modules/`, `dist/`, `build/`, `coverage/`, `*.min.js`
   - Prettier: `node_modules/`, `dist/`, `build/`, `coverage/`, `package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`
   - Terraform: `.terraform/`, `*.tfstate*`, `*.tfvars`, `.terraform.lock.hcl`
   - Kubernetes/k8s: `*.secret.yaml`, `secrets/`, `.kube/`, `kubeconfig*`, `*.key`, `*.crt`

5. Parse tasks.md and extract: task phases (Setup, Tests, Core, Integration, Polish), dependencies (sequential vs parallel rules), task details (ID, description, file paths, parallel markers [P]), execution flow (order and dependency requirements).

6. Execute per the task plan: complete each phase before the next; run sequential tasks in order, parallel tasks [P] can run together; TDD — test tasks before their corresponding implementation tasks; tasks affecting the same files must run sequentially; verify phase completion before proceeding.

7. Execution order: setup first (project structure, dependencies, configuration); tests before code (write tests for contracts, entities, integration scenarios if needed); core development (models, services, CLI commands, endpoints); integration work (database connections, middleware, logging, external services); polish and validation (unit tests, performance optimization, documentation).

8. Progress tracking and error handling: report progress after each completed task; halt if any non-parallel task fails; for parallel tasks [P], continue with successful tasks and report failed ones; give clear error messages with debugging context; suggest next steps if implementation cannot proceed. **IMPORTANT** Mark each completed task off as [X] in the tasks file.

9. Completion validation: all required tasks completed; implemented features match the original specification; tests pass and coverage meets requirements; implementation follows the technical plan.

Note: This command assumes a complete task breakdown in tasks.md; if tasks are incomplete or missing, suggest running `__SPECKIT_COMMAND_TASKS__` first to regenerate the task list.

## Mandatory Post-Execution Hooks

**You MUST complete this section before reporting completion to the user.**

Check `.specify/extensions.yml` in the project root for entries under `hooks.after_implement`:
- If the file does not exist, no hooks are registered, or the YAML cannot be parsed or is invalid, skip hook checking silently and continue to the Completion Report
- Apply the same `enabled` and `condition` filtering rules as Pre-Execution Checks above
- For each executable hook, output per its `optional` flag:
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

Report final status with summary of completed work.

## Done When

- [ ] All tasks in tasks.md completed and marked `[X]`
- [ ] Implementation validated against specification, plan, and test coverage
- [ ] Extension hooks dispatched or skipped per Mandatory Post-Execution Hooks above
- [ ] Completion reported to user with summary of completed work
