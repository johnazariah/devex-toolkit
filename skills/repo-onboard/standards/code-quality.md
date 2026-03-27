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

## .NET Language Selection

**Default: F# for core logic, C# for UI bindings.**

F# is preferred for domain modelling, data pipelines, and Tagless-Final architectures. C# is used for Avalonia/MAUI XAML bindings where tooling requires it.

**Exception — Orleans**: If the project uses Microsoft Orleans (virtual actors, grain persistence), use **C# throughout**. Orleans' serialization code generation, grain interfaces, and source generators are C#-first. F# interop is technically possible but causes friction with `[<GenerateSerializer>]`, surrogate patterns, and grain state serialization.

## Architecture: Tagless-Final (Default)

For projects with multiple backend providers or external dependencies, use the **Tagless-Final** pattern as the default architecture. Define capabilities as abstract records of functions parameterized over the effect type. Wire concrete implementations at the composition root.

This applies to any abstraction boundary: API clients, storage, AI backends, search engines, etc.

### F# Example

```fsharp
type EmailProvider<'F> = {
    ListMessages: DateTimeOffset option -> 'F<EmailMessage list>
    GetAttachments: string -> 'F<EmailAttachment list>
}
// Concrete: GmailProvider : EmailProvider<Task>
// Test:     FakeProvider  : EmailProvider<Id>
```

### Python Example (Protocol-based equivalent)

```python
from typing import Protocol

class EmailProvider(Protocol):
    def list_messages(self, since: datetime | None) -> list[EmailMessage]: ...
    def get_attachments(self, message_id: str) -> list[EmailAttachment]: ...
```

### TypeScript Example

```typescript
interface EmailProvider<F> {
    listMessages: (since?: Date) => F<EmailMessage[]>;
    getAttachments: (messageId: string) => F<EmailAttachment[]>;
}
```

### Why

- Testability: swap in pure/fake interpreters without mocks
- Extensibility: new provider = new record value, no interface changes
- Composability: pipeline stages stay abstract over their effects
- Clarity: capabilities are explicit, not hidden behind DI registrations
