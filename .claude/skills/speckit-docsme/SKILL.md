---
name: "speckit-docsme"
description: Analyze an existing brownfield codebase exhaustively across two passes and write a structured docs.md covering principles, standards, and governance.
argument-hint: "Optional scope path to narrow documentation analysis (omit for full-repo)"
compatibility: "Requires spec-kit project structure with .specify/ directory"
metadata:
  author: "github-spec-kit"
  source: "templates/commands/docsme.md"
user-invocable: true
disable-model-invocation: true
---

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty).

## Pre-Execution Checks

**Check for extension hooks (before documentation)**:
- Check if `.specify/extensions.yml` exists in the project root.
- If it exists, read it and look for entries under the `hooks.before_docsme` key
- If the YAML cannot be parsed or is invalid, skip hook checking silently and continue normally
- Filter out hooks where `enabled` is explicitly `false`. Treat hooks without an `enabled` field as enabled by default.
- For each remaining hook, do **not** attempt to interpret or evaluate hook `condition` expressions:
  - If the hook has no `condition` field, or it is null/empty, treat the hook as executable
  - If the hook defines a non-empty `condition`, skip the hook and leave condition evaluation to the HookExecutor implementation
- For each executable hook, output the following based on its `optional` flag:
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

### Step 1 — Scope & Existing File Check

Parse the user input from `$ARGUMENTS`:

**If `$ARGUMENTS` is non-empty**:
- Treat the value as a scope path or keyword.
- If it resolves to an existing directory within the current working directory, set `ANALYSIS_ROOT` to that directory path.
- Otherwise, match it against top-level directory and module names. If no match is found, inform the user: `[red]Error:[/red] Scope path or keyword not found. Provide a valid directory path or omit for full-repo analysis.` and stop.
- Derive `SCOPE_LABEL` from the last non-empty path segment of the resolved directory (e.g., `src/specify_cli/extensions` → `extensions`).

**If `$ARGUMENTS` is empty or absent**:
- Set `ANALYSIS_ROOT` to the current working directory.
- Set `SCOPE_LABEL` to the project name (from `pyproject.toml` name field or root directory basename).

**Size check** (FR-010):
- Estimate the lines of source code within `ANALYSIS_ROOT` by scanning file counts and sizes.
- If the codebase appears to exceed **5,000 lines** of source code, inform the user:
  `The repository contains more than 5,000 lines of source code. For best results, provide a scope path to narrow the analysis (e.g., \`/speckit-docsme src/specify_cli\`). Proceeding with full-repo analysis may span multiple turns.`
- Continue regardless — do not stop. Let the user decide.

Confirm scope to the user: `Analyzing scope: <ANALYSIS_ROOT>`

**Existing file check** (FR-007):
- Check whether `specs/docs.md` already exists.
- If it exists:
  - Display:
    ```
    [yellow]Warning:[/yellow] specs/docs.md already exists.

    Choose an action:
      A — Overwrite with a fresh analysis (complete re-analysis; existing file is replaced)
      B — Amend (append a ## Amendment — YYYY-MM-DD section with changes since last generation)
    ```
  - Wait for the user's explicit response.
    - If the user chooses **A** (overwrite): set `WRITE_MODE = overwrite`. Proceed to Step 2. The existing file will be fully replaced.
    - If the user chooses **B** (amend): set `WRITE_MODE = amend`. Read the existing `specs/docs.md` content into memory as `EXISTING_DOCS`. Proceed to Step 2. Steps 3–4 (both analysis passes) MUST still run in full — the amendment section is based on a fresh analysis compared against `EXISTING_DOCS`. When writing the amendment in Step 5, include only findings that differ from `EXISTING_DOCS` (new principles observed, changed examples, new or resolved gaps, revised governance rules). Preserve `EXISTING_DOCS` content exactly — do NOT modify the original sections.
    - If the user cancels or responds ambiguously: stop cleanly.
- If `specs/docs.md` does not exist: set `WRITE_MODE = overwrite`. Proceed.

### Step 2 — Load Constitution

Check whether `.specify/memory/constitution.md` exists.

