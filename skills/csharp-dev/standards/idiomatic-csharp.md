# Standard: Idiomatic C#

> **Audience:** AI agents writing, reviewing, or refactoring C# code.
> **Prerequisite:** Read `skills/repo-onboard/standards/code-quality.md` for Tagless-Final architecture and general .NET rules.

## Core Principle

Modern C# (12+) supports a **functional-object hybrid** style. Prefer immutability, value semantics, expression-bodied members, and LINQ pipelines. Write code that is **data-centric and compositional** — not ceremony-heavy OOP with mutable state scattered across class hierarchies.

---

## 1. Immutability by Default

**Rule:** All data types are `record` or `readonly struct`. All fields are `init`-only or `required`. Never expose a settable property unless mutation is the explicit purpose of the type.

### Bad

```csharp
public class SearchResult
{
    public string Title { get; set; } = "";
    public string Path { get; set; } = "";
    public double Score { get; set; }
    public List<string> Tags { get; set; } = new();
}
```

### Good

```csharp
public sealed record SearchResult(
    string Title,
    string Path,
    double Score,
    IReadOnlyList<string> Tags);
```

### Guidelines

- **Records** for domain types, DTOs, configuration, and return types.
- **`readonly record struct`** for small value types (≤4 fields, no reference-type fields that dominate).
- **Classes** only when you need mutable state with controlled mutation (e.g., builders, UI view models).
- **Collections:** expose `IReadOnlyList<T>`, `IReadOnlyDictionary<K,V>`, or `ImmutableArray<T>`. Never expose `List<T>` or `Dictionary<K,V>` from a public API.

---

## 2. Primary Constructors and Records

**Rule:** Use primary constructors for records and classes where all dependencies are constructor-injected. Use `required` properties only when primary constructors aren't suitable.

### Bad

```csharp
public class EmailSyncService
{
    private readonly IEmailProvider _provider;
    private readonly IDatabase _db;
    private readonly ILogger<EmailSyncService> _logger;

    public EmailSyncService(IEmailProvider provider, IDatabase db, ILogger<EmailSyncService> logger)
    {
        _provider = provider;
        _db = db;
        _logger = logger;
    }

    public async Task SyncAsync() { /* uses _provider, _db, _logger */ }
}
```

### Good

```csharp
public sealed class EmailSyncService(
    IEmailProvider provider,
    IDatabase db,
    ILogger<EmailSyncService> logger)
{
    public async Task SyncAsync() { /* uses provider, db, logger directly */ }
}
```

### When Not to Use Primary Constructors

- When you need to validate/transform constructor parameters (use a regular constructor with `ArgumentException`).
- When the class has multiple constructor overloads.

---

## 3. Non-Nullability

**Rule:** Nullable reference types are enabled project-wide (`<Nullable>enable</Nullable>`). Never suppress with `!`. Never use `null` where an empty collection, `Option<T>`, or default value works.

### Patterns

| Scenario | Approach |
|----------|----------|
| Optional value | `T?` for reference types, `T?` (Nullable) for value types |
| Missing config | Default value via `??` or `?? throw` |
| Return "not found" | Return `T?` and let caller decide |
| Collection may be empty | Return empty collection, never `null` |
| Async "not found" | Return `Task<T?>` |

### Bad

```csharp
public Document? GetDocument(long id)
{
    var result = db.Query(id);
    if (result == null) return null;
    return MapToDocument(result);
}

// Caller:
var doc = service.GetDocument(42);
if (doc != null) { ... }
```

### Good

```csharp
public Document? GetDocument(long id)
    => db.Query(id) is { } row ? MapToDocument(row) : null;

// Caller — pattern matching:
if (service.GetDocument(42) is { } doc) { ... }
```

---

## 4. Function Size and Composition

**Rule:** No method exceeds ~20 lines of logic. Extract inner functions (local functions) for readability. Compose via method chaining, LINQ, or static helper methods.

