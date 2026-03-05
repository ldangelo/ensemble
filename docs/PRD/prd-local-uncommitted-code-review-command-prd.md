All in. Here's the complete updated PRD with all 18 findings addressed:

# PRD: Local Uncommitted Code Review Command

## Summary

Add a `/review-changes` command to the Ensemble plugin ecosystem that reviews local uncommitted code changes against five quality dimensions: code completeness, code complexity, tests, code coverage, and documentation. The command integrates into the existing `ensemble-quality` package and leverages Claude Code's AI capabilities to provide actionable, structured feedback in a single combined analysis pass before code is committed.

## Problem Statement

Developers currently lack a pre-commit quality gate within the Ensemble workflow. Code review happens after commits are pushed—often as part of a pull request—meaning defects in completeness, complexity, test coverage, and documentation are caught late in the cycle. This leads to:

- **Rework loops** from PR review feedback that could have been caught locally.
- **Inconsistent quality** when developers forget to self-review before committing.
- **Missing tests and docs** that slip through because there is no automated prompt to add them.
- **Complexity creep** that accumulates unnoticed across small, unreviewed changes.

A local review command closes this gap by shifting quality feedback left—to the moment between editing and committing.

## User Analysis

### Primary Users

| Persona | Description | Key Need |
|---------|-------------|----------|
| **Solo Developer** | Works alone on a project, no PR reviewer available | Automated second pair of eyes before commit |
| **Team Contributor** | Submits PRs that go through team review | Reduce PR round-trips by catching issues early |
| **Tech Lead** | Sets quality standards for the team | Enforce consistent review criteria across the team |

### User Environment

- Working inside Claude Code with Ensemble plugins installed.
- Using Git for version control (uncommitted changes available via `git diff` and `git status`).
- Node.js ≥ 20 runtime (per `package.json` engine requirements).
- May have test runners (Jest, Pytest, RSpec, etc.) configured via companion Ensemble testing plugins.

### Workflow Context

`/review-changes` fits into the developer workflow between editing and committing:

```
Edit code → /review-changes → Fix findings → git add → git commit → /review-code (PR)
```

**Relationship to existing `ensemble-quality` commands:**

- **`/review-code`** — Reviews code in the context of a pull request (post-commit, remote). `/review-changes` is the local, pre-commit counterpart.
- **DoD enforcement** — Definition of Done checks run at PR/task level. `/review-changes` provides earlier, more granular feedback on the uncommitted diff only.
- **`/create-trd` → `/implement-trd`** — Implementation workflows that produce code. `/review-changes` is a natural next step after implementation, before commit.

## Goals

1. **G1:** Provide a single command that reviews all staged and unstaged changes across the five quality dimensions in one combined AI analysis pass.
2. **G2:** Produce structured, actionable output—not vague suggestions—with file-level and line-level annotations.
3. **G3:** Integrate cleanly into the existing `ensemble-quality` package and follow Ensemble plugin conventions (manifest, agents, commands).
4. **G4:** Support incremental adoption—useful out of the box with zero configuration, but configurable for teams with specific thresholds.

## Non-Goals

- **NG1:** Replacing CI/CD pipelines or PR review tooling (this is a local, pre-commit aid).
- **NG2:** Automatically fixing code (the command reports; the developer acts).
- **NG3:** Running tests or test suites. The command never executes test runners. For coverage, it reads existing coverage reports only.
- **NG4:** Supporting non-Git version control systems in this iteration.

## Technical Architecture

### Package Integration

The command is added to the existing `ensemble-quality` package (`packages/quality/`) following Ensemble plugin conventions:

```
packages/quality/
├── plugin.json                  # Manifest — registers /review-changes command
├── agents/
│   └── review-changes-agent.md  # AI agent prompt for combined analysis
├── commands/
│   └── review-changes.md        # Command definition and parameter schema
└── lib/
    ├── change-detector.js       # Git diff/status collection
    ├── coverage-locator.js      # Coverage report discovery
    └── review-formatter.js      # Output formatting (text/JSON)
```