**If present**:
- Read it in full.
- Extract and record in memory as `CONSTITUTION_PRINCIPLES`:
  - All principle names (e.g., "I. Code Quality & Modularity", "II. Test-First & Coverage Standards", etc.)
  - All MUST/SHOULD rules from each principle
  - The development standards (technology stack, quality gates)
- These will be used in Step 5 to add "See also:" references in each relevant docs.md section rather than duplicating the text.

**If absent**:
- Record `CONSTITUTION_PRINCIPLES = []`.
- Note in the Governance section of `docs.md`: "No project constitution is present. The governance rules below are derived entirely from codebase observation and should be reviewed for adoption into a formal constitution."

### Step 3 — Structural Pass (Pass 1 of 2)

**Progress**: Display step status `Structural analysis (pass 1/2)` as running.

Perform an exhaustive structural scan of `ANALYSIS_ROOT`. Use file-reading tools iteratively — do NOT skim. Read module files, configuration files, and test files within scope.

1. **Entry points**: Identify all public entry points — CLI commands, exported functions, registered hooks, script invocations, public APIs.
2. **Module boundaries**: Map every module/package/directory and its stated responsibility (from docstrings, `__init__.py`, README files within scope).
3. **Code conventions**: Observe naming patterns (file names, function names, variable names), module sizes (line counts), import organisation, comment density, and formatting consistency.
4. **Test patterns**: Scan all test files within or referencing the scope. Measure approximate test-to-source line ratio. Note testing frameworks, fixture patterns, mocking approaches, and which behaviors have tests.
5. **Configuration files**: Read any `.json`, `.yaml`, `.toml`, `.ini`, `.cfg` files within scope that define behavior, dependencies, or quality tooling.
6. **Dependency manifest**: If `pyproject.toml`, `package.json`, `requirements.txt`, or equivalent exists, read it and record declared dependencies and quality tooling configured.

Produce an **internal intermediate outline** (not written to disk):

```
STRUCTURAL_OUTLINE = {
  entry_points: [...],
  module_responsibilities: {...},
  code_conventions: {
    naming: ..., module_sizes: ..., import_style: ..., comment_style: ...
  },
  test_patterns: {
    frameworks: [...], ratio: ..., fixtures: [...], coverage_tooling: ...
  },
  configuration_contracts: [...],
  dependency_manifest: {...}
}
```

**Progress**: Mark `Structural analysis (pass 1/2)` as done.

### Step 4 — Standards Pass (Pass 2 of 2)

**Progress**: Display step status `Standards analysis (pass 2/2)` as running.

Using `STRUCTURAL_OUTLINE` as the foundation, perform a deeper standards reconstruction. Do NOT skim — re-read files as needed to gather concrete examples for each finding.

1. **Code quality principles**: For each observable quality pattern (module size discipline, error handling approach, type annotation usage, dependency management, etc.), derive a named principle. For each principle:
   - Write a clear description of the pattern as it exists in the codebase.
   - Identify ≥1 concrete example — cite actual file paths, function names, or configuration values observed (these MAY appear in examples; they MUST NOT appear as primary explanatory content in the principle description itself).
   - Identify any **gaps** where current code deviates from the pattern (e.g., files that exceed a size convention, areas without type annotations).

2. **Testing standards**: Derive the testing discipline from observed test patterns:
   - What testing approach is followed (unit, integration, contract, etc.)?
   - What is the approximate test coverage level?
   - What frameworks and patterns are in use?
   - Identify gaps (behaviors without tests, coverage areas not met).

3. **UX consistency patterns**: Observe how the project communicates with users:
   - Output format conventions (progress indicators, success/error formatting).
   - Interactive prompt patterns.
   - Error message quality (specificity, hints, examples).
   - Identify gaps (inconsistent formatting, missing hints, etc.).

4. **Performance behaviors**: Observe how the project handles performance-sensitive operations:
   - Caching mechanisms and TTLs.
   - Timeout configurations.
   - Offline/air-gapped support.
   - Synchronous vs asynchronous I/O patterns.
   - Identify gaps.

5. **Constitution alignment** (if `CONSTITUTION_PRINCIPLES` is non-empty): For each identified principle, find the closest matching constitution principle by name and record the mapping for use as "See also:" references.

Produce `STANDARDS_OUTLINE`:

