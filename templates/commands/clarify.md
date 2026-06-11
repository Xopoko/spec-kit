---
description: Identify underspecified areas in the current feature spec by asking up to 5 highly targeted clarification questions and encoding answers back into the spec.
handoffs: 
  - label: Build Technical Plan
    agent: speckit.plan
    prompt: Create a plan for the spec. I am building with...
scripts:
   sh: scripts/bash/check-prerequisites.sh --json --paths-only
   ps: scripts/powershell/check-prerequisites.ps1 -Json -PathsOnly
---

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty).

## Pre-Execution Checks

**Extension hooks (before clarification)**: if `.specify/extensions.yml` exists in the project root, read entries under `hooks.before_clarify`; missing file or no registered hooks → skip silently. Invalid/unparseable YAML → skip hook checking silently, continue normally.
- Skip `enabled: false` hooks; missing `enabled` = enabled.
- Never interpret/evaluate hook `condition` expressions: none/null/empty → executable; non-empty → skip the hook (condition evaluation belongs to the HookExecutor implementation).
- Output per executable hook, by `optional` flag:
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

Goal: detect and reduce ambiguity or missing decision points in the active feature spec; record clarifications directly in the spec file.

Note: run and complete BEFORE `__SPECKIT_COMMAND_PLAN__`. If the user explicitly skips (e.g., exploratory spike), proceed but warn that downstream rework risk increases.

Execution steps:

1. Run `{SCRIPT}` from repo root **once** (combined `--json --paths-only` mode / `-Json -PathsOnly`). Parse `FEATURE_DIR`, `FEATURE_SPEC` (optionally `IMPL_PLAN`, `TASKS` for chained flows). Parse failure → abort; instruct user to re-run `__SPECKIT_COMMAND_SPECIFY__` or verify feature branch environment. Single quotes in args ("I'm Groot"): use escape syntax 'I'\''m Groot' (or double-quote: "I'm Groot").

2. **IF EXISTS**: load `/memory/constitution.md` for project principles and governance constraints.

3. Load the spec. Scan per taxonomy below, marking each category Clear / Partial / Missing; build an internal coverage map for prioritization (output only if no questions will be asked). Each Partial/Missing category yields a candidate question unless it would not materially change implementation/validation strategy, or is better deferred to planning (note internally).

   - Functional Scope & Behavior: core user goals & success criteria; explicit out-of-scope; user roles/personas
   - Domain & Data Model: entities, attributes, relationships; identity & uniqueness rules; lifecycle/state transitions; data volume/scale assumptions
   - Interaction & UX Flow: critical user journeys/sequences; error/empty/loading states; accessibility/localization notes
   - Non-Functional Quality Attributes: performance (latency/throughput); scalability (horizontal/vertical, limits); reliability & availability (uptime, recovery); observability (logging/metrics/tracing); security & privacy (authN/Z, data protection, threat assumptions); compliance/regulatory (if any)
   - Integration & External Dependencies: external services/APIs and failure modes; import/export formats; protocol/versioning assumptions
   - Edge Cases & Failure Handling: negative scenarios; rate limiting/throttling; conflict resolution (e.g., concurrent edits)
   - Constraints & Tradeoffs: technical constraints (language, storage, hosting); explicit tradeoffs/rejected alternatives
   - Terminology & Consistency: canonical glossary terms; avoided synonyms/deprecated terms
   - Completion Signals: acceptance criteria testability; measurable Definition of Done indicators
   - Misc / Placeholders: TODO markers/unresolved decisions; ambiguous adjectives ("robust", "intuitive") lacking quantification

4. Internally queue prioritized questions — max 5 total per session; do NOT output all at once.
    - Each answerable by multiple-choice (2–5 distinct, mutually exclusive options) OR one-word/short-phrase (explicitly constrain: "Answer in <=5 words").
    - Only answers materially impacting architecture, data modeling, task decomposition, test design, UX behavior, operational readiness, or compliance validation.
    - Highest-impact unresolved categories first; never two low-impact questions while a high-impact area (e.g., security posture) is unresolved.
    - Exclude already-answered questions, trivial stylistic preferences, plan-level execution details (unless blocking correctness).
    - Favor reducing downstream rework risk / preventing misaligned acceptance tests.
    - More than 5 categories unresolved → top 5 by (Impact * Uncertainty).