### Bad

```csharp
public async Task<SyncResult> SyncAccountAsync(string account, CancellationToken ct)
{
    var downloaded = 0;
    var duplicates = 0;
    var errors = new List<string>();
    try
    {
        var lastSync = await LoadSyncStateAsync(account, ct);
        var messages = await _provider.ListNewMessagesAsync(lastSync, ct);
        foreach (var msg in messages)
        {
            try
            {
                if (await MessageExistsAsync(account, msg.ProviderId, ct))
                    continue;
                await RecordMessageAsync(account, msg, ct);
                var attachments = await _provider.GetAttachmentsAsync(msg.ProviderId, ct);
                foreach (var att in attachments)
                {
                    var sha = ComputeSha256(att.Content);
                    if (await IsDuplicateAsync(sha, ct))
                    {
                        duplicates++;
                        continue;
                    }
                    await SaveAttachmentAsync(att, ct);
                    downloaded++;
                }
            }
            catch (Exception ex) { errors.Add(ex.Message); }
        }
    }
    catch (Exception ex) { errors.Add($"Sync failed: {ex.Message}"); }
    return new SyncResult(downloaded, duplicates, errors);
}
```

### Good

```csharp
public async Task<SyncResult> SyncAccountAsync(string account, CancellationToken ct)
{
    var lastSync = await LoadSyncStateAsync(account, ct);
    var messages = await _provider.ListNewMessagesAsync(lastSync, ct);
    var results = await Task.WhenAll(messages.Select(m => ProcessMessageAsync(account, m, ct)));
    return SyncResult.Aggregate(results);
}

private async Task<MessageResult> ProcessMessageAsync(string account, EmailMessage msg, CancellationToken ct)
{
    if (await MessageExistsAsync(account, msg.ProviderId, ct))
        return MessageResult.Skipped;

    await RecordMessageAsync(account, msg, ct);
    var attachments = await _provider.GetAttachmentsAsync(msg.ProviderId, ct);
    var counts = await ProcessAttachmentsAsync(attachments, ct);
    return counts;
}

private async Task<MessageResult> ProcessAttachmentsAsync(
    IReadOnlyList<Attachment> attachments, CancellationToken ct)
{
    var downloaded = 0;
    var duplicates = 0;
    foreach (var att in attachments)
    {
        if (await IsDuplicateAsync(ComputeSha256(att.Content), ct))
            duplicates++;
        else { await SaveAttachmentAsync(att, ct); downloaded++; }
    }
    return new MessageResult(downloaded, duplicates);
}
```

### Local Functions

Use local functions to keep related logic close while still decomposing:

```csharp
public IReadOnlyList<ClassificationResult> ClassifyBatch(IEnumerable<Document> docs)
{
    ClassificationResult Classify(Document doc)
        => ApplyRules(doc) is { } match
            ? new ClassificationResult(doc.Id, match.Category, match.Confidence)
            : new ClassificationResult(doc.Id, Category.Unsorted, 0.0);

    return docs.Select(Classify).ToList();
}
```

---

## 5. Tuples over `out`/`ref` Parameters

**Rule:** Never use `out` or `ref` parameters. Return tuples or records instead. The only exception is `TryParse` patterns from the BCL.

### Bad

```csharp
public bool TryExtractDate(string text, out DateTime date, out string format)
{
    // ...
}
```

### Good

```csharp
public (DateTime Date, string Format)? TryExtractDate(string text)
{
    // ... return null if not found
}
```

Or with a record for richer results:

```csharp
public sealed record DateExtraction(DateTime Date, string Format, int Position);

public DateExtraction? TryExtractDate(string text) { ... }
```

---

## 6. LINQ and Pipelines

**Rule:** Use LINQ method syntax for data transformations. Use query syntax only when multiple `from` / `join` / `let` clauses improve readability over method chains.

### Bad

