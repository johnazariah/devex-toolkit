# {{PROJECT_NAME}} — Phase Dependency Graph

> Machine-readable phase definitions for automated planning.
> An agent reads this file to determine what can run in parallel and what must go first.

## Phases

| Phase | Name | Branch | Spec | Depends On | Status |
|-------|------|--------|------|-----------|--------|
| 0 | {{PHASE_0_NAME}} | `feat/0-{{phase-0-slug}}` | [spec](docs/specs/phase-0-{{slug}}.md) | — | Not Started |

<!-- Add more phases as needed. The agent reads this table to plan execution waves. -->

## Execution Waves

Phases within a wave can be implemented in parallel. Each wave requires all previous waves to be merged.

```
Wave 1:  [0]                          ← foundation
```

<!-- Update as phases are added. Example:
Wave 1:  [0]
Wave 2:  [1] [2]
Wave 3:  [3]
-->

## Agent Instructions

To implement a phase:

1. Read this file to understand dependencies and confirm all prerequisites are merged
2. Read the phase spec linked in the table above
3. Read `.github/copilot-instructions.md` for conventions
4. Create the feature branch listed in the table
5. Implement all tasks in the spec
6. Run the build and test commands
7. Update `.project/testing-register.md` with any new tests
8. Update this file — set the phase status to "Done"
9. Update `.project/STATUS.md` with current state
10. Commit using `.github/prompts/commit.prompt.md`
11. Open a PR using `.github/prompts/pr-prep.prompt.md`
