# Standard: Code Quality

## EditorConfig

Every repo must have a `.editorconfig`. Base rules:

```ini
root = true

[*]
indent_style = space
indent_size = 4
end_of_line = lf
charset = utf-8
trim_trailing_whitespace = true
insert_final_newline = true

[*.md]
trim_trailing_whitespace = false
```

### Language-Specific Overrides

| Language | Indent | Additional |
|----------|--------|-----------|
| F# / C# | 4 spaces | |
| Python | 4 spaces | |
| TypeScript / JavaScript | 2 spaces | |
| YAML / JSON | 2 spaces | |
| XML / csproj / fsproj | 2 spaces | |
| Makefile | tabs | |
| Go | tabs | |

## Line Endings

**LF everywhere.** Enforced by `.gitattributes`:

```
* text=auto eol=lf
```

Plus language-specific extensions marked as `text eol=lf` and binary files marked as `binary`.

## Formatting

| Language | Formatter | Enforcement |
|----------|-----------|-------------|
| F# | Fantomas | pre-commit hook or CI |
| C# | `dotnet format` | pre-commit hook or CI |
| Python | ruff format | pre-commit hook |
| TypeScript | prettier | pre-commit hook |
| Go | gofmt | pre-commit hook |

## Linting

Enable the strictest reasonable lint rules for your language:

| Language | Linter | Recommended Config |
|----------|--------|--------------------|
| F# / C# | Roslyn analyzers | `TreatWarningsAsErrors` in Directory.Build.props |
| Python | ruff check | Extended rules: E, W, F, I, B, C4, UP, ARG, SIM |
| TypeScript | ESLint | strict config |

## Warnings as Errors

Production code should compile cleanly:

- .NET: `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>` in `Directory.Build.props`
- Python: `pyright` strict mode
- TypeScript: `strict: true` in tsconfig.json

## Nullable / Strict Types

- .NET: `<Nullable>enable</Nullable>`
- TypeScript: `strictNullChecks: true`
- Python: type annotations + pyright strict