```csharp
var results = new List<SearchHit>();
foreach (var row in rows)
{
    if (row["score"] is double score && score > 0.5)
    {
        var hit = new SearchHit(
            (string)row["title"],
            (string)row["path"],
            score);
        results.Add(hit);
    }
}
return results;
```

### Good

```csharp
return rows
    .Where(r => r["score"] is double score && score > 0.5)
    .Select(r => new SearchHit(
        (string)r["title"],
        (string)r["path"],
        (double)r["score"]))
    .ToList();
```

### Pipeline Length Guideline

- **1–5 stages:** Single LINQ chain.
- **6–10 stages:** Add comments between logical groups or extract intermediate queries.
- **10+ stages:** Break into named methods, each a short chain.

### Prefer LINQ over Loops When

- Filtering (`Where`)
- Projecting (`Select`)
- Aggregating (`Aggregate`, `Sum`, `Count`, `GroupBy`)
- Flattening (`SelectMany`)

### Prefer `foreach` over LINQ When

- Side effects (I/O, logging, mutation of external state)
- Async iteration (`await foreach`)
- Performance-critical inner loops where LINQ allocation matters

---

## 7. Pattern Matching

**Rule:** Use pattern matching exhaustively. Prefer `switch` expressions over `if/else` chains. Use property patterns for complex conditions.

### Bad

```csharp
public string GetDisplayName(Document doc)
{
    if (doc.Category == Category.Invoice)
        return $"Invoice: {doc.Title}";
    else if (doc.Category == Category.Receipt)
        return $"Receipt: {doc.Title}";
    else if (doc.Category == Category.BankStatement)
        return $"Statement: {doc.Title}";
    else
        return doc.Title;
}
```

### Good

```csharp
public string GetDisplayName(Document doc) => doc.Category switch
{
    Category.Invoice => $"Invoice: {doc.Title}",
    Category.Receipt => $"Receipt: {doc.Title}",
    Category.BankStatement => $"Statement: {doc.Title}",
    _ => doc.Title,
};
```

### Property Patterns

```csharp
public decimal CalculateDiscount(Order order) => order switch
{
    { Total: > 1000, Customer.IsPremium: true } => order.Total * 0.15m,
    { Total: > 500 } => order.Total * 0.10m,
    { Customer.IsNewCustomer: true } => order.Total * 0.05m,
    _ => 0m,
};
```

### List Patterns (C# 11+)

Match on collection structure — head, tail, length:

```csharp
public string DescribeArgs(string[] args) => args switch
{
    [] => "No arguments",
    [var single] => $"One argument: {single}",
    [var first, .. var rest] => $"First: {first}, remaining: {rest.Length}",
};
```

### Type Patterns and `is` Decomposition

Always prefer pattern matching over `is` + cast or `as` + null check:

```csharp
// Bad
if (obj is Document)
{
    var doc = (Document)obj;
    Process(doc);
}

// Good
if (obj is Document doc)
    Process(doc);

// Good — nested pattern
if (obj is Document { Category: Category.Invoice, Title: var title })
    logger.LogInformation("Processing invoice: {Title}", title);
```

### Relational and Logical Patterns

```csharp
public string ClassifyScore(double score) => score switch
{
    < 0.0 => throw new ArgumentOutOfRangeException(nameof(score)),
    < 0.3 => "low",
    < 0.7 => "medium",
    < 0.9 => "high",
    _ => "excellent",
};

public bool IsWeekend(DayOfWeek day) => day is DayOfWeek.Saturday or DayOfWeek.Sunday;
```

### Null Checking — Always Pattern Match

```csharp
// Bad
if (result != null) { ... }
if (result == null) return;

// Good
if (result is { } value) { ... }
if (result is null) return;

// Good — with further decomposition
if (await GetDocumentAsync(id, ct) is { Category: var cat, Title: var title })
    logger.LogInformation("Found {Category}: {Title}", cat, title);
```

