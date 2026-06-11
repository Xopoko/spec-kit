---
description: Perform a non-destructive cross-artifact consistency and quality analysis across spec.md, plan.md, and tasks.md after task generation.
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

- If `.specify/extensions.yml` exists at the project root, read entries under `hooks.before_analyze`
- Invalid YAML: skip hook checking silently and continue
- Skip hooks with explicit `enabled: false`; missing `enabled` = enabled
- Never interpret/evaluate hook `condition`: absent/null/empty = executable; non-empty = skip; evaluation belongs to the HookExecutor implementation
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

    Wait for the result of the hook command before proceeding to the Goal.
    ```
- No hooks registered or no `.specify/extensions.yml`: skip silently

## Goal

Find inconsistencies, duplication, ambiguity, and underspecification across `spec.md`, `plan.md`, and `tasks.md` before implementation. MUST run only after `__SPECKIT_COMMAND_TASKS__` produced a complete `tasks.md`.

## Operating Constraints

**STRICTLY READ-ONLY**: Do **not** modify any files; output a structured analysis report. Remediation plan: optional, applied only via manually invoked follow-up commands after explicit user approval.

**Constitution Authority**: `/memory/constitution.md` is **non-negotiable** within this analysis scope. Conflicts are automatically CRITICAL: adjust spec/plan/tasks — never dilute, reinterpret, or silently ignore the principle. Principle changes happen only in a separate, explicit constitution update outside `__SPECKIT_COMMAND_ANALYZE__`.

## Execution Steps

### 1. Initialize Context

Run `{SCRIPT}` once from repo root; parse FEATURE_DIR and AVAILABLE_DOCS from JSON; derive absolute paths:

- SPEC = FEATURE_DIR/spec.md
- PLAN = FEATURE_DIR/plan.md
- TASKS = FEATURE_DIR/tasks.md

Missing required file: abort and name the prerequisite command to run.
For single quotes in args like "I'm Groot", use escape syntax: e.g 'I'\''m Groot' (or double-quote if possible: "I'm Groot").

### 2. Load Artifacts (Progressive Disclosure)

Load incrementally, minimal necessary context only; don't dump all content into analysis:

- **spec.md**: Overview/Context; Functional Requirements; Success Criteria (measurable outcomes — e.g., performance, security, availability, user success, business impact); User Stories; Edge Cases (if present)
- **plan.md**: architecture/stack choices; Data Model references; phases; technical constraints
- **tasks.md**: task IDs; descriptions; phase grouping; parallel markers [P]; referenced file paths
- **Constitution**: load `/memory/constitution.md` for principle validation

### 3. Build Semantic Models

Internal representations only; no raw artifacts in output:

- **Requirements inventory**: stable key per Functional Requirement (FR-###) and Success Criterion (SC-###); explicit ID = primary key when present, optionally plus an imperative-phrase slug for readability (e.g., "User can upload file" → `user-can-upload-file`). Only buildable-work Success Criteria (e.g., load-testing infrastructure); exclude post-launch metrics/business KPIs (e.g., "Reduce support tickets by 50%")
- **User story/action inventory**: discrete user actions with acceptance criteria
- **Task coverage mapping**: map each task to requirement(s)/story(ies) via keyword inference or explicit references (IDs, key phrases)
- **Constitution rule set**: principle names and MUST/SHOULD normative statements

### 4. Detection Passes (Token-Efficient Analysis)

Max 50 high-signal findings; summarize overflow.

- **A. Duplication**: near-duplicate requirements; mark lower-quality phrasing for consolidation
- **B. Ambiguity**: vague adjectives (fast, scalable, secure, intuitive, robust) lacking measurable criteria; unresolved placeholders (TODO, TKTK, ???, `<placeholder>`, etc.)
- **C. Underspecification**: requirements whose verb lacks an object/measurable outcome; user stories missing acceptance-criteria alignment; tasks referencing files/components not defined in spec/plan
- **D. Constitution Alignment**: requirement/plan elements conflicting with a MUST principle; missing constitution-mandated sections or quality gates
- **E. Coverage Gaps**: requirements with zero tasks; tasks with no mapped requirement/story; Success Criteria requiring buildable work (performance, security, availability) not reflected in tasks
- **F. Inconsistency**: terminology drift (same concept named differently across files); data entities in plan but absent in spec (or vice versa); task ordering contradictions (e.g., integration tasks before foundational setup without dependency note); conflicting requirements (e.g., Next.js vs Vue)

### 5. Severity Assignment

- **CRITICAL**: constitution MUST violation; missing core spec artifact; zero-coverage requirement blocking baseline functionality
- **HIGH**: duplicate or conflicting requirement; ambiguous security/performance attribute; untestable acceptance criterion
- **MEDIUM**: terminology drift; missing non-functional task coverage; underspecified edge case
- **LOW**: style/wording; minor redundancy not affecting execution order

### 6. Produce Compact Analysis Report

Output a Markdown report (no file writes):

## Specification Analysis Report

| ID | Category | Severity | Location(s) | Summary | Recommendation |
|----|----------|----------|-------------|---------|----------------|
| A1 | Duplication | HIGH | spec.md:L120-134 | Two similar requirements ... | Merge phrasing; keep clearer version |

(one row per finding; stable IDs prefixed by category initial)

**Coverage Summary Table:**

| Requirement Key | Has Task? | Task IDs | Notes |
|-----------------|-----------|----------|-------|

**Constitution Alignment Issues:** (if any)

**Unmapped Tasks:** (if any)

**Metrics:**

- Total Requirements
- Total Tasks
- Coverage % (requirements with >=1 task)
- Ambiguity Count
- Duplication Count
- Critical Issues Count

### 7. Next Actions

Concise Next Actions block at end:

- CRITICAL issues: recommend resolving before `__SPECKIT_COMMAND_IMPLEMENT__`
- Only LOW/MEDIUM: user may proceed; suggest improvements
- Suggest explicit commands, e.g. "Run __SPECKIT_COMMAND_SPECIFY__ with refinement", "Run __SPECKIT_COMMAND_PLAN__ to adjust architecture", "Manually edit tasks.md to add coverage for 'performance-metrics'"

### 8. Offer Remediation

Ask the user: "Would you like me to suggest concrete remediation edits for the top N issues?" (Do NOT apply them automatically.)

### 9. Check for extension hooks

After reporting, repeat the Pre-Execution hook procedure with `hooks.after_analyze` and these output templates:

- Per executable hook, output by `optional` flag:
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

## Operating Principles

- **Minimal high-signal tokens**: Focus on actionable findings, not exhaustive documentation
- **Progressive disclosure**: Load artifacts incrementally; don't dump all content into analysis
- **Deterministic**: unchanged reruns yield consistent IDs and counts
- **NEVER modify files** (read-only analysis)
- **NEVER hallucinate missing sections**; report absences accurately
- Constitution violations: always CRITICAL, top priority
- Cite specific instances, not generic patterns (examples over exhaustive rules)
- Zero issues: emit success report with coverage statistics

## Context

{ARGS}