5. Sequential questioning loop (interactive):
    - EXACTLY ONE question at a time; never reveal queued questions in advance.
    - Multiple-choice: pick the most suitable option (project-type best practices; common patterns; risk reduction — security/performance/maintainability; alignment with spec goals/constraints). Lead with `**Recommended:** Option [X] - <reasoning>` (1-2 sentences why), then a Markdown table of all options:

       | Option | Description |
       |--------|-------------|
       | A | <Option A description> |
       | B | <Option B description> |
       | C | <Option C description> (add D/E as needed up to 5) |
       | Short | Provide a different short answer (<=5 words) (Include only if free-form alternative is appropriate) |

       After the table add: `You can reply with the option letter (e.g., "A"), accept the recommendation by saying "yes" or "recommended", or provide your own short answer.`
    - Short-answer (no meaningful discrete options): suggest from best practices/context as `**Suggested:** <your proposed answer> - <brief reasoning>`, then output: `Format: Short answer (<=5 words). You can accept the suggestion by saying "yes" or "suggested", or provide your own answer.`
    - On answer: "yes"/"recommended"/"suggested" → adopt your stated recommendation/suggestion; else validate it maps to one option or fits <=5 words; ambiguous → quick disambiguation (same question's count; do not advance); satisfactory → record in working memory (not yet to disk), next question.
    - Stop when: all critical ambiguities resolved early (remaining queued items unnecessary), OR user signals completion ("done", "good", "no more"), OR 5 questions asked.
    - No valid questions at start → immediately report no critical ambiguities.

6. Integration after EACH accepted answer:
    - Keep an in-memory spec (loaded once at start) plus raw file contents.
    - First answer this session: ensure `## Clarifications` exists (create just after the highest-level contextual/overview section per spec template); under it `### Session YYYY-MM-DD` for today if absent.
    - Append on acceptance: `- Q: <question> → A: <final answer>`.
    - Then apply to the most appropriate section(s):
       - Functional ambiguity → update/add bullet in Functional Requirements.
       - Interaction/actor distinction → User Stories or Actors subsection (if present): clarified role/constraint/scenario.
       - Data shape/entities → Data Model (fields, types, relationships); preserve ordering; note constraints succinctly.
       - Non-functional → measurable criteria in Success Criteria > Measurable Outcomes (vague adjective → metric/explicit target).
       - Edge case/negative flow → bullet under Edge Cases / Error Handling (create subsection if template has placeholder).
       - Terminology conflict → normalize across spec; original only if necessary via `(formerly referred to as "X")` once.
    - Invalidated earlier ambiguous statement → replace, don't duplicate; no obsolete contradictory text.
    - Save spec AFTER each integration (atomic overwrite, minimizes context loss).
    - No reordering unrelated sections; heading hierarchy intact.
    - Insertions minimal and testable (no narrative drift).

7. Validation (after EACH write plus final pass): Clarifications session has exactly one bullet per accepted answer (no duplicates); total asked (accepted) questions ≤ 5; no lingering vague placeholders the answers were meant to resolve; no contradictory earlier statements (now-invalid alternative choices removed); valid Markdown with only new headings `## Clarifications` and `### Session YYYY-MM-DD`; same canonical term across all updated sections.

8. Write the updated spec back to `FEATURE_SPEC`.

9. **Re-validate Spec Quality Checklist**: skip silently unless `FEATURE_DIR/checklists/requirements.md` exists. Then:
     1. Read the checklist file.
     2. Find GitHub task-list checkbox lines — `- [ ]`, `- [x]`, `- [X]` (case-insensitive, leading whitespace allowed for nesting) outside code fences; ignore all else (headings, notes, etc.).
     3. Snapshot each checkbox's marker state and item text (before-snapshot).
     4. Re-evaluate each item against the **updated** spec (saved in step 7).
     5. Toggle only real state changes: now passes & was unchecked → `[ ]` to `[x]`; now fails & was checked → `[x]`/`[X]` to `[ ]`; else leave as-is (preserve case, avoid cosmetic diffs).
     6. Save. **Only toggle the `[ ]`/`[x]` marker portion of checkbox lines whose state changed**; all other content (headings, notes, ordering, whitespace) unchanged — no noisy diffs.
     7. Compute for the Completion Report: **Newly passing** (unchecked → checked), **Regressions** (checked → unchecked), **Still unchecked**.
     8. Record before/after pass counts as checked/total (e.g., "12/16 → 15/16 items passing").

Behavior rules:

- No meaningful ambiguities (or only low-impact candidates) → respond: "No critical ambiguities detected worth formal clarification." and suggest proceeding.
- Spec file missing → instruct user to run `__SPECKIT_COMMAND_SPECIFY__` first (do not create a new spec here).
- Never exceed 5 total asked questions (retries for a single question do not count as new).
- No speculative tech stack questions unless absence blocks functional clarity.
- Respect early termination signals ("stop", "done", "proceed").
- Full coverage, no questions asked → compact coverage summary (all categories Clear), suggest advancing.
- Quota reached with high-impact categories unresolved → explicitly flag them under Deferred with rationale.

Context for prioritization: {ARGS}

## Mandatory Post-Execution Hooks

**You MUST complete this section before reporting completion to the user.**

If `.specify/extensions.yml` does not exist in the project root, or no hooks are registered under `hooks.after_clarify`, skip to the Completion Report. Otherwise apply the same rules as Pre-Execution Checks: invalid/unparseable YAML → skip hook checking silently, continue to the Completion Report; skip `enabled: false` hooks (missing `enabled` = enabled); never interpret/evaluate `condition` (none/null/empty → executable; non-empty → skip, HookExecutor's job). Output per executable hook, by `optional` flag:
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

After the questioning loop ends or early termination, report:
- Questions asked & answered count.
- Path to updated spec.
- Sections touched (names).
- Checklist status (if `FEATURE_DIR/checklists/requirements.md` was re-validated): before/after pass counts (e.g., "Spec Quality Checklist: 12/16 → 15/16 items passing"); state changes — newly checked and regressions; remaining unchecked items as areas needing attention.
- Coverage summary table per taxonomy category — Resolved (was Partial/Missing, addressed), Deferred (exceeds question quota or better suited for planning), Clear (already sufficient), Outstanding (still Partial/Missing but low impact).
- If Outstanding/Deferred remain: recommend proceeding to `__SPECKIT_COMMAND_PLAN__` or re-running `__SPECKIT_COMMAND_CLARIFY__` later post-plan.
- Suggested next command.

## Done When

- [ ] Spec ambiguities identified and clarifications integrated into spec file
- [ ] Spec quality checklist re-validated against updated spec (if `FEATURE_DIR/checklists/requirements.md` exists)
- [ ] Extension hooks dispatched or skipped per Mandatory Post-Execution Hooks rules above
- [ ] Completion reported: questions answered, sections touched, checklist status, coverage summary
