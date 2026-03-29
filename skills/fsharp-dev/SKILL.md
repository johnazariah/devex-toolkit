# Skill: F# Development

## Purpose

Write, review, and refactor F# code to be idiomatic, functional-first, and production-grade. This skill ensures AI agents produce F# that leverages the language's strengths — pipelines, composition, type inference, discriminated unions — rather than writing "C# in F# syntax."

## Configuration

| Setting | Description |
|---------|-------------|
| Standards Dir | `skills/fsharp-dev/standards/` |
| Required Reading | `standards/idiomatic-fsharp.md` — the complete F# idiom standard |
| Cross-reference | `skills/repo-onboard/standards/code-quality.md` — Tagless-Final architecture |

## Modes

This skill operates in three modes. Always read `standards/idiomatic-fsharp.md` before beginning any mode.

---

### Mode 1: Write

Generate new F# code that conforms to the idiom standard from the start.

**Flow:**

1. **Understand the requirement.** Read the spec, issue, or user description.
2. **Identify the module boundary.** Where does this code live in the `.fsproj` compilation order? What types does it depend on? What types does it produce?
3. **Design types first.** Define records, DUs, and type aliases before writing any logic. Types are documentation.
4. **Write small functions.** Each function ≤20 lines. Each `task {}` block ≤15 lines. Pipeline where possible.
5. **Compose.** Wire the small functions together via `|>`, `Result.bind`, `TaskResult.bind`, or short `task {}` blocks that call named functions.
6. **🛑 Self-review gate.** Before presenting code, check every function against the Quick Reference Card in the standard. Fix violations before showing the user.

**Checklist (verify before output):**

- [ ] No function exceeds 20 lines
- [ ] No `task {}` block exceeds 15 lines or has nested `task {}`
- [ ] No `mutable` outside of Prelude combinators
- [ ] Pipelines used where data flows linearly
- [ ] `Option.map`/`bind`/`defaultValue` used instead of explicit match (where ≤2 branches)
- [ ] No magic strings — DUs for fixed sets
- [ ] Named records for return types with >2 fields
- [ ] Module has single responsibility, ≤150 lines target
- [ ] `[<RequireQualifiedAccess>]` on modules unless there's a reason not to

---

### Mode 2: Review

Audit existing F# code against the idiom standard. Produce actionable findings, not vague advice.

**Flow:**

1. **Read the file(s) under review.** Understand what they do before judging how they do it.
2. **Check each rule in `standards/idiomatic-fsharp.md`** systematically. Don't cherry-pick.
3. **Classify findings:**

| Severity | Meaning | Action |
|----------|---------|--------|
| 🔴 Critical | Causes compilation/perf issues (e.g., 200-line task block → FS3511) | Must fix |
| 🟡 Major | Violates core idiom (mutable counters, missing DU, nested task) | Should fix |
| 🟢 Minor | Style preference (naming, pipeline vs let-binding for clarity) | Nice to fix |

4. **For each finding, provide:**
   - File and line range
   - Rule violated (by number from the standard)
   - Concrete before/after code snippet
   - Estimated effort (trivial / small / medium)

5. **Produce a summary table:**

```markdown
| # | File | Lines | Rule | Severity | Effort | Description |
|---|------|-------|------|----------|--------|-------------|
| 1 | EmailSync.fs | 160-345 | 1,3 | 🔴 | medium | 185-line function with mutable counters |
```

6. **🛑 Priority gate.** Present the 🔴 findings first. Don't overwhelm with 🟢 items if there are unresolved 🔴s.

---

### Mode 3: Refactor

Apply the idiom standard to existing code with **minimal behavior change**. Refactoring must not alter observable behavior.

**Flow:**

1. **Start from a Review.** Either run Mode 2 first, or receive a review report.
2. **Prioritise:** 🔴 → 🟡 → 🟢. One finding at a time.
3. **For each refactoring:**
   a. Read the existing code thoroughly.
   b. Plan the refactoring — what functions to extract, what types to introduce.
   c. Apply the change.
   d. Verify: does the code still compile? Do existing tests still pass?
   e. **🛑 Stop gate:** If tests fail, fix the regression before moving to the next finding.
4. **Preserve public API.** If the function is called from outside the module, keep its signature. Extract internal helpers, don't rename public functions.
5. **One concept per commit.** Each refactoring should be a single, reviewable commit with a conventional commit message: `refactor(module): description`.

**Refactoring Recipes:**

| Anti-pattern | Recipe |
|-------------|--------|
| Oversized function | Extract sub-functions, compose with `\|>` or `>>`  |
| Mutable counters | Define accumulator record, use `fold` or `foldTask` |
| Nested `task { task { } }` | Extract inner block as named function |
| Explicit `match` on Option/Result (2 branches) | Replace with `map`/`bind`/`defaultValue` |
| Magic strings | Define DU + companion module with `toString`/`tryParse` |
| Large module (200+ lines) | Identify concepts, extract to new modules, update `.fsproj` order |
| `while` + mutable | Recursive function with accumulator, or `Seq.unfold` |
| Repeated row-reading pattern | Introduce `RowReader` type (see standard Rule 8) |

---

## Integration with Tagless-Final

This skill complements the Tagless-Final architecture defined in `code-quality.md`. The architecture rule says _what_ to abstract; this skill says _how_ to write the implementations idiomatically.

- Algebra records → defined in `Algebra.fs`, following Rule 7 (module organisation).
- Concrete implementations → each in its own module, following all idiom rules.
- Fakes for testing → simple record values, no mocking frameworks (Rule 10).
- Composition root → wires concrete implementations, kept minimal.

## Anti-pattern Catalog

The standard was derived from real anti-patterns found in production F# codebases. The most common agent mistakes:

1. **"C# in F# syntax"** — imperative loops, mutable everywhere, no pipelines.
2. **"One giant task block"** — 100+ line `task {}` that causes FS3511 and 220-second compilation.
3. **"Stringly typed"** — magic strings instead of DUs, repeated across files.
4. **"Match everything"** — explicit `match` on Option/Result where `map`/`bind` suffices.
5. **"God module"** — 500+ line file mixing I/O, pure logic, helpers, and constants.
6. **"Anonymous returns"** — returning `string * int * bool` instead of a named record.

These are all addressed by specific rules in `standards/idiomatic-fsharp.md`.
