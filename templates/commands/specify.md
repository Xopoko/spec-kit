---
description: Create or update the feature specification from a natural language feature description.
handoffs: 
  - label: Build Technical Plan
    agent: speckit.plan
    prompt: Create a plan for the spec. I am building with...
  - label: Clarify Spec Requirements
    agent: speckit.clarify
    prompt: Clarify specification requirements
    send: true
---

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty).

## Pre-Execution Checks

If `.specify/extensions.yml` exists in the project root, read entries under `hooks.before_specify`; unparsable/invalid YAML → skip hook checking silently and continue normally. Filter out hooks with `enabled` explicitly `false` (missing `enabled` = enabled). Do **not** interpret or evaluate hook `condition` expressions: missing/null/empty `condition` = executable; non-empty = skip the hook (left to the HookExecutor implementation). For each executable hook, output by its `optional` flag:
- `optional: true`:
    ```
    ## Extension Hooks

    **Optional Pre-Hook**: {extension}
    Command: `/{command}`
    Description: {description}

    Prompt: {prompt}
    To execute: `/{command}`
    ```
- `optional: false`:
    ```
    ## Extension Hooks

    **Automatic Pre-Hook**: {extension}
    Executing: `/{command}`
    EXECUTE_COMMAND: {command}

    Wait for the result of the hook command before proceeding to the Outline.
    ```

No registered hooks or no `.specify/extensions.yml`: skip silently.

## Outline

Text the user typed after `__SPECKIT_COMMAND_SPECIFY__` in the triggering message **is** the feature description — available even if `{ARGS}` appears literally below; do not ask the user to repeat it unless they provided an empty command. Then:

1. **Generate a concise short name** (2-4 words): meaningful keywords, action-noun format when possible (e.g., "add-user-auth", "fix-payment-bug"), preserve technical terms/acronyms (OAuth2, API, JWT), descriptive at a glance. Examples: "Implement OAuth2 integration for the API" → "oauth2-api-integration"; "Fix payment processing timeout bug" → "fix-payment-timeout".

2. **Branch creation** (optional, via hook): a successful `before_specify` hook created/switched to a git branch and output JSON with `BRANCH_NAME` and `FEATURE_NUM`. Note them; the branch name does **not** dictate the spec directory name. A user-provided `GIT_BRANCH_NAME` passes through to the hook as the exact branch name (bypassing all prefix/suffix generation).