### Analysis Flow

1. **Change detection** (deterministic) — `change-detector.js` runs `git diff`, `git diff --cached`, and `git status` to collect all uncommitted changes.
2. **Coverage report lookup** (deterministic) — `coverage-locator.js` searches conventional paths for existing reports.
3. **Combined AI analysis** (single pass) — The collected diff, file metadata, and any coverage data are passed to the review agent in a single prompt. Claude analyzes all five dimensions together and returns structured findings.
4. **Output formatting** (deterministic) — `review-formatter.js` renders findings as a summary table plus detailed file-grouped annotations.

### Analysis Nature

All five review dimensions are **AI best-effort judgments**. Claude Code reads the diff and applies its understanding of the code, language conventions, and project context. Findings are recommendations, not deterministic static analysis guarantees.

## Dependencies and Prerequisites

| Dependency | Version | Required | Purpose |
|------------|---------|----------|---------|
| **Claude Code** | Active session | Yes | AI analysis engine; the command runs exclusively within Claude Code |
| **Git** | ≥ 2.25 | Yes | Change detection via `git diff`, `git diff --cached`, `git status` |
| **Node.js** | ≥ 20 | Yes | Plugin runtime (per `package.json` engine requirements) |
| **ensemble-core** | ≥ 5.0.0 | Yes | Plugin registration and command infrastructure |
| **Coverage report** | Any | No | Optional; lcov, cobertura, or Istanbul JSON if available |

## Functional Requirements

### FR-1: Change Detection

```gherkin
Feature: Detect uncommitted code changes

  Scenario: Staged and unstaged changes exist
    Given the developer has modified files tracked by Git
    When they run /review-changes
    Then the command collects all staged and unstaged diffs
    And identifies new, modified, and deleted files

  Scenario: No uncommitted changes
    Given the working tree is clean
    When they run /review-changes
    Then the command outputs "No uncommitted changes to review"
    And exits with code 0

  Scenario: Untracked files
    Given there are new files not yet added to Git
    When they run /review-changes
    Then untracked files are included in the review by default
    And the developer can exclude them with --tracked-only

  Scenario: Binary files in changeset
    Given the changeset includes binary files (images, compiled assets, etc.)
    When they run /review-changes
    Then binary files are skipped with an informational note
    And only text-based source files are analyzed

  Scenario: Very large diff
    Given the changeset exceeds 10,000 changed lines
    When they run /review-changes
    Then the command emits a warning that analysis may be incomplete
    And suggests using --only to narrow the review scope
    And proceeds with best-effort analysis

  Scenario: Merge conflict state
    Given the repository has unresolved merge conflicts
    When they run /review-changes
    Then the command outputs "Cannot review: unresolved merge conflicts detected"
    And exits with code 1
```

### FR-2: Code Completeness Review

```gherkin
Feature: Review code completeness

  Scenario: Incomplete implementation markers detected
    Given a changed file contains TODO, FIXME, or HACK comments
    When the completeness review runs
    Then each marker is reported with file path, line number, and surrounding context

  Scenario: Partial interface or abstract implementation
    Given a changed file implements an interface or abstract class
    When the completeness review runs
    Then the AI identifies any apparently unimplemented methods
    And flags them with the expected method signatures where inferable

  Scenario: Obviously unhandled error paths
    Given a changed function performs I/O, network, or async operations
    When the completeness review runs
    Then the AI flags obviously unhandled error paths using language-appropriate judgment
    And avoids prescribing specific patterns (e.g., does not mandate try/catch in languages with other error handling idioms)
```

### FR-3: Code Complexity Analysis

