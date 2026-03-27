---
description: "Gather context about the project and recommend the next best thing to work on."
---

# Pick Next Work

## Steps

1. **Gather Context**:
   - Read specs/plans/issues — check what's in progress, what's blocked
   - Run `git log --oneline -20` — see recent activity
   - Check `gh issue list` — open issues and their status
   - Run the build and tests — is the project healthy?
   - Check `.project/testing-register.md` — test coverage gaps

2. **Categorise Open Work**:
   - **Blocking**: build failures, test failures, broken pipeline
   - **Momentum**: current tasks that are in-progress
   - **Strategic**: next milestone that should start once current is done
   - **Debt**: test gaps, doc staleness, refactoring opportunities

3. **Recommend Top 3**:
   - Present 3 recommended next actions, prioritised
   - For each: what to do, why now, estimated scope (small/medium/large)
   - Always fix Blocking items first
