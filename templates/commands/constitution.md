---
description: Create or update the project constitution from interactive or provided principle inputs, ensuring all dependent templates stay in sync.
handoffs: 
  - label: Build Specification
    agent: speckit.specify
    prompt: Implement the feature specification based on the updated constitution. I want to build...
---

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty).

## Pre-Execution Checks

**Extension hooks (before constitution update)**:
- If `.specify/extensions.yml` exists in the project root, read `hooks.before_constitution`; on invalid or unparsable YAML, skip hook checking silently and continue.
- Filter out hooks with explicit `enabled: false`; no `enabled` field means enabled.
- Do **not** interpret or evaluate hook `condition` expressions: absent/null/empty means executable; non-empty means skip the hook and leave evaluation to the HookExecutor implementation.
- Per executable hook, output by its `optional` flag:
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
- No hooks registered or `.specify/extensions.yml` absent: skip silently.

## Outline

Update `.specify/memory/constitution.md` — a TEMPLATE with bracketed placeholder tokens (e.g. `[PROJECT_NAME]`, `[PRINCIPLE_1_NAME]`): collect/derive concrete values, fill the template precisely, and propagate amendments across dependent artifacts. If the file is missing, copy `.specify/templates/constitution-template.md` first.

Execution flow:

1. Load `.specify/memory/constitution.md`; identify every `[ALL_CAPS_IDENTIFIER]` placeholder. **IMPORTANT**: the user might require fewer or more principles; if a number is specified, respect it — follow the general template and update the doc accordingly.

2. Collect/derive values: user input (conversation) first; else infer from repo context (README, docs, embedded prior constitution versions). `RATIFICATION_DATE` is the original adoption date (unknown: ask or mark TODO); `LAST_AMENDED_DATE` is today if changes are made, else keep previous. `CONSTITUTION_VERSION` bumps per semver — MAJOR: backward incompatible governance/principle removals or redefinitions; MINOR: new principle/section or materially expanded guidance; PATCH: clarifications, wording, typo fixes, non-semantic refinements. Ambiguous bump type: propose reasoning before finalizing.

3. Draft: replace every placeholder (none left except intentionally retained slots the project chose not to define — explicitly justify any left); preserve heading hierarchy; comments can be removed once replaced unless still clarifying. Each Principle section: succinct name line, paragraph or bullets of non-negotiable rules, explicit rationale if not obvious. Governance section: amendment procedure, versioning policy, compliance review expectations.

4. Consistency propagation (convert prior checklist into active validations) — read and check: `.specify/templates/plan-template.md` ("Constitution Check"/rules align with updated principles); `.specify/templates/spec-template.md` (update if mandatory sections/constraints change); `.specify/templates/tasks-template.md` (task categorization reflects new/removed principle-driven task types, e.g. observability, versioning, testing discipline); every `.specify/templates/commands/*.md` file including this one (no outdated references, e.g. agent-specific names like CLAUDE only, where generic guidance is required); runtime guidance docs (`README.md`, `docs/quickstart.md`, agent-specific guidance files if present) — update references to changed principles.

5. Sync Impact Report, prepended as an HTML comment atop the updated file: version old → new; modified principles (old title → new title if renamed); added sections; removed sections; templates requiring updates (✅ updated / ⚠ pending) with file paths; TODOs for intentionally deferred placeholders.

6. Validate before final output: no unexplained bracket tokens; version line matches report; dates ISO YYYY-MM-DD; principles declarative, testable, free of vague language ("should" → MUST/SHOULD with rationale where appropriate).

7. Write the result back to `.specify/memory/constitution.md` (overwrite).

8. Final summary: new version and bump rationale; files flagged for manual follow-up; suggested commit message (e.g. `docs: amend constitution to vX.Y.Z (principle additions + governance update)`).

Formatting: heading levels exactly as the template (do not demote/promote); wrap long rationale lines (<100 chars ideally) without awkward hard breaks; single blank line between sections; no trailing whitespace.

Partial updates (e.g. one principle revision) still require the validation and version decision steps. If critical info is missing (e.g. ratification date truly unknown), insert `TODO(<FIELD_NAME>): explanation` and include it under deferred items in the Sync Impact Report. Do not create a new template; always operate on the existing `.specify/memory/constitution.md` file.

## Post-Execution Checks

**Extension hooks (after constitution update)**: same rules as Pre-Execution Checks (existence check, invalid-YAML silent skip, `enabled` filtering, `condition` handling, silent skip when no hooks or no file), but read `hooks.after_constitution` and output per `optional` flag:
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