```gherkin
Feature: Review code complexity

  Scenario: High cyclomatic complexity
    Given a changed function has cyclomatic complexity above the threshold (default: 10)
    When the complexity review runs
    Then the function is flagged with its estimated complexity score
    And a suggestion to refactor is provided

  Scenario: Deep nesting
    Given a changed code block has nesting depth above the threshold (default: 4)
    When the complexity review runs
    Then the block is flagged with its nesting depth

  Scenario: Large function
    Given a changed function exceeds the line count threshold (default: 50 lines)
    When the complexity review runs
    Then the function is flagged with its line count
```

### FR-4: Test Review

```gherkin
Feature: Review test presence and quality

  Scenario: Changed source file has no corresponding test
    Given a source file in src/ or lib/ was modified
    And no test file exists matching the project's inferred test naming convention
    When the test review runs
    Then the file is flagged as "missing test file"

  Background: Test naming convention inference
    The AI infers the project's test naming convention from existing test files
    in the repository (e.g., *.test.ts, *.spec.js, test_*.py, *_test.go).
    If no existing tests are found, the AI uses language-standard conventions.

  Scenario: New public function lacks test coverage
    Given a new exported/public function was added
    And no test case appears to reference that function
    When the test review runs
    Then the function is flagged as "untested"

  Scenario: Test file was modified
    Given a test file was changed
    When the test review runs
    Then the test structure is reviewed for assertion presence and edge case coverage
```

### FR-5: Code Coverage Check

```gherkin
Feature: Review code coverage

  Scenario: Coverage report exists
    Given a coverage report exists in a conventional location
    When the coverage review runs
    Then changed files are cross-referenced against the report
    And files below the threshold (default: 80%) are flagged

  Background: Coverage report discovery
    The command searches the following paths in order:
      - coverage/lcov.info
      - coverage/cobertura.xml
      - coverage/coverage-final.json (Istanbul JSON format)
      - .nyc_output/
      - build/reports/cobertura.xml
    The first valid report found is used. The search path can be
    overridden via .ensemble/review.json configuration.

  Scenario: No coverage report available
    Given no coverage report is found in any conventional location
    When the coverage review runs
    Then a warning is emitted: "No coverage report found. Run your test suite with coverage enabled to get coverage analysis."
    And the AI suggests the appropriate coverage command based on detected test framework (e.g., "npx jest --coverage", "pytest --cov")
    And the dimension is marked as "skipped" in the summary
```

### FR-6: Documentation Review

```gherkin
Feature: Review documentation

  Scenario: Public API lacks JSDoc/docstring
    Given a new or modified exported function, class, or method exists
    And it has no documentation comment
    When the documentation review runs
    Then the symbol is flagged as "undocumented"

  Scenario: README not updated for new public-facing feature
    Given a new command, agent, plugin endpoint, or public API export was added
    And README.md was not modified in the changeset
    When the documentation review runs
    Then a suggestion is emitted to update the README
    Note: Internal module exports, utility functions, and private helpers
    do not trigger this check.

  Scenario: Stale documentation
    Given a function signature was changed (parameters added, removed, or renamed)
    And its existing docstring references old parameter names or outdated behavior
    When the documentation review runs
    Then the AI flags the stale documentation with specific details about the mismatch
```

### FR-7: Output and Reporting

```gherkin
Feature: Structured review output

  Scenario: Default output
    Given the review completes with findings
    When results are rendered
    Then a summary table shows pass/warn/fail per dimension
    And detailed findings are grouped by file
    And each finding includes severity (info/warning/error), location, and recommendation
    And the command exits with code 1

  Scenario: Clean review
    Given the review completes with no findings
    When results are rendered
    Then a summary shows all dimensions passed
    And the command exits with code 0

  Scenario: JSON output
    Given the developer passes --format json
    When results are rendered
    Then output is valid JSON matching the review output schema

  Scenario: Specific dimension only
    Given the developer passes --only complexity,tests
    When the review runs
    Then only the specified dimensions are evaluated
```

### FR-8: Configuration