```
STANDARDS_OUTLINE = {
  ...STRUCTURAL_OUTLINE,
  quality_principles: [
    { name: ..., description: ..., examples: [...], gaps: [...], constitution_ref: ... }
  ],
  testing_standards: { description: ..., frameworks: [...], coverage: ..., gaps: [...] },
  ux_patterns: { description: ..., conventions: [...], gaps: [...] },
  performance_behaviors: { description: ..., mechanisms: [...], gaps: [...] }
}
```

**Progress**: Mark `Standards analysis (pass 2/2)` as done.

### Step 5 — Write docs.md

**Progress**: Display step status `Writing docs.md` as running.

Create `specs/` directory if it does not exist.

Using `STANDARDS_OUTLINE`, write the documentation file.

**If `WRITE_MODE = overwrite`**: Write the full document to `specs/docs.md` (replace any existing content).

**If `WRITE_MODE = amend`**: Preserve `EXISTING_DOCS` content exactly. Append the following at the end of the file:

```markdown
## Amendment — YYYY-MM-DD

> Re-analysis of `<ANALYSIS_ROOT>` performed on YYYY-MM-DD.

[For each section below, include only changes detected since the previous version.
If a section is unchanged, omit it from this amendment.]

### Updated Code Quality Principles
[Any new principles observed, updated examples, or resolved/new gaps]

### Updated Testing Standards
[Changes to test patterns, coverage, or gaps]

### Updated User Experience Consistency
[Changes to UX patterns or gaps]

### Updated Performance Requirements
[Changes to performance behaviors or gaps]

### Updated Governance
[New or revised governance rules]
```

**Document structure** (for overwrite mode — full document):

Write `specs/docs.md` with these sections in this exact order:

---

```markdown
# Project Documentation: <project name>

> Generated by `/speckit-docsme` on YYYY-MM-DD. Scope: <ANALYSIS_ROOT>.
> This document describes the project's current standards, principles, and governance.
> It is intended for developers new to the project and serves as the reference for code reviews.

## Project Overview

<Describe the project's purpose, primary capabilities, and target users derived from entry points,
module responsibilities, and configuration. 2–4 paragraphs. No implementation jargon —
write for a developer who has never seen the codebase.>

## Code Quality Principles

<For each identified quality principle from STANDARDS_OUTLINE.quality_principles:>

### <Principle Name>

<Description of the principle as observed in the codebase. Written for a new developer.
No class names or file paths in the description itself.>

**Observed in practice**: <Concrete example citing actual file paths or patterns. One or more bullet points.>

**Current gaps**: <Any deviations from this principle observed in the codebase. If none, write "None identified.">

<If constitution_ref is set: **See also**: [<constitution_ref>] in the Project Constitution.>

## Testing Standards

<Description of the testing approach, frameworks, coverage level, and discipline observed.>

**Frameworks and patterns in use**: <Bullet list>

**Approximate test-to-source ratio**: <value>

**Current gaps**: <Any areas lacking test coverage or deviating from the testing discipline. If none, write "None identified.">

<If constitution reference applies: **See also**: [<ref>] in the Project Constitution.>

## User Experience Consistency

<Description of the UX conventions observed — output formatting, progress indicators, error messages, interactive prompts.>

**Observed conventions**: <Bullet list>

**Current gaps**: <Inconsistencies or missing patterns. If none, write "None identified.">

<If constitution reference applies: **See also**: [<ref>] in the Project Constitution.>

## Performance Requirements

<Description of performance behaviors observed — caching, timeouts, offline support, I/O patterns.>

**Observed behaviors**: <Bullet list>

**Current gaps**: <Areas without performance handling. If none, write "None identified.">

<If constitution reference applies: **See also**: [<ref>] in the Project Constitution.>

## Governance

> These rules prescribe how the identified principles MUST guide future technical decisions,
> code reviews, and implementation choices. A developer facing an ambiguous decision
> SHOULD consult this section before escalating.

<If CONSTITUTION_PRINCIPLES is non-empty:>
> This project has a formal constitution at `.specify/memory/constitution.md`.
> The governance rules below are consistent with and complementary to the constitution.
> Where both apply, the constitution takes precedence.

<For each principle from STANDARDS_OUTLINE.quality_principles:>

### Governance: <Principle Name>

<One or more decision rules, each phrased as:>
- When <situation>, the team MUST/SHOULD <action>.

**Gaps requiring remediation**:
<List any gaps from this principle. Each gap becomes an obligation to address.
If no gaps: "None identified — principle is currently met across the codebase.">

<Repeat for Testing Standards, UX Consistency, and Performance Requirements:>

### Governance: Testing Standards

- When adding new functionality, the team MUST write tests before or alongside the implementation.
- [Additional rules derived from observed testing discipline and gaps.]

### Governance: User Experience Consistency

- When implementing any user-facing output, the team MUST follow the output formatting conventions observed in this document.
- [Additional rules derived from observed UX patterns and gaps.]

### Governance: Performance Requirements

- When adding network I/O or long-running operations, the team MUST implement caching and/or a timeout.
- [Additional rules derived from observed performance behaviors and gaps.]
```

