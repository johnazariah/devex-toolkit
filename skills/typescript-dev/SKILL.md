# Skill: TypeScript Development

## Purpose

Write, review, and refactor TypeScript code to be idiomatic, type-safe, and production-grade. This skill ensures AI agents produce TypeScript that leverages strict mode, discriminated unions, readonly data, Result types, and Tagless-Final architecture — rather than writing loose JavaScript with type annotations bolted on.

## Configuration

| Setting | Description |
|---------|-------------|
| Standards Dir | `skills/typescript-dev/standards/` |
| Required Reading | `standards/idiomatic-typescript.md` — the complete TS idiom standard |
| Cross-reference | `skills/repo-onboard/standards/code-quality.md` — Tagless-Final architecture |

## Modes

This skill operates in three modes. Always read `standards/idiomatic-typescript.md` before beginning any mode.

---

### Mode 1: Write

Generate new TypeScript code that conforms to the idiom standard from the start.

**Flow:**

1. **Understand the requirement.** Read the spec, issue, or user description.
2. **Design types first.** Define interfaces, discriminated unions, branded types, and Result types before writing logic.
3. **Define capability interfaces.** Tagless-Final interface for every external dependency.
4. **Write small functions.** Each function ≤20 lines. Compose via array methods, `Promise.all`, and pure helpers.
5. **🛑 Self-review gate.** Check every function against the Quick Reference Card. Fix violations.

**Checklist (verify before output):**

- [ ] `strict: true` assumed in tsconfig
- [ ] All interface properties are `readonly`
- [ ] No `any`, no `as` casts (use type guards / `satisfies` / `unknown` narrowing)
- [ ] No `export default` — named exports only
- [ ] Discriminated unions for polymorphic types with `kind` discriminant
- [ ] Exhaustive `switch` with `assertNever()` or `satisfies never` in `default`
- [ ] `Result<T, E>` for fallible operations; throw only for exceptional conditions
- [ ] No function exceeds 20 lines
- [ ] `const` by default; `let` only when reassignment is necessary
- [ ] Array methods (`.map`, `.filter`, `.flatMap`, `.reduce`) instead of accumulation loops
- [ ] Custom type guards (`(x): x is T =>`) for complex narrowing
- [ ] `.filter(isNonNullable)` with type guard for null removal
- [ ] Fakes built from interfaces, not `jest.mock()`
- [ ] `AbortSignal` parameter on I/O functions

---

### Mode 2: Review

Audit existing TypeScript code against the idiom standard.

**Classification:**

| Severity | Meaning | Action |
|----------|---------|--------|
| 🔴 Critical | `any` types, `as` casts, mutable exported state, missing strict mode | Must fix |
| 🟡 Major | Non-readonly properties, magic strings, oversized functions, `export default` | Should fix |
| 🟢 Minor | Style preferences (naming, `??` vs `\|\|`, const enum style) | Nice to fix |

---

### Mode 3: Refactor

Apply the idiom standard with minimal behavior change. Same prioritized flow as other language skills.

**Refactoring Recipes:**

| Anti-pattern | Recipe |
|-------------|--------|
| `any` type | `unknown` + type guard function |
| `as` cast | `satisfies` or type narrowing |
| Mutable properties | Add `readonly` to all interface fields |
| `export default` | Convert to named export |
| `if/else` on string literal | Discriminated union + exhaustive `switch` |
| Non-exhaustive `switch` | Add `default: assertNever(x)` |
| Thrown error for expected failure | Return `Result<T, E>` |
| `jest.mock()` | Tagless-Final interface + fake object |
| Magic strings | `as const` object + type extraction |
| Accumulation `for` loop | `.map()` / `.filter()` / `.flatMap()` |
| Nested `for` flattening | `.flatMap()` |
| Manual null filtering | `.filter(isNonNullable)` with type guard |
| `typeof` chain for validation | Custom type guard `(x): x is T =>` |
| Oversized function | Extract pure helper functions, compose |

---

## Anti-pattern Catalog

1. **"JavaScript with types"** — `any` everywhere, `as` casts, no discriminated unions.
2. **"Mutation city"** — `push()`, `splice()`, property reassignment on shared objects.
3. **"Mock everything"** — `jest.mock()` for every import instead of injecting capabilities.
4. **"String union soup"** — `type Status = 'a' | 'b' | 'c'` without exhaustive handling.
5. **"God module"** — 500+ line file with mixed concerns, barrel re-exporting everything.
6. **"any escape hatch"** — `(value as any).foo` to bypass the type system.
