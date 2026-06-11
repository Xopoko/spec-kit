---
description: Convert existing tasks into actionable, dependency-ordered GitHub issues for the feature based on available design artifacts.
tools: ['github/github-mcp-server/issue_write']
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

**Check for extension hooks (before tasks-to-issues conversion)**:
- Read `.specify/extensions.yml` (project root) entries under `hooks.before_taskstoissues`. If the file is missing, no hooks are registered, or the YAML is invalid, skip silently and continue.
- Skip hooks with explicit `enabled: false`; missing `enabled` = enabled.
- Do **not** interpret or evaluate hook `condition` expressions: missing/null/empty = executable; non-empty = skip and leave evaluation to the HookExecutor implementation.
- Output per executable hook's `optional` flag:
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

1. Run `{SCRIPT}` from repo root and parse FEATURE_DIR and AVAILABLE_DOCS list. All paths must be absolute. Escape single quotes in args: 'I'\''m Groot' (or double-quote if possible: "I'm Groot").
1. **IF EXISTS**: Load `/memory/constitution.md` for project principles and governance constraints.
1. Extract the **tasks** path from the executed script.
1. Get the Git remote:

```bash
git config --get remote.origin.url
```

> [!CAUTION]
> ONLY PROCEED TO NEXT STEPS IF THE REMOTE IS A GITHUB URL

1. For each task in the list, use the GitHub MCP server to create an issue in the repository matching the Git remote.

> [!CAUTION]
> UNDER NO CIRCUMSTANCES EVER CREATE ISSUES IN REPOSITORIES THAT DO NOT MATCH THE REMOTE URL

## Post-Execution Checks

**Check for extension hooks (after tasks-to-issues conversion)**: same rules as Pre-Execution Checks, but read `hooks.after_taskstoissues`; output per `optional` flag:
  - **Optional hook** (`optional: true`):
    ```
    ## Extension Hooks

    **Optional Hook**: {extension}
    Command: `/{command}`
    Description: {description}

    Prompt: {prompt}
    To execute: `/{command}`
    ```
  - **Mandatory hook** (`optional: false`):
    ```
    ## Extension Hooks

    **Automatic Hook**: {extension}
    Executing: `/{command}`
    EXECUTE_COMMAND: {command}
    ```