```gherkin
Feature: Configurable thresholds

  Scenario: Project-level configuration
    Given an .ensemble/review.json file exists in the project root
    When the review runs
    Then thresholds and rules from the config file override defaults

  Scenario: Command-line overrides
    Given the developer passes --complexity-threshold 15
    When the review runs
    Then the complexity threshold is set to 15 for this run only
```

## Edge Cases and Error Handling

| Scenario | Behavior |
|----------|----------|
| **Binary files** | Skipped with informational note; only text source files analyzed |
| **Very large diffs (> 10k lines)** | Warning emitted; suggests `--only` to narrow scope; proceeds with best-effort |
| **Merge conflict state** | Refuses to run; exits with code 1 and actionable message |
| **Empty diff with untracked files** | Includes untracked files by default; `--tracked-only` to exclude |
| **Corrupted Git state** | Surfaces Git error message; exits with code 1 |
| **No `.git` directory** | Outputs "Not a Git repository"; exits with code 1 |
| **Permission errors on files** | Skips unreadable files with warning; reviews remaining files |

## Data Privacy

All analysis runs entirely within the active Claude Code session. The `/review-changes` command:

- **Does not** transmit source code to any service beyond what Claude Code already handles.
- **Does not** store review results outside the session unless the developer explicitly copies them.
- **Telemetry** (if `ensemble-metrics` is installed and opted-in) captures only aggregate usage data: invocation count, dimensions selected, finding counts. No source code, file paths, or finding content is captured.

## Non-Functional Requirements

| ID | Requirement | Target |
|----|-------------|--------|
| **NFR-1** | **Performance** — Single combined AI analysis pass for all dimensions. Review of ≤ 500 changed lines completes within 60 seconds. | p95 < 60s |
| **NFR-2** | **Compatibility** — Works with Node.js ≥ 20 and Git ≥ 2.25 | All supported environments |
| **NFR-3** | **Plugin conventions** — Follows Ensemble manifest schema (plugin.json), registers via standard command/agent patterns | Schema validation passes |
| **NFR-4** | **Extensibility** — Dimensions are modular; new review dimensions can be added without modifying existing ones | Plugin-internal SPI |
| **NFR-5** | **Testability** — Command logic (change detection, coverage locator, formatter) is covered by unit tests with ≥ 80% coverage | Measured via Jest |
| **NFR-6** | **Zero-config default** — Command is useful immediately after `ensemble-quality` installation with no project-level config | Works on any Git repo with an active Claude Code session |

## Rollout Plan

| Phase | Scope | Timeline |
|-------|-------|----------|
| **Phase 1: Core** | Change detection, completeness, complexity, and output formatting. JSON output. | v5.4.0 |
| **Phase 2: Testing dimensions** | Test review and coverage check (read-only). `--only` flag. | v5.5.0 |
| **Phase 3: Documentation** | Documentation review. Project-level configuration (`.ensemble/review.json`). | v5.6.0 |

Each phase ships as a minor version bump to `ensemble-quality`. Dimensions from later phases are hidden until their release — no feature flags needed since they simply don't exist in the agent prompt yet.

## Success Metrics

| Metric | Measurement Method | Target |
|--------|--------------------|--------|
| **Adoption** | Unique invocations per week (telemetry opt-in via ensemble-metrics) | 10% of active `ensemble-quality` users within 4 weeks of Phase 1 |
| **Issue catch rate** | % of review runs that surface ≥ 1 actionable finding | ≥ 60% |
| **PR rejection reduction** | Self-reported survey: "Did /review-changes reduce PR rework?" | ≥ 40% report yes |
| **Dimension coverage** | % of runs using all available dimensions (not filtered via --only) | ≥ 70% |
| **Latency** | p95 execution time for typical changeset (< 500 LOC diff) | < 60 seconds |
| **Developer satisfaction** | Post-use rating (1–5 scale) collected via optional prompt | ≥ 4.0 average |

---

*Package:* `ensemble-quality` · *Command:* `/review-changes` · *Ecosystem version:* 5.3.x · *Status:* Draft