### When to Use Each Pattern Form

| Scenario | Pattern Form |
|----------|-------------|
| Enum dispatch | `switch` expression on enum |
| Null check | `is { }` / `is null` |
| Type dispatch | `is TypeName varName` |
| Nested property check | Property pattern `{ Prop: value }` |
| Collection structure | List pattern `[first, .. rest]` |
| Range/threshold | Relational pattern `< 0.5` |
| Complex condition | Logical pattern `and` / `or` / `not` |
| Multi-field decomposition | Positional pattern for records |

---

## 8. Tagless-Final Architecture

**Rule:** Define capabilities as interfaces with generic effect types. Wire implementations at the composition root. This is the default architecture for any project with external dependencies.

### Capability Interface

```csharp
public interface IEmailProvider<F>
    where F : allows ref struct  // C# 13+ unconstrained
{
    F<IReadOnlyList<EmailMessage>> ListMessagesAsync(DateTimeOffset? since, CancellationToken ct);
    F<IReadOnlyList<Attachment>> GetAttachmentsAsync(string messageId, CancellationToken ct);
}
```

### Practical Approach (Task-based)

For most C# projects, parameterizing over the effect type adds complexity without proportional benefit. The pragmatic Tagless-Final in C# uses **interface + record-of-delegates** duality:

```csharp
// Option A: Interface (when implementations need state/DI)
public interface IEmailProvider
{
    Task<IReadOnlyList<EmailMessage>> ListMessagesAsync(DateTimeOffset? since, CancellationToken ct);
    Task<IReadOnlyList<Attachment>> GetAttachmentsAsync(string messageId, CancellationToken ct);
}

// Option B: Delegate record (when implementations are simple/stateless)
public sealed record EmailProviderOps(
    Func<DateTimeOffset?, CancellationToken, Task<IReadOnlyList<EmailMessage>>> ListMessages,
    Func<string, CancellationToken, Task<IReadOnlyList<Attachment>>> GetAttachments);
```

### Fakes for Testing

```csharp
// Interface fake
public sealed class FakeEmailProvider : IEmailProvider
{
    public List<EmailMessage> Messages { get; init; } = [];

    public Task<IReadOnlyList<EmailMessage>> ListMessagesAsync(DateTimeOffset? since, CancellationToken ct)
        => Task.FromResult<IReadOnlyList<EmailMessage>>(
            Messages.Where(m => since is null || m.Date > since).ToList());

    public Task<IReadOnlyList<Attachment>> GetAttachmentsAsync(string messageId, CancellationToken ct)
        => Task.FromResult<IReadOnlyList<Attachment>>([]);
}

// Delegate record fake — even simpler
var fakeProvider = new EmailProviderOps(
    ListMessages: (_, _) => Task.FromResult<IReadOnlyList<EmailMessage>>([]),
    GetAttachments: (_, _) => Task.FromResult<IReadOnlyList<Attachment>>([]));
```

### Composition Root

```csharp
// Program.cs or host builder
services.AddSingleton<IEmailProvider>(sp =>
    new GmailProvider(sp.GetRequiredService<GmailConfig>()));
services.AddSingleton<IDocumentExtractor>(sp =>
    new PdfPigExtractor());
```

---

## 9. Value Semantics and Enums

### Enums Over Strings

**Rule:** If a value comes from a fixed set, use an `enum`. If you need associated data, use a record hierarchy with a base abstract record or a DU-like pattern.

### Bad

```csharp
var source = "email_attachment";  // repeated in 5 files
var category = "bank-statements";
```

### Good

