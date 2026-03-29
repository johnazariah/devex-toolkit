# Skill: C# Development

## Purpose

Write, review, and refactor C# code to be idiomatic, modern (C# 12+), and production-grade. This skill ensures AI agents produce C# that leverages records, primary constructors, pattern matching, LINQ, non-nullability, and Tagless-Final architecture — rather than writing legacy ceremony-heavy OOP.

## Configuration

| Setting | Description |
|---------|-------------|
| Standards Dir | `skills/csharp-dev/standards/` |
| Required Reading | `standards/idiomatic-csharp.md` — the complete C# idiom standard |
| Cross-reference | `skills/repo-onboard/standards/code-quality.md` — Tagless-Final architecture |

## Modes

This skill operates in three modes. Always read `standards/idiomatic-csharp.md` before beginning any mode.

---

### Mode 1: Write

Generate new C# code that conforms to the idiom standard from the start.

**Flow:**

1. **Understand the requirement.** Read the spec, issue, or user description.
2. **Design types first.** Define records, enums, and interfaces before writing any logic. Types are documentation.
3. **Define capability interfaces.** For any external dependency, define a Tagless-Final interface before writing the implementation.
4. **Write small methods.** Each method ≤20 lines. Use local functions for readability. LINQ for data transformations.
5. **Compose.** Wire implementations at the composition root. Use DI for capability injection.
6. **🛑 Self-review gate.** Before presenting code, check every method against the Quick Reference Card in the standard. Fix violations before showing the user.

**Checklist (verify before output):**

- [ ] All data types are `record` or `readonly record struct`
- [ ] All classes are `sealed` unless designed for inheritance
- [ ] No settable properties on domain types — `init`-only or positional parameters
- [ ] No `out`/`ref` parameters — tuples or records instead
- [ ] No method exceeds 20 lines
- [ ] LINQ used for filter/map/reduce; `foreach` only for side effects
- [ ] Pattern matching (`switch` expressions) instead of `if/else` chains
- [ ] No magic strings — enums for fixed sets
- [ ] Every async method takes `CancellationToken` as last parameter
- [ ] `Nullable` enabled, no `!` suppressions
- [ ] Capability interfaces follow Tagless-Final pattern

---

### Mode 2: Review

Audit existing C# code against the idiom standard. Produce actionable findings, not vague advice.

**Flow:**

1. **Read the file(s) under review.** Understand purpose before judging style.
2. **Check each rule in `standards/idiomatic-csharp.md`** systematically.
3. **Classify findings:**

| Severity | Meaning | Action |
|----------|---------|--------|
| 🔴 Critical | Null safety violation, exposed mutable collection, missing CancellationToken | Must fix |
| 🟡 Major | Non-record data type, magic strings, oversized method, `out` parameter | Should fix |
| 🟢 Minor | Style preference (naming, collection expression, using declaration) | Nice to fix |

4. **For each finding, provide:**
   - File and line range
   - Rule violated (by number from the standard)
   - Concrete before/after code snippet
   - Estimated effort (trivial / small / medium)

5. **Produce a summary table.**

6. **🛑 Priority gate.** Present 🔴 findings first.

---

### Mode 3: Refactor

Apply the idiom standard to existing code with **minimal behavior change**.

**Flow:**

1. **Start from a Review.** Either run Mode 2 first, or receive a review report.
2. **Prioritise:** 🔴 → 🟡 → 🟢. One finding at a time.
3. **For each refactoring:**
   a. Read the existing code thoroughly.
   b. Plan the change — what records to introduce, what methods to extract.
   c. Apply the change.
   d. Verify: `dotnet build` passes, `dotnet test` passes.
   e. **🛑 Stop gate:** If tests fail, fix the regression before continuing.
4. **Preserve public API.** Keep method signatures unless explicitly asked to change them.
5. **One concept per commit.** `refactor(Module): description`.

**Refactoring Recipes:**

| Anti-pattern | Recipe |
|-------------|--------|
| Mutable class with set properties | Convert to `record` with positional parameters |
| `out`/`ref` parameters | Return tuple or named record |
| `if/else if/else` chain | `switch` expression with pattern matching |
| Oversized method | Extract local or private methods |
| Magic strings | Introduce `enum` + extension methods |
| `List<T>` exposed publicly | Return `IReadOnlyList<T>` |
| Concrete dependency in constructor | Extract interface, use Tagless-Final |
| `catch (Exception) { }` | Narrow catch, convert to `Result<T>` at boundary |
| Missing `sealed` | Add `sealed` to all non-inherited classes/records |

---

## Anti-pattern Catalog

The most common agent mistakes when writing C#:

1. **"Java in C# syntax"** — verbose class hierarchies, getter/setter beans, no records or pattern matching.
2. **"Mutable everything"** — `List<T>` with `set` accessors everywhere, no immutability.
3. **"Null anxiety"** — `!` suppressions scattered through code, or defensive `if (x != null)` instead of `x is { }`.
4. **"Ceremony OOP"** — abstract base class → interface → adapter → factory → provider, when a simple record + function suffices.
5. **"Missing async flow"** — no `CancellationToken`, `.Result` or `.Wait()` calls instead of `await`.
6. **"Stringly typed"** — configuration keys, categories, and status values as raw strings instead of enums.

These are all addressed by specific rules in `standards/idiomatic-csharp.md`.
