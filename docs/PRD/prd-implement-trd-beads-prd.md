Now let me look at the existing codebase to understand the implement-trd-enhanced command:Now I have a thorough understanding of the existing command and the beads CLI. Let me create the PRD.I have all the context I need. Let me write the PRD.The parentheses in the JSON are causing shell issues. Let me write it to a file first.The API endpoint requires authentication that isn't available. Let me output the PRD directly instead.---

# PRD: implement-trd-beads

**Version:** 1.0.0  
**Date:** 2026-03-05  
**Status:** Draft

---

## 1. Summary

Clone the existing `implement-trd-enhanced` command (v4.2.0) and replace its custom `.trd-state/` file-based progress tracking with the [Beads CLI (`bd`)](https://github.com/steveyegge/beads) — a distributed, git-backed graph issue tracker designed for AI agents. The new command, `implement-trd-beads`, retains all execution logic (strategy detection, specialist delegation, quality gates, resume support) while using Beads as the single source of truth for task state, dependencies, and progress.

This gives the ensemble ecosystem a production-grade, version-controlled task tracker with dependency graphs, hierarchical IDs, atomic claiming, and multi-agent conflict prevention — capabilities the current JSON state file lacks.

---

## 2. Problem Statement

The current `implement-trd-enhanced` command tracks task progress via a flat JSON file (`.trd-state/<trd-name>/implement.json`). This approach has several limitations:

1. **No dependency graph** — Task dependencies are parsed from the TRD but stored as simple strings, not a queryable graph. There is no way to ask "what is ready to work on?" without re-parsing.
2. **No atomic claiming** — When parallel tasks run, there is no lock or atomic claim mechanism; concurrent writes to the JSON file can race.
3. **No audit trail** — The state file records final status but not the history of state transitions (claimed → in_progress → done).
4. **No cross-branch visibility** — State lives in an untracked directory; progress is invisible to other branches or collaborators.
5. **Fragile resume** — Resume depends on a SHA-256 hash match of the TRD file. Any formatting change invalidates state, even if tasks haven't changed.

Beads solves all five problems natively: it provides a dependency-aware graph (`bd dep add`), atomic task claiming (`bd update --claim`), a full audit trail (`bd show`), git-backed persistence, and hash-based IDs that survive file edits.

---

## 3. User Analysis

### Primary Users
- **AI coding agents** (Claude Code, Codex, etc.) executing TRD implementations via the ensemble plugin system
- **Developers** invoking `/implement-trd-beads` to orchestrate multi-phase feature builds

### Secondary Users
- **Team leads / PMs** reviewing task progress via `bd ready`, `bd show`, and Beads web UIs
- **Multi-agent setups** where multiple agents work on different phases or branches simultaneously

### User Needs

| Need | Current State | With Beads |
|------|--------------|------------|
| See what's ready to work on | Re-parse TRD + JSON | `bd ready` |
| Resume after interruption | Hash-match or manual reset | `bd ready` returns unclaimed tasks |
| Parallel agent safety | No protection | `bd update --claim` is atomic |
| Progress visibility | Read JSON file | `bd show`, git log, Dolt queries |
| Dependency management | Implicit in TRD ordering | Explicit graph: `bd dep add` |

---

## 4. Goals and Non-Goals

### Goals
1. **G1:** Create `implement-trd-beads.md` command that mirrors all functionality of `implement-trd-enhanced.md`.
2. **G2:** Replace `.trd-state/` JSON tracking with Beads CLI (`bd`) commands for task creation, status updates, dependency tracking, and progress queries.
3. **G3:** Use Beads hierarchical IDs to model the TRD's phase/task/subtask hierarchy (e.g., `bd-a3f8` for phase epic, `bd-a3f8.1` for task).
4. **G4:** Use `bd ready` as the primary mechanism for determining which tasks to execute next, replacing manual dependency resolution.
5. **G5:** Use `bd update --claim` for atomic task claiming to enable safe parallel execution.
6. **G6:** Maintain full backward compatibility with existing TRD file format — no changes required to TRDs.
7. **G7:** Support resume via Beads state (query open/in-progress tasks) instead of JSON hash matching.

### Non-Goals
- **NG1:** Modifying the `implement-trd-enhanced` command. It remains unchanged.
- **NG2:** Requiring Beads to be pre-installed. The command should detect and install `bd` if missing.
- **NG3:** Changing specialist delegation logic, strategy detection, or quality gate behavior.
- **NG4:** Building a custom UI for Beads progress visualization.
- **NG5:** Supporting Beads Dolt remote sync (local-only for v1).

---

## 5. Functional Requirements

### FR-1: Command File Structure

```gherkin
Feature: Command definition
  Scenario: Command file exists in ensemble development package
    Given the ensemble-plugins repository
    When I look in packages/development/commands/ensemble/
    Then I find "implement-trd-beads.md"
    And it has frontmatter with name "implement-trd-beads"
    And it has version "1.0.0"
    And it has category "implementation"
```

### FR-2: Beads Initialization

```gherkin
Feature: Beads project initialization
  Scenario: First run in a project without Beads
    Given a project without a .beads/ directory
    When implement-trd-beads starts the Preflight step
    Then it runs "bd init" in the project root
    And Beads is initialized for the project

  Scenario: First run in a project that already has Beads
    Given a project with an existing .beads/ directory
    When implement-trd-beads starts the Preflight step
    Then it skips "bd init"
    And uses the existing Beads instance

  Scenario: Beads CLI not installed
    Given the "bd" command is not found on PATH
    When implement-trd-beads starts
    Then it installs Beads via "npm install -g @beads/bd"
    And retries initialization
```

### FR-3: TRD Task Import to Beads

```gherkin
Feature: Import TRD tasks into Beads
  Scenario: Parse TRD and create Beads issues
    Given a TRD file with a "Master Task List" section
    And tasks formatted as "- [ ] **TRD-XXX**: Description"
    When implement-trd-beads parses the TRD
    Then it creates a Beads epic for each Phase heading
    And creates a Beads task under the epic for each TRD task
    And the Beads task title includes the TRD-XXX identifier
    And the Beads task description includes acceptance criteria

  Scenario: Map TRD dependencies to Beads
    Given TRD-002 has "Depends: TRD-001"
    When tasks are imported into Beads
    Then "bd dep add" links TRD-002 as blocked by TRD-001
    And TRD-002 will not appear in "bd ready" until TRD-001 is done

  Scenario: Already-checked tasks are marked done
    Given TRD-003 is marked "- [x]" in the TRD
    When tasks are imported into Beads
    Then the corresponding Beads issue is created with status "done"

  Scenario: Re-import detects existing Beads tasks
    Given tasks were previously imported for this TRD
    When implement-trd-beads runs again
    Then it matches existing Beads tasks by TRD-XXX in the title
    And does not create duplicates
    And updates any changed descriptions or dependencies
```

### FR-4: Task Execution with Beads Tracking

```gherkin
Feature: Execute tasks using Beads for state
  Scenario: Select next task
    Given a phase is being executed
    When the command needs the next task to execute
    Then it runs "bd ready" to get tasks with no open blockers
    And selects the highest-priority ready task

  Scenario: Claim a task before execution
    Given a ready task with Beads ID bd-xxxx
    When a specialist is about to start work
    Then the command runs "bd update bd-xxxx --claim"
    And the task status becomes "in_progress"

  Scenario: Mark task complete
    Given a specialist has completed task bd-xxxx
    When deliverables are confirmed
    Then the command runs "bd update bd-xxxx --status done"
    And updates the TRD file checkbox: "- [ ]" → "- [x]"
    And commits with message "<type>(TRD-XXX): <description>"

  Scenario: Mark task failed
    Given a specialist fails to complete task bd-xxxx
    When the failure is confirmed after debug retries
    Then the command runs "bd update bd-xxxx --status blocked"
    And records the error in Beads task comments
```

### FR-5: Resume via Beads

```gherkin
Feature: Resume interrupted implementation
  Scenario: Resume after interruption
    Given a previous run was interrupted
    When the user runs implement-trd-beads with "resume"
    Then the command queries Beads for tasks with status "in_progress" or "open"
    And resumes from the first incomplete phase
    And does not re-execute tasks with status "done"

  Scenario: Resume with TRD changes
    Given tasks exist in Beads from a prior run
    And the TRD file has been modified (new tasks added)
    When the user runs implement-trd-beads with "resume"
    Then new TRD tasks not in Beads are imported
    And existing Beads tasks are not modified
    And the user is informed of newly added tasks
```

### FR-6: Phase Quality Gates (Unchanged Logic)

```gherkin
Feature: Quality gates use Beads for phase tracking
  Scenario: Phase completion check
    Given all tasks in Phase N have Beads status "done"
    When the quality gate runs
    Then test execution, debug loops, and coverage checks
      behave identically to implement-trd-enhanced
    And phase completion is recorded as a Beads milestone comment

  Scenario: Phase checkpoint commit
    Given quality gate passes for Phase N
    Then a checkpoint commit is created
    And Beads task states are committed with the checkpoint
```

### FR-7: Completion Report

```gherkin
Feature: Final completion report
  Scenario: All tasks done
    Given all TRD tasks have Beads status "done"
    When the completion step runs
    Then a summary report is displayed
    And the report includes task counts from Beads
    And Beads compaction is run to summarize closed tasks
    And the format matches implement-trd-enhanced output
```

### FR-8: Preserved Behaviors

```gherkin
Feature: All implement-trd-enhanced behaviors preserved
  Scenario Outline: Feature parity
    Given implement-trd-enhanced supports <feature>
    When using implement-trd-beads
    Then <feature> behaves identically

    Examples:
      | feature                          |
      | Constitution loading             |
      | Strategy auto-detection          |
      | Strategy override via arguments  |
      | Branch creation via git town     |
      | Specialist selection by keywords |
      | Skill matching via router-rules  |
      | File conflict detection          |
      | Parallel task execution (max 2)  |
      | Debug loop with max 2 retries    |
      | Coverage threshold checking      |
      | Security scan delegation         |
      | TRD checkbox updates             |
      | Conventional commit messages     |
```

---

## 6. Non-Functional Requirements

| ID | Requirement | Acceptance Criteria |
|----|-------------|---------------------|
| NFR-1 | **Performance** | Beads operations (`bd ready`, `bd update`) complete in < 2 seconds each. Total overhead vs implement-trd-enhanced < 30 seconds per phase. |
| NFR-2 | **Compatibility** | Works on macOS, Linux, and Windows (same platforms as Beads). |
| NFR-3 | **Beads Version** | Targets Beads CLI v1.x (current stable). Minimum version documented in command frontmatter. |
| NFR-4 | **Git Cleanliness** | Beads files (`.beads/`) committed alongside code changes, not left as untracked artifacts. |
| NFR-5 | **Idempotency** | Running the command multiple times on the same TRD does not create duplicate Beads tasks or corrupt state. |
| NFR-6 | **Graceful Degradation** | If `bd` commands fail (e.g., corrupt Beads DB), reports error clearly and suggests `bd init --force` recovery. |
| NFR-7 | **Documentation** | Includes inline usage documentation matching the ensemble command format. |
| NFR-8 | **No TRD Format Changes** | Existing TRD files work without modification. |

---

## 7. Success Metrics

| Metric | Target | Measurement |
|--------|--------|-------------|
| **Feature parity** | 100% of implement-trd-enhanced scenarios pass with beads variant | Run both commands against the same TRD; compare outcomes |
| **Resume reliability** | Resume works after TRD edits without manual state reset | Test: modify TRD between runs, verify resume succeeds |
| **Parallel safety** | Zero race conditions in 10 consecutive parallel-task runs | Run with `max parallel 2` on tasks touching different files |
| **Task visibility** | `bd ready` accurately reflects next executable tasks at every step | Audit bd ready output at each phase transition |
| **Overhead** | < 5% wall-clock time increase vs implement-trd-enhanced | Benchmark both commands on a 20-task TRD |
| **Adoption** | Command is usable without prior Beads knowledge | New user can run `/implement-trd-beads` with zero Beads setup |

---

## Appendix A: Beads CLI Command Mapping

| Current (JSON state) | New (Beads CLI) |
|---------------------|-----------------|
| Create `.trd-state/` dir | `bd init` |
| Write task to JSON | `bd create "TRD-XXX: desc" -p <priority>` |
| Record dependency | `bd dep add <child> <parent>` |
| Find next task | `bd ready` |
| Set task in-progress | `bd update <id> --claim` |
| Set task complete | `bd update <id> --status done` |
| Set task failed | `bd update <id> --status blocked` |
| Read task status | `bd show <id>` |
| Check TRD hash for resume | Query Beads for open/in-progress tasks |
| Phase checkpoint | Commit includes `.beads/` state |
| Completion summary | `bd` (list all) + compaction |

## Appendix B: Hierarchical ID Example

```
bd-a3f8          → Phase 1 (Epic)
  bd-a3f8.1      → TRD-001: Create user model
  bd-a3f8.2      → TRD-002: Add authentication service
  bd-a3f8.3      → TRD-003: Build login endpoint
bd-c7e2          → Phase 2 (Epic)
  bd-c7e2.1      → TRD-004: Create dashboard component
  bd-c7e2.2      → TRD-005: Add data fetching hooks
```