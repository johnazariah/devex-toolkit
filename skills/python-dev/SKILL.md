# Skill: Python Development

## Purpose

Write, review, and refactor Python code to be idiomatic, fully typed, and production-grade. This skill ensures AI agents produce Python that leverages type annotations, frozen dataclasses, Protocol-based Tagless-Final architecture, structural pattern matching, and comprehensions — rather than writing untyped, dict-heavy, mock-patched code.

## Configuration

| Setting | Description |
|---------|-------------|
| Standards Dir | `skills/python-dev/standards/` |
| Required Reading | `standards/idiomatic-python.md` — the complete Python idiom standard |
| Cross-reference | `skills/repo-onboard/standards/code-quality.md` — Tagless-Final architecture |

## Modes

This skill operates in three modes. Always read `standards/idiomatic-python.md` before beginning any mode.

---

### Mode 1: Write

Generate new Python code that conforms to the idiom standard from the start.

**Flow:**

1. **Understand the requirement.**
2. **Design types first.** `@dataclass(frozen=True, slots=True)` for domain types. `Protocol` for capabilities. `StrEnum` for fixed sets.
3. **Define capability protocols.** Tagless-Final Protocol for every external dependency.
4. **Write small functions.** ≤20 lines. Comprehensions for transforms. `asyncio.gather` for concurrency.
5. **🛑 Self-review gate.** Check every function against Quick Reference Card. Fix violations.

**Checklist (verify before output):**

- [ ] Every function fully annotated (parameters + return type)
- [ ] `pyright` strict mode config assumed
- [ ] All domain types are `@dataclass(frozen=True, slots=True)`
- [ ] `Protocol` classes for all external dependencies
- [ ] No `Any` types, no `type: ignore`, no `Optional[X]` (use `X | None`)
- [ ] List/dict/set comprehensions instead of accumulation loops
- [ ] `match/case` for all multi-branch dispatch with `assert_never` exhaustiveness
- [ ] `StrEnum` for string-valued fixed sets
- [ ] `Result[T, E]` for fallible operations
- [ ] No function exceeds 20 lines
- [ ] Fakes, not mocks (`unittest.mock.patch` prohibited)
- [ ] All containers parameterised: `list[str]`, `dict[str, int]`, `set[T]`
- [ ] `TypeGuard` functions for complex narrowing

---

### Mode 2: Review

Audit existing Python code against the idiom standard.

**Classification:**

| Severity | Meaning | Action |
|----------|---------|--------|
| 🔴 Critical | Untyped functions, `Any` types, bare `except`, mutable domain types | Must fix |
| 🟡 Major | Accumulation loops, magic strings, oversized functions, `mock.patch` | Should fix |
| 🟢 Minor | Style preferences (naming, walrus operator, Final constants) | Nice to fix |

---

### Mode 3: Refactor

Apply the idiom standard with minimal behavior change.

**Refactoring Recipes:**

| Anti-pattern | Recipe |
|-------------|--------|
| Untyped function | Add all annotations, run pyright strict |
| `dict` for structured data | `@dataclass(frozen=True, slots=True)` |
| Mutable dataclass | Add `frozen=True` |
| Accumulation loop building list | List comprehension |
| Accumulation loop building dict | Dict comprehension |
| Accumulation loop building set | Set comprehension |
| `isinstance()` chain | `match/case` + `assert_never` |
| `if/elif` cascade on types | `match/case` with class patterns |
| Dict key access chain for JSON | `match/case` with mapping patterns |
| `args[0]`, `args[1:]` slicing | `match/case` with sequence patterns |
| Magic strings | `StrEnum` |
| Bare `except` | Narrow exception + `Result` |
| `mock.patch` | Protocol + fake class |
| `**kwargs: Any` | Explicit params or `TypedDict` |
| `Optional[X]` / `Union[X, Y]` | `X \| None` / `X \| Y` |
| Global mutable state | `Final` constants or module-level frozen dataclass |

---

## Anti-pattern Catalog

1. **"Untyped Python"** — no annotations, `dict` everywhere, discovered at runtime.
2. **"Mutable soup"** — regular dataclasses with `list` fields mutated freely.
3. **"Mock everything"** — `@patch` on every import, testing implementation not behavior.
4. **"str-enum"** — `category = "bank-statements"` repeated across 5 files.
5. **"God function"** — 100-line function with nested `try/except` and mutable counters.
6. **"Any escape hatch"** — `value: Any` to bypass pyright, `type: ignore` to silence it.
