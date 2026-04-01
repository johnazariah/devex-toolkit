# Standard: Testing

## Testing Register

Every automated test must be catalogued in `.project/testing-register.md`.

### Format

```markdown
### Module_Function_Condition_ExpectedResult
- **Kind**: Unit | Property | Integration
- **File**: `tests/path/to/test.ext`
- **Intent**: Plain-English description of what this test verifies
```

### Summary Table

Keep a summary count at the top:

```markdown
| Category | Count |
|----------|-------|
| Unit     | 42    |
| Property | 15    |
| Integration | 8  |
| **Total** | **65** |
```

### Enforcement

The pre-commit hook rejects commits that modify test files without updating the testing register.

## Test Naming

Use `Module_Function_Condition_ExpectedResult` or equivalent for your language:

| Language | Convention | Example |
|----------|-----------|---------|
| F# / C# | `Module_Function_Condition_ExpectedResult` | `Config_Load_MissingFile_ReturnsDefaults` |
| Python | `test_module_function_condition_expected` | `test_config_load_missing_file_returns_defaults` |
| TypeScript | `describe/it` blocks | `describe('Config') > it('returns defaults when file missing')` |

## Test Categories

Tag tests with categories for selective execution:

| Category | What | When to Run |
|----------|------|-------------|
| Unit | Pure logic, no I/O | Every commit (pre-commit) |
| Property | Randomised invariant checks (FsCheck, Hypothesis) | Every commit |
| Integration | External dependencies (DB, API, filesystem) | Pre-push, CI |
| E2E | Full system tests | CI only |

## Coverage

**Mandatory floor: 85% line coverage, 85% branch coverage.**

- New code must maintain or improve coverage — never drop below the floor.
- Track coverage in CI (Coverlet, pytest-cov, istanbul, etc.).
- Enforce in the pre-commit hook — commits that drop coverage below 85% are rejected.
- Report coverage in CI summary (badge in README).
- Do not mark a task as complete if it drops coverage below the current level.

| Tool | Language | Command |
|------|----------|---------|
| Coverlet | .NET (F#/C#) | `dotnet test --collect:"XPlat Code Coverage"` |
| pytest-cov | Python | `pytest --cov=src --cov-report=term-missing` |
| istanbul/c8 | TypeScript | `vitest run --coverage` |