```csharp
public enum SourceType { EmailAttachment, WatchedFolder, ManualDrop }
public enum DocumentCategory { Unclassified, BankStatements, Insurance, Invoices, Unsorted }

public static class SourceTypeExtensions
{
    public static string ToWireFormat(this SourceType s) => s switch
    {
        SourceType.EmailAttachment => "email_attachment",
        SourceType.WatchedFolder => "watched_folder",
        SourceType.ManualDrop => "manual_drop",
        _ => throw new ArgumentOutOfRangeException(nameof(s)),
    };

    public static SourceType? ParseSourceType(string value) => value switch
    {
        "email_attachment" => SourceType.EmailAttachment,
        "watched_folder" => SourceType.WatchedFolder,
        "manual_drop" => SourceType.ManualDrop,
        _ => null,
    };
}
```

### Discriminated Union Pattern (C# 12+)

When enum cases carry different data, use sealed record hierarchy:

```csharp
public abstract record PipelineError
{
    public sealed record FileNotFound(string Path) : PipelineError;
    public sealed record ExtractionFailed(string Path, string Reason) : PipelineError;
    public sealed record Duplicate(string Sha256) : PipelineError;
    public sealed record DatabaseError(string Operation, Exception Inner) : PipelineError;

    public string Describe() => this switch
    {
        FileNotFound e => $"File not found: {e.Path}",
        ExtractionFailed e => $"Extraction failed for {e.Path}: {e.Reason}",
        Duplicate e => $"Duplicate: {e.Sha256}",
        DatabaseError e => $"DB error during {e.Operation}: {e.Inner.Message}",
        _ => ToString(),
    };
}
```

---

## 10. Error Handling

**Rule:** Use exceptions for truly exceptional conditions. Use `Result<T>` or nullable returns for expected failures. Never catch-and-ignore.

### Result Pattern

```csharp
// Define once
public readonly record struct Result<T>(T? Value, string? Error)
{
    public bool IsOk => Error is null;
    public static Result<T> Ok(T value) => new(value, null);
    public static Result<T> Fail(string error) => new(default, error);

    public Result<U> Map<U>(Func<T, U> f) => IsOk ? Result<U>.Ok(f(Value!)) : Result<U>.Fail(Error!);
    public Result<U> Bind<U>(Func<T, Result<U>> f) => IsOk ? f(Value!) : Result<U>.Fail(Error!);
}
```

### try/catch Scope

- Wrap the **narrowest possible expression** in `try/catch`.
- Convert to `Result<T>` at the boundary; compose with `.Bind()` / `.Map()` above.

---

## 11. Async Conventions

**Rule:** Every async method takes `CancellationToken ct` as its last parameter. Every awaited call forwards `ct`. Suffix async methods with `Async`.

### Bad

```csharp
public async Task<Document> GetDocumentAsync(long id)
{
    var row = await db.QueryAsync($"SELECT * FROM documents WHERE id = {id}");
    return MapToDocument(row);
}
```

### Good

```csharp
public async Task<Document?> GetDocumentAsync(long id, CancellationToken ct)
{
    var row = await db.QueryAsync("SELECT * FROM documents WHERE id = @id",
        new { id }, ct);
    return row is not null ? MapToDocument(row) : null;
}
```

### `ValueTask` vs `Task`

| Scenario | Use |
|----------|-----|
| Usually completes asynchronously | `Task<T>` |
| Often completes synchronously (cache hit, buffered read) | `ValueTask<T>` |
| Interface method (callers may cache/await multiple times) | `Task<T>` |

---

## 12. File and Type Organisation

**Rule:** One primary type per file. File name matches type name. Keep files ≤200 lines; if exceeding 300, look for a concept to extract.

### Namespace Structure

```
ProjectName/
├── Domain/           — Records, enums, value objects (no logic)
├── Capabilities/     — Tagless-Final interfaces
├── Providers/        — Concrete implementations of capabilities
├── Pipeline/         — Orchestration and composition
├── Infrastructure/   — DB, filesystem, HTTP adapters
└── Program.cs        — Composition root
```

### `sealed` by Default

Mark all classes and records `sealed` unless you explicitly design for inheritance. This is both a performance optimization and a design signal.

---

## 13. Testing