3. **Create the spec feature directory**:

   Resolve `SPECIFY_FEATURE_DIRECTORY`:
   1. Explicit user value (env var/argument/config): use as-is.
   2. Otherwise auto-generate under the default `specs/`:
      - Check `.specify/init-options.json` for `feature_numbering` (preferred) or `branch_numbering` (deprecated, migration only, will be removed)
      - `"timestamp"` → prefix `YYYYMMDD-HHMMSS` (current timestamp); `"sequential"`/absent → prefix `NNN` (next available 3-digit number after scanning `specs/`)
      - Directory name `<prefix>-<short-name>` (e.g., `003-user-auth` or `20260319-143022-user-auth`); set `SPECIFY_FEATURE_DIRECTORY` to `specs/<directory-name>`
      - `branch_numbering` used without `feature_numbering` → one-line warning: "⚠️ `branch_numbering` in init-options.json is deprecated. Rename to `feature_numbering`."

   Then:
   - `mkdir -p SPECIFY_FEATURE_DIRECTORY`
   - Resolve the active `spec-template` via the Spec Kit preset/template resolution stack (equivalent to `specify preset resolve spec-template`); copy it to `SPECIFY_FEATURE_DIRECTORY/spec.md` and set `SPEC_FILE` to that path
   - Persist the resolved path to `.specify/feature.json`:
     ```json
     {
       "feature_directory": "<resolved feature dir>"
     }
     ```
     Write the actual resolved path (e.g., `specs/003-user-auth`), not the literal `SPECIFY_FEATURE_DIRECTORY`; downstream commands (`__SPECKIT_COMMAND_PLAN__`, `__SPECKIT_COMMAND_TASKS__`, etc.) read it to locate the feature directory without relying on git branch names.

   **IMPORTANT**: only one feature per `__SPECKIT_COMMAND_SPECIFY__` invocation; spec directory and git branch names are independent (matching is the user's choice); spec directory and file are always created by this command, never by the hook.

4. Load the resolved active `spec-template` for its required sections.

5. **IF EXISTS**: Load `/memory/constitution.md` for principles and governance constraints.

6. Execution flow:
    1. Parse user description from arguments; empty → ERROR "No feature description provided"
    2. Extract key concepts: actors, actions, data, constraints
    3. Unclear aspects: informed guesses from context and industry standards; mark [NEEDS CLARIFICATION: specific question] only when the choice significantly impacts feature scope or user experience, multiple reasonable interpretations diverge in implications, or no reasonable default exists. **LIMIT: max 3 [NEEDS CLARIFICATION] markers total**; priority: scope > security/privacy > user experience > technical details
    4. Fill User Scenarios & Testing; no clear user flow → ERROR "Cannot determine user scenarios"
    5. Generate Functional Requirements, each testable; reasonable defaults for unspecified details (document in Assumptions)
    6. Define Success Criteria: measurable, technology-agnostic outcomes; quantitative metrics (time, performance, volume) and qualitative measures (user satisfaction, task completion); each verifiable without implementation details
    7. Identify Key Entities (if data involved)
    8. Return: SUCCESS (spec ready for planning)

6. Write the spec to SPEC_FILE using the template structure, replacing placeholders with concrete details from the feature description (arguments), preserving section order and headings.

7. **Specification Quality Validation**:

   a. Create `SPECIFY_FEATURE_DIRECTORY/checklists/requirements.md` (checklist template structure) with these items:

      ```markdown
      # Specification Quality Checklist: [FEATURE NAME]
      
      **Purpose**: Validate specification completeness and quality before proceeding to planning
      **Created**: [DATE]
      **Feature**: [Link to spec.md]
      
      ## Content Quality
      
      - [ ] No implementation details (languages, frameworks, APIs)
      - [ ] Focused on user value and business needs
      - [ ] Written for non-technical stakeholders
      - [ ] All mandatory sections completed
      
      ## Requirement Completeness
      
      - [ ] No [NEEDS CLARIFICATION] markers remain
      - [ ] Requirements are testable and unambiguous
      - [ ] Success criteria are measurable
      - [ ] Success criteria are technology-agnostic (no implementation details)
      - [ ] All acceptance scenarios are defined
      - [ ] Edge cases are identified
      - [ ] Scope is clearly bounded
      - [ ] Dependencies and assumptions identified
      
      ## Feature Readiness
      
      - [ ] All functional requirements have clear acceptance criteria
      - [ ] User scenarios cover primary flows
      - [ ] Feature meets measurable outcomes defined in Success Criteria
      - [ ] No implementation details leak into specification
      
      ## Notes
      
      - Items marked incomplete require spec updates before `__SPECKIT_COMMAND_CLARIFY__` or `__SPECKIT_COMMAND_PLAN__`
      ```

   b. Review the spec against each checklist item: pass/fail; document specific issues (quote spec sections).

   c. Handle results:
      - **All items pass**: mark checklist complete; proceed to Mandatory Post-Execution Hooks.
      - **Items fail (excluding [NEEDS CLARIFICATION])**: list failing items and specific issues; update the spec to address each; re-run validation until all pass (max 3 iterations); still failing after 3 → document remaining issues in checklist notes and warn user.
      - **[NEEDS CLARIFICATION] markers remain**:
        1. Extract all [NEEDS CLARIFICATION: ...] markers from the spec
        2. **LIMIT CHECK**: more than 3 → keep the 3 most critical (by scope/security/UX impact), informed guesses for the rest
        3. Present options for each clarification (max 3) in this format:

           ```markdown
           ## Question [N]: [Topic]
           
           **Context**: [Quote relevant spec section]
           
           **What we need to know**: [Specific question from NEEDS CLARIFICATION marker]
           
           **Suggested Answers**:
           
           | Option | Answer | Implications |
           |--------|--------|--------------|
           | A      | [First suggested answer] | [What this means for the feature] |
           | B      | [Second suggested answer] | [What this means for the feature] |
           | C      | [Third suggested answer] | [What this means for the feature] |
           | Custom | Provide your own answer | [Explain how to provide custom input] |
           
           **Your choice**: _[Wait for user response]_
           ```

        4. **CRITICAL - Table Formatting**: pipes aligned, consistent spacing; spaces around cell content (`| Content |` not `|Content|`); header separator at least 3 dashes (`|--------|`); test that the table renders correctly in markdown preview
        5. Number questions sequentially (Q1, Q2, Q3 - max 3 total)
        6. Present all questions together before waiting for responses
        7. Wait for choices for all questions (e.g., "Q1: A, Q2: Custom - [details], Q3: B")
        8. Replace each [NEEDS CLARIFICATION] marker with the selected or provided answer
        9. Re-run validation after all clarifications are resolved

   d. Update the checklist's pass/fail status after each validation iteration.

## Mandatory Post-Execution Hooks

**You MUST complete this section before reporting completion to the user.**

If `.specify/extensions.yml` is missing, has no entries under `hooks.after_specify`, or its YAML is unparsable/invalid, skip (silently) to the Completion Report. Otherwise apply the Pre-Execution Checks rules (`enabled` filtering, `condition` handling) to `hooks.after_specify`; for each executable hook, output by its `optional` flag:
- `optional: false` — **You MUST emit `EXECUTE_COMMAND:` for each mandatory hook**:
    ```
    ## Extension Hooks

    **Automatic Hook**: {extension}
    Executing: `/{command}`
    EXECUTE_COMMAND: {command}
    ```
- `optional: true`:
    ```
    ## Extension Hooks

    **Optional Hook**: {extension}
    Command: `/{command}`
    Description: {description}

    Prompt: {prompt}
    To execute: `/{command}`
    ```

## Completion Report

Report to the user: `SPECIFY_FEATURE_DIRECTORY`, `SPEC_FILE`, checklist results summary, and readiness for the next phase (`__SPECKIT_COMMAND_CLARIFY__` or `__SPECKIT_COMMAND_PLAN__`). (Branch creation: `before_specify` hook, git extension; spec directory/file: always this core command.)

## Quick Guidelines

- Focus on **WHAT** users need and **WHY**; avoid HOW (no tech stack, APIs, code structure); write for business stakeholders, not developers.
- DO NOT create checklists embedded in the spec — separate command.
- Complete all mandatory sections; include optional sections only when relevant; remove non-applicable sections entirely (don't leave as "N/A").

### For AI Generation

1. **Make informed guesses**: fill gaps from context, industry standards, and common patterns
2. **Document assumptions**: record reasonable defaults in the Assumptions section
3. **Limit clarifications**: max 3 [NEEDS CLARIFICATION] markers — same critical-decision criteria as execution-flow step 3
4. **Prioritize clarifications**: scope > security/privacy > user experience > technical details
5. **Think like a tester**: every vague requirement should fail the "testable and unambiguous" checklist item
6. **Common areas needing clarification** (only if no reasonable default exists): feature scope and boundaries (include/exclude use cases); user types and permissions (if conflicting interpretations possible); security/compliance requirements (when legally/financially significant)

**Reasonable defaults** (don't ask): industry-standard data retention for the domain; standard web/mobile performance targets unless specified; user-friendly error messages with appropriate fallbacks; standard session-based or OAuth2 authentication for web apps; project-appropriate integration patterns (REST/GraphQL for web services, function calls for libraries, CLI args for tools, etc.)

### Success Criteria Guidelines

Must be **measurable** (time, percentage, count, rate), **technology-agnostic** (no frameworks, languages, databases, or tools), **user-focused** (user/business outcomes, not system internals), and **verifiable** without implementation details. Good: "Users can complete checkout in under 3 minutes", "System supports 10,000 concurrent users", "95% of searches return results in under 1 second", "Task completion rate improves by 40%". Bad: "API response time is under 200ms" (too technical, use "Users see results instantly"), "Database can handle 1000 TPS" (implementation detail, use user-facing metric), "React components render efficiently" (framework-specific), "Redis cache hit rate above 80%" (technology-specific).

## Done When

- [ ] Specification written to `SPEC_FILE` and validated against quality checklist
- [ ] Extension hooks dispatched or skipped per Mandatory Post-Execution Hooks
- [ ] Completion reported to user with feature directory, spec file path, and checklist results