---

**Governance section rules** (FR-004, FR-006, FR-008):
- The Governance section MUST open with an introductory paragraph explaining that these rules serve as code review criteria and decision-making guidance for technical choices — a developer facing an ambiguous decision SHOULD consult this section before escalating.
- For each identified principle (Code Quality, Testing, UX, Performance), write a dedicated `### Governance: <Principle Name>` subsection.
- Each subsection MUST contain ≥1 decision rule phrased as "When X, the team MUST/SHOULD Y." — where X is a concrete triggering situation and Y is the required or recommended action. Generic rules without a trigger situation are not acceptable.
- Each subsection MUST include a "Gaps requiring remediation" bullet that explicitly lists gaps observed during the Standards Pass. If no gaps exist, write "None identified — principle is currently met across the codebase." — do NOT omit the bullet (FR-008).
- If `CONSTITUTION_PRINCIPLES` is non-empty and a matching constitution principle exists, add a "See also:" line referencing the constitution principle by name (not by copying its text). This fulfils FR-006 — reference, not duplication.

**CRITICAL content rules** (FR-005, FR-008, FR-009):
- Every principle MUST cite ≥1 concrete codebase example (file path, function name, configuration value, or observed pattern).
- Gaps MUST be listed explicitly under each section — never omitted or silently excluded.
- All sections MUST be written in language accessible to a developer new to the project. Technical jargon MUST be explained. Implementation-specific identifiers (class names, file paths, function names) MAY appear only in "Observed in practice" examples, never as primary explanatory content.
- No template placeholder tokens MUST remain in the written file.

**Progress**: Mark `Writing docs.md` as done.

### Step 6 — Completion Report

**Progress**: Display all steps as done.

Output a completion summary:

```
[green]✓[/green] Documentation created: specs/docs.md
  Scope:      <ANALYSIS_ROOT>
  Mode:       <overwrite | amend>
  Principles: <count of quality principles identified>
  Sections:   6 (Project Overview, Code Quality Principles, Testing Standards,
                  User Experience Consistency, Performance Requirements, Governance)
  Gaps identified: <total gap count across all sections>
  Constitution: <"Aligned with constitution at .specify/memory/constitution.md" | "No constitution present — governance derived from codebase observation">
```

Suggest next steps:
- `[dim]Next: /speckit.constitution — codify identified principles into the project constitution[/dim]`
- `[dim]Or:   /speckit.plan — plan improvements to address identified gaps[/dim]`

## Post-Execution Checks

**Check for extension hooks (after documentation)**:
- Check if `.specify/extensions.yml` exists in the project root.
- If it exists, read it and look for entries under the `hooks.after_docsme` key
- If the YAML cannot be parsed or is invalid, skip hook checking silently and continue normally
- Filter out hooks where `enabled` is explicitly `false`. Treat hooks without an `enabled` field as enabled by default.
- For each remaining hook, do **not** attempt to interpret or evaluate hook `condition` expressions:
  - If the hook has no `condition` field, or it is null/empty, treat the hook as executable
  - If the hook defines a non-empty `condition`, skip the hook and leave condition evaluation to the HookExecutor implementation
- For each executable hook, output the following based on its `optional` flag:
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
- If no hooks are registered or `.specify/extensions.yml` does not exist, skip silently