### xUnit + Naming

```csharp
public sealed class EmailSyncTests
{
    [Fact]
    public async Task SyncAccount_NewMessages_RecordsAllMessages()
    {
        // Arrange
        var provider = new FakeEmailProvider { Messages = [TestData.EmailMessage()] };
        var db = new FakeDatabase();

        // Act
        var result = await new EmailSyncService(provider, db, NullLogger.Instance)
            .SyncAccountAsync("test@example.com", CancellationToken.None);

        // Assert
        Assert.Equal(1, result.MessagesProcessed);
    }
}
```

### Test Data Factories

```csharp
public static class TestData
{
    public static EmailMessage EmailMessage(
        string subject = "Test",
        string sender = "test@example.com",
        string? bodyText = null)
        => new(
            ProviderId: "msg-001",
            Subject: subject,
            Sender: sender,
            Date: DateTimeOffset.UtcNow,
            BodyText: bodyText);
}
```

### No Mocking Frameworks

For Tagless-Final architectures, **fakes are simpler than mocks**. Define fake implementations of your capability interfaces with `List<T>` or lambda-based behavior. No Moq, no NSubstitute. Mocking frameworks encourage testing implementation details instead of behavior.

---

## 14. Additional Idioms

### `using` Declarations over `using` Blocks

```csharp
// Bad — unnecessary nesting
using (var conn = new SqliteConnection(connectionString))
{
    using (var cmd = conn.CreateCommand())
    {
        // ...
    }
}

// Good — flat
using var conn = new SqliteConnection(connectionString);
using var cmd = conn.CreateCommand();
```

### Collection Expressions (C# 12+)

```csharp
// Bad
var list = new List<string> { "a", "b", "c" };
var empty = Array.Empty<string>();

// Good
List<string> list = ["a", "b", "c"];
string[] empty = [];
```

### Raw String Literals for SQL/JSON

```csharp
var sql = """
    SELECT d.id, d.title, d.category
    FROM documents d
    JOIN documents_fts f ON d.id = f.rowid
    WHERE documents_fts MATCH @query
    ORDER BY rank
    LIMIT @limit
    """;
```

### `required` for DTOs

```csharp
public sealed record HermesConfigDto
{
    public required string ArchiveDir { get; init; }
    public required AccountConfigDto[] Accounts { get; init; }
    public OllamaConfigDto? Ollama { get; init; }
}
```

### Discard with `_`

```csharp
_ = cmd.Parameters.Add(param);  // return value intentionally ignored
```

### Target-Typed `new()`

```csharp
SqliteParameter param = new() { ParameterName = name, Value = value };
```

---

## Quick Reference Card

| Smell | Fix |
|-------|-----|
| `class` with all settable properties | `record` with positional parameters |
| `null` return for empty collection | Return `[]` or `Array.Empty<T>()` |
| `out` / `ref` parameter | Return tuple or record |
| Method > 20 lines | Extract local/private methods |
| `if/else if/else if` chain | `switch` expression |
| Magic string repeated > 1× | `enum` + extension methods |
| `catch (Exception) { }` | Narrow the catch; convert to Result at boundary |
| Mutable `List<T>` exposed publicly | `IReadOnlyList<T>` |
| `new List<string>()` | `List<string> list = []` |
| `using (var x = ...) { }` | `using var x = ...;` |
| Class without `sealed` | Add `sealed` unless designed for inheritance |
| Missing `CancellationToken` | Add as last parameter to every async method |
| `string.Format` or `+` concatenation | String interpolation `$"..."` |
| Nested `if (x != null)` | Pattern matching `if (x is { } val)` |
| `is Type` then cast `(Type)x` | Combined `is Type name` pattern |
| `if/else` on value ranges | Relational pattern: `switch { < 0.5 => ... }` |
| Manual list head/tail check | List pattern: `[first, .. rest]` |
| `as` + null check | `is Type name` pattern |
