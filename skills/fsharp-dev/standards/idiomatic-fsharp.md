# Standard: Idiomatic F#

> **Audience:** AI agents writing, reviewing, or refactoring F# code.
> **Prerequisite:** Read `skills/repo-onboard/standards/code-quality.md` for Tagless-Final architecture and general .NET rules.

## Core Principle

F# is a functional-first language. Code should read as **data transformations flowing through pipelines**, not as imperative step-by-step instructions. Every rule below serves this principle.

---

## 1. Function Size

**Rule:** No function exceeds ~20 lines of logic. No `task {}` block exceeds ~15 lines.

**Why:** Large functions hide complexity, resist composition, and cause pathological F# compilation times (the compiler's type inference slows exponentially with block size).

### Bad

```fsharp
let syncAccount (db: Database) (provider: EmailProvider) (account: string) =
    task {
        let mutable downloaded = 0
        let mutable duplicates = 0
        let mutable processed = 0
        let errors = ResizeArray<string>()
        try
            let! lastSync = loadSyncState db account
            let! messages = provider.listNewMessages lastSync
            for msg in messages do
                try
                    let! exists = messageExists db account msg.ProviderId
                    if not exists then
                        do! recordMessage db account msg
                        processed <- processed + 1
                        let! attachments = provider.getAttachments msg.ProviderId
                        for att in attachments do
                            let! sha = computeSha256 att.Content
                            let! isDup = isDuplicate db sha
                            if isDup then
                                duplicates <- duplicates + 1
                            else
                                do! saveAttachment db att
                                downloaded <- downloaded + 1
                with ex ->
                    errors.Add(ex.Message)
        with ex ->
            errors.Add($"Sync failed: {ex.Message}")
        return { MessagesProcessed = processed
                 AttachmentsDownloaded = downloaded
                 DuplicatesSkipped = duplicates
                 Errors = errors |> Seq.toList }
    }
```

### Good

```fsharp
type SyncAccum =
    { Processed: int; Downloaded: int; Duplicates: int; Errors: string list }

module SyncAccum =
    let zero = { Processed = 0; Downloaded = 0; Duplicates = 0; Errors = [] }
    let addProcessed a = { a with Processed = a.Processed + 1 }
    let addDownloaded a = { a with Downloaded = a.Downloaded + 1 }
    let addDuplicate a = { a with Duplicates = a.Duplicates + 1 }
    let addError msg a = { a with Errors = msg :: a.Errors }

let private processAttachment (db: Database) (accum: SyncAccum) (att: Attachment) =
    task {
        let! sha = computeSha256 att.Content
        let! isDup = isDuplicate db sha
        if isDup then return accum |> SyncAccum.addDuplicate
        else
            do! saveAttachment db att
            return accum |> SyncAccum.addDownloaded
    }

let private processMessage (db: Database) (provider: EmailProvider) account accum (msg: EmailMessage) =
    task {
        let! exists = messageExists db account msg.ProviderId
        if exists then return accum
        else
            do! recordMessage db account msg
            let accum = accum |> SyncAccum.addProcessed
            let! attachments = provider.getAttachments msg.ProviderId
            return! attachments |> List.foldTask (processAttachment db) accum
    }

let syncAccount (db: Database) (provider: EmailProvider) (account: string) =
    task {
        let! lastSync = loadSyncState db account
        let! messages = provider.listNewMessages lastSync
        let! result = messages |> List.foldTask (processMessage db provider account) SyncAccum.zero
        return { result with Errors = result.Errors |> List.rev }
    }
```

---

## 2. Pipelining

**Rule:** Data flows left-to-right via `|>`. Avoid intermediate `let` bindings unless they add clarity to a non-obvious step.

### Bad

```fsharp
let accounts =
    if isNull (box dto.Accounts) || dto.Accounts.Length = 0 then
        def.Accounts
    else
        let mapped = dto.Accounts |> Array.map toAccountConfig
        let result = mapped |> Array.toList
        result
```

### Good

```fsharp
let accounts =
    dto.Accounts
    |> Option.ofObj
    |> Option.filter (fun a -> a.Length > 0)
    |> Option.map (Array.map toAccountConfig >> Array.toList)
    |> Option.defaultValue def.Accounts
```

### Pipeline Length Guideline

- **1–5 stages:** Single pipeline, no comments needed.
- **6–10 stages:** Add a `// phase: description` comment before logical groups.
- **10+ stages:** Break into named sub-functions, each a short pipeline.

---

## 3. No Mutable State

**Rule:** Never use `mutable` for accumulators, counters, or loop state. Use `fold`, recursive functions with accumulators, or accumulator records.

`ResizeArray<T>` is permitted only at I/O boundaries (e.g., reading database rows) and must not leak into domain logic.

### Bad

```fsharp
let mutable completed = 0
let mutable failures = 0
for docId in ids do
    let! result = embedDocument db docId
    match result with
    | Ok _ -> completed <- completed + 1
    | Error _ -> failures <- failures + 1
```

### Good

```fsharp
type BatchResult = { Completed: int; Failures: int }

let processOne (accum: BatchResult) docId =
    task {
        let! result = embedDocument db docId
        return
            match result with
            | Ok _ -> { accum with Completed = accum.Completed + 1 }
            | Error _ -> { accum with Failures = accum.Failures + 1 }
    }

let! result = ids |> Array.foldTask processOne { Completed = 0; Failures = 0 }
```

### When Mutation is Acceptable

- **Performance-critical inner loops** with measured benchmarks proving the need.
- **Interop boundaries** with C# libraries requiring mutable state.
- **Never** in domain logic, pipeline stages, or public API surfaces.

---

## 4. Task Composition

**Rule:** Small named functions returning `Task<'T>` composed together. Never nest `task { task { } }`.

### Bad

```fsharp
let processDocument (fs: FileSystem) (db: Database) (path: string) =
    task {
        let! bytes = fs.readAllBytes path
        let! extractionResult =
            task {
                if isPdf path then
                    let! result = extractor.extractPdf bytes
                    match result with
                    | Ok text -> return Ok (analyseText text "pdfpig" None)
                    | Error e -> return Error e
                elif isImage path then
                    let! result = extractor.extractImage bytes
                    match result with
                    | Ok text -> return Ok (analyseText text "vision" (Some 0.7))
                    | Error e -> return Error e
                else
                    return Error $"Unsupported: {path}"
            }
        match extractionResult with
        | Error e -> return Error e
        | Ok result ->
            do! db.updateDocument path result
            return Ok result
    }
```

### Good

```fsharp
let private extractByType (extractor: Extractor) (path: string) (bytes: byte[]) =
    task {
        if isPdf path then return! extractor.extractPdf bytes |> Task.mapResult (analyseText "pdfpig" None)
        elif isImage path then return! extractor.extractImage bytes |> Task.mapResult (analyseText "vision" (Some 0.7))
        else return Error $"Unsupported: {path}"
    }

let processDocument (fs: FileSystem) (db: Database) (extractor: Extractor) (path: string) =
    task {
        let! bytes = fs.readAllBytes path
        let! result = extractByType extractor path bytes
        match result with
        | Error e -> return Error e
        | Ok extraction ->
            do! db.updateDocument path extraction
            return Ok extraction
    }
```

### Guideline for `task {}` / `async {}` Blocks

- Each `task {}` block should do **one thing**: fetch, transform, or persist.
- If you need a `let!` followed by branching followed by another `let!`, the branching is a separate function.
- Compose via `Task.bind`, `Task.map`, or short `task {}` blocks that call named functions.

---

## 5. Option and Result Composition

**Rule:** Use `Option.map`, `Option.bind`, `Option.defaultValue`, `Result.map`, `Result.bind` instead of explicit `match` or `if`.

Reserve explicit `match` for when you need to handle more than two cases or when the transformation is non-trivial (>1 line per branch).

### Bad

```fsharp
let watchFolders =
    if isNull (box dto.WatchFolders) || dto.WatchFolders.Length = 0 then
        def.WatchFolders
    else
        dto.WatchFolders
        |> Array.map (fun w ->
            { Path = if isNull w.Path then "" else w.Path
              Patterns =
                if isNull (box w.Patterns) then []
                else w.Patterns |> Array.toList })
        |> Array.toList
```

### Good

```fsharp
let toWatchFolder (w: WatchFolderDto) : WatchFolderConfig =
    { Path = w.Path |> Option.ofObj |> Option.defaultValue "" |> expandHome
      Patterns = w.Patterns |> Option.ofObj |> Option.map Array.toList |> Option.defaultValue [] }

let watchFolders =
    dto.WatchFolders
    |> Option.ofObj
    |> Option.filter (fun a -> a.Length > 0)
    |> Option.map (Array.map toWatchFolder >> Array.toList)
    |> Option.defaultValue def.WatchFolders
```

### TaskResult Composition

For chains of `Task<Result<'T, 'E>>`, either:
1. **stdlib:** define a small `TaskResult` module with `bind` and `map` locally.
2. **FsToolkit.ErrorHandling:** use `taskResult {}` CE if the project adopts it.

```fsharp
// stdlib approach — define once in a Prelude module
module TaskResult =
    let bind (f: 'a -> Task<Result<'b, 'e>>) (t: Task<Result<'a, 'e>>) =
        task {
            let! r = t
            match r with
            | Ok v -> return! f v
            | Error e -> return Error e
        }

    let map (f: 'a -> 'b) (t: Task<Result<'a, 'e>>) =
        task {
            let! r = t
            return r |> Result.map f
        }
```

---

## 6. Type Discipline

### Discriminated Unions over Strings

**Rule:** If a value comes from a fixed set, use a DU. Provide `toString` and `tryParse` in a companion module.

### Bad

```fsharp
let source = "email_attachment"  // repeated across 5 files
let category = "bank-statements" // ditto
```

### Good

```fsharp
type SourceType = EmailAttachment | WatchedFolder | ManualDrop

module SourceType =
    let toString = function
        | EmailAttachment -> "email_attachment"
        | WatchedFolder -> "watched_folder"
        | ManualDrop -> "manual_drop"

    let tryParse = function
        | "email_attachment" -> Some EmailAttachment
        | "watched_folder" -> Some WatchedFolder
        | "manual_drop" -> Some ManualDrop
        | _ -> None
```

### Named Records over Tuples

**Rule:** If a tuple has more than 2 elements, or is returned from a public function, use a named record.

### Bad

```fsharp
let buildQuery (filter: SearchFilter) : string * (string * obj) list = ...
```

### Good

```fsharp
type QuerySpec = { Sql: string; Parameters: (string * obj) list }

let buildQuery (filter: SearchFilter) : QuerySpec = ...
```

**Exception:** `string * obj` pairs for SQL parameters are acceptable — they map directly to ADO.NET's API and a wrapper adds no clarity.

---

## 7. Module Organisation

**Rule:** One concept per module. Use `[<RequireQualifiedAccess>]` by default. A module file should be ≤150 lines; if it exceeds 200, look for a concept to extract.

### Signs a Module Needs Splitting

- It has `private` helper functions unrelated to its primary concept.
- It mixes I/O concerns (DB, filesystem) with pure logic.
- It has more than 3 `let` bindings that other modules don't call.

### Naming

| Entity | Convention | Example |
|--------|-----------|---------|
| Module | PascalCase, noun (concept) | `EmailSync`, `Classification`, `Extraction` |
| Public function | PascalCase, verb-first | `processMessage`, `buildQuery`, `tryParse` |
| Private function | camelCase | `computeHash`, `stripHtml` |
| Type | PascalCase | `SearchFilter`, `SyncResult` |
| DU case | PascalCase | `EmailAttachment`, `WatchedFolder` |

### File Ordering in `.fsproj`

F# compilation is order-dependent. Follow this structure:

```
1. Prelude.fs         — shared helpers (TaskResult, List.foldTask, etc.)
2. Domain.fs          — types, DUs, records (no logic)
3. Algebra.fs         — abstract capability records
4. Pure logic modules — classifiers, rules, text processing
5. I/O modules        — database, filesystem, email
6. Composition        — service host, MCP server, entry point
```

---

## 8. Database Access

**Rule:** Database row reading should use a typed accessor pattern, not repeated `Map.tryFind` + `Option.bind` + casting.

### Bad

```fsharp
let get (key: string) : string option =
    row |> Map.tryFind key
    |> Option.bind (fun v ->
        match box v with
        | :? DBNull -> None
        | x when isNull x -> None
        | x -> Some (string x))

let text = get "extracted_text" |> Option.defaultValue ""
let saved = get "saved_path" |> Option.defaultValue ""
let cat = get "category" |> Option.defaultValue "unsorted"
// ... 8 more identical lines
```

### Good

```fsharp
/// Define once in a Prelude or RowReader module
type RowReader(row: Map<string, obj>) =
    member _.String key fallback =
        row |> Map.tryFind key
        |> Option.bind (fun v ->
            match v with
            | :? DBNull -> None
            | null -> None
            | x -> Some (string x))
        |> Option.defaultValue fallback

    member _.Int64 key fallback =
        row |> Map.tryFind key
        |> Option.bind (fun v ->
            match v with
            | :? int64 as i -> Some i
            | :? int as i -> Some (int64 i)
            | _ -> None)
        |> Option.defaultValue fallback

    member _.OptString key =
        row |> Map.tryFind key
        |> Option.bind (fun v ->
            match v with
            | :? DBNull -> None
            | null -> None
            | x -> Some (string x))

// Usage
let r = RowReader(row)
let text = r.String "extracted_text" ""
let saved = r.String "saved_path" ""
let cat = r.String "category" "unsorted"
```

---

## 9. Error Handling

**Rule:** Use `Result<'T, 'Error>` for expected failures. Exceptions only for truly exceptional conditions (network down, disk full). Never catch-and-ignore.

### Pattern: Error DU

```fsharp
type PipelineError =
    | FileNotFound of path: string
    | ExtractionFailed of path: string * reason: string
    | DuplicateDocument of sha256: string
    | DatabaseError of operation: string * exn: exn

module PipelineError =
    let describe = function
        | FileNotFound p -> $"File not found: {p}"
        | ExtractionFailed (p, r) -> $"Extraction failed for {p}: {r}"
        | DuplicateDocument s -> $"Duplicate document: {s}"
        | DatabaseError (op, ex) -> $"Database error during {op}: {ex.Message}"
```

### try/with Scope

- Wrap the **smallest possible expression** in `try/with`.
- Transform exceptions to `Result` at the boundary, then compose with `Result.bind`.

---

## 10. Testing

**Rule:** Tests follow the same idioms as production code. No mutable state in tests. Use factory functions for test data.

### Test Data Factories

```fsharp
module TestData =
    let emailMessage ?(subject = "Test") ?(sender = "test@example.com") ?(bodyText = None) =
        { ProviderId = "msg-001"
          Subject = subject
          Sender = sender
          Date = DateTimeOffset.UtcNow
          BodyText = bodyText }

// Usage
let msg = TestData.emailMessage (subject = "Invoice") (bodyText = Some "amount: $100")
```

### Algebra Fakes

Build fakes as simple records — no mocking frameworks:

```fsharp
let fakeFileSystem =
    { fileExists = fun _ -> true
      readAllBytes = fun _ -> Task.FromResult [||]
      writeAllBytes = fun _ _ -> Task.FromResult ()
      moveFile = fun _ _ -> Task.FromResult () }
```

### Property-based Tests

Use FsCheck for any function with invariants:

```fsharp
[<Property>]
let ``sanitiseFileName replaces all invalid chars`` (filename: NonNull<string>) =
    let sanitised = sanitiseFileName filename.Get
    sanitised.ToCharArray ()
    |> Array.forall (fun c -> not (invalidChars.Contains c))
```

---

## 11. Computation Expression Hygiene

### Avoid Oversized CEs

The F# compiler's type inference scales poorly with CE block size. A `task {}` block over ~15 `let!` bindings will cause:
- Compilation times of 30–220+ seconds for a single file.
- FS3511 warnings ("state machine not statically compilable") in Release builds.
- IDE responsiveness degradation.

**Fix:** Break into composed named functions (see Rules 1 and 4).

### CE Selection

| Need | CE | Library |
|------|----|---------|
| Async I/O | `task { }` | stdlib (F# 6+) |
| Legacy async | `async { }` | stdlib (avoid for new code) |
| Result chaining | `result { }` | FsToolkit.ErrorHandling (optional) |
| Task + Result | `taskResult { }` | FsToolkit.ErrorHandling (optional) |
| Option chaining | `option { }` | FsToolkit.ErrorHandling (optional) |

If the project doesn't use FsToolkit, define minimal `TaskResult.bind` / `TaskResult.map` in a `Prelude.fs` (see Rule 5).

---

## 12. Common Helpers to Define in Prelude

These are small utilities that prevent reimplementation across modules:

```fsharp
[<RequireQualifiedAccess>]
module Prelude

/// Async fold over a list, threading state through each step
let foldTask (f: 'State -> 'T -> Task<'State>) (init: 'State) (items: 'T list) : Task<'State> =
    task {
        let mutable state = init
        for item in items do
            let! next = f state item
            state <- next
        return state
    }

/// Map Result inside a Task
module TaskResult =
    let map f t = task { let! r = t in return Result.map f r }
    let bind f t = task { let! r = t in match r with Ok v -> return! f v | Error e -> return Error e }
    let mapError f t = task { let! r = t in return Result.mapError f r }
```

> Note: `foldTask` uses mutable internally for performance — this is the one acceptable use (implementing a higher-order combinator).

---

## Quick Reference Card

| Smell | Fix |
|-------|-----|
| Function > 20 lines | Extract sub-functions |
| `mutable` counter | Accumulator record + `fold` |
| Nested `task { task { } }` | Named function returning `Task<'T>` |
| `match opt with Some x -> ... \| None -> ...` | `Option.map` / `Option.bind` / `Option.defaultValue` |
| Magic string repeated > 1× | DU + companion module |
| Tuple return from public function | Named record |
| Module > 200 lines | Split by concept |
| `try/with` wrapping 50+ lines | Narrow the `try` scope; convert to Result at boundary |
| `while` loop with mutable | Recursive function or `fold` |
| `let x = ...\nlet y = f x\nlet z = g y` | `... \|> f \|> g` |

---

## 13. Tagless-Final Architecture (Deep Dive)

The Tagless-Final pattern is the **default architecture** for F# projects with external dependencies. This section expands on the overview in `code-quality.md` with F#-specific implementation guidance.

### Algebra Definition

Define capabilities as records of functions, parameterized over the effect type. One record per bounded context / external dependency.

```fsharp
/// Each capability is a record of functions — not an interface.
type EmailProvider<'F> = {
    listNewMessages: DateTimeOffset option -> 'F<EmailMessage list>
    getAttachments: string -> 'F<Attachment list>
    getMessageBody: string -> 'F<string option>
}

type Database = {
    execNonQuery: string -> (string * obj) list -> Task<int>
    execScalar: string -> (string * obj) list -> Task<obj option>
    execReader: string -> (string * obj) list -> Task<Map<string, obj> list>
}

type FileSystem = {
    fileExists: string -> bool
    readAllBytes: string -> Task<byte[]>
    writeAllBytes: string -> byte[] -> Task<unit>
    moveFile: string -> string -> Task<unit>
    ensureDirectory: string -> unit
}
```

### Why Records, Not Interfaces

| Concern | F# Interface | Record of Functions |
|---------|-------------|-------------------|
| Declaration | Requires type + members | Just a record |
| Instantiation | Needs class implementing interface | Record expression `{ fn1 = ...; fn2 = ... }` |
| Partial fakes | Must implement all members | Can use `{ defaults with fn1 = custom }` |
| Composition | Awkward (decorator pattern) | Function composition on record fields |
| Parameterize over effect | Requires HKT workarounds | Natural: `'F<'T>` in record fields |

### Implementation Pattern

Implementations are just functions that return populated records:

```fsharp
module GmailProvider =
    let create (credentials: GmailCredentials) : EmailProvider<Task> =
        { listNewMessages = fun since ->
            task {
                let! raw = Gmail.listMessages credentials since
                return raw |> List.map toDomainMessage
            }
          getAttachments = fun messageId ->
            task {
                let! raw = Gmail.getAttachments credentials messageId
                return raw |> List.map toDomainAttachment
            }
          getMessageBody = fun messageId ->
            task {
                let! body = Gmail.getMessageBody credentials messageId
                return body |> Option.map stripHtml
            }
        }
```

### Composition Root

Wire everything at the entry point — this is the **only** place concrete implementations appear:

```fsharp
module CompositionRoot =
    let build (config: HermesConfig) : AppServices =
        let db = Database.connect config.DatabasePath
        let fs = FileSystem.live ()
        let email = GmailProvider.create config.GmailCredentials
        let extractor = PdfPigExtractor.create ()
        { db = db; fs = fs; email = email; extractor = extractor }
```

### Algebra Layering

For complex systems, compose smaller algebras into larger ones:

```fsharp
/// Low-level capability
type EmbeddingClient = {
    embed: string -> Task<float32[]>
    isAvailable: unit -> Task<bool>
}

/// Higher-level capability built from lower ones
type SearchEngine = {
    keywordSearch: SearchFilter -> Task<SearchResult list>
    semanticSearch: string -> int -> Task<SearchResult list>
    unifiedSearch: SearchFilter -> Task<SearchResult list>
}

/// The higher-level is constructed from the lower
module SearchEngine =
    let create (db: Database) (embedder: EmbeddingClient) : SearchEngine =
        { keywordSearch = Search.keyword db
          semanticSearch = Search.semantic db embedder
          unifiedSearch = Search.unified db embedder }
```

### Testing with Tagless-Final

The architecture makes testing trivial — no mocking frameworks, just record values:

```fsharp
let fakeEmailProvider (messages: EmailMessage list) : EmailProvider<Task> =
    { listNewMessages = fun _ -> Task.FromResult messages
      getAttachments = fun _ -> Task.FromResult []
      getMessageBody = fun _ -> Task.FromResult None }

let fakeDatabase () : Database =
    let store = System.Collections.Concurrent.ConcurrentDictionary<string, obj>()
    { execNonQuery = fun _ _ -> Task.FromResult 0
      execScalar = fun key _ -> Task.FromResult (store.TryGetValue(key) |> function true, v -> Some v | _ -> None)
      execReader = fun _ _ -> Task.FromResult [] }
```

---

## 14. Partial Application and Currying

**Rule:** Design multi-parameter functions with the "most stable" parameters first so they can be partially applied. Dependencies and configuration first, data last.

### Bad

```fsharp
let processMessage (msg: EmailMessage) (db: Database) (account: string) =
    // db and account are stable; msg varies — but msg is first
    ...
```

### Good

```fsharp
let processMessage (db: Database) (account: string) (msg: EmailMessage) =
    // Stable params first → can partially apply
    ...

// Usage: partial application in a pipeline
messages |> List.map (processMessage db account)
```

### Section Application with `>>` (Function Composition)

```fsharp
let pipeline =
    sanitiseFileName
    >> classifyByName rules
    >> Result.map (fun cat -> Path.Combine(archiveDir, DocumentCategory.toString cat))

// Apply to data
let destPath = fileName |> pipeline
```

---

## 15. Active Patterns

**Rule:** Use active patterns to encapsulate complex matching logic, keeping `match` expressions clean and reusable.

### Single-Case Active Pattern (for parsing/extraction)

```fsharp
let (|ParsedDate|_|) (text: string) =
    match DateTimeOffset.TryParse(text) with
    | true, dt -> Some dt
    | _ -> None

let (|Int64|_|) (text: string) =
    match Int64.TryParse(text) with
    | true, n -> Some n
    | _ -> None

// Usage
match row.["date"] with
| ParsedDate dt -> Some dt
| _ -> None
```

### Multi-Case Active Pattern (for classification)

```fsharp
let (|Pdf|Image|Text|Unknown|) (path: string) =
    match Path.GetExtension(path).ToLowerInvariant() with
    | ".pdf" -> Pdf
    | ".jpg" | ".jpeg" | ".png" | ".tiff" -> Image
    | ".txt" | ".md" | ".csv" -> Text
    | _ -> Unknown

// Usage — clean and exhaustive
let extract (path: string) (bytes: byte[]) =
    match path with
    | Pdf -> extractPdf bytes
    | Image -> extractImage bytes
    | Text -> extractText bytes
    | Unknown -> Error $"Unsupported: {path}"
```

---

## 16. Units of Measure

**Rule:** For numeric domains where unit confusion is possible, use F#'s units of measure.

```fsharp
[<Measure>] type ms
[<Measure>] type bytes
[<Measure>] type chars

let chunkSize = 500<chars>
let timeout = 30_000<ms>
let maxFileSize = 10_485_760L<bytes>

let toSeconds (ms: int<ms>) = float ms / 1000.0
```

---

## 17. Computation Expression Builders (When to Create)

**Rule:** Create a custom CE builder only when you have a recurring pattern of bind/return/error-handling that appears in 3+ functions. Otherwise, use explicit `Result.bind` or `TaskResult.bind`.

### When to Build

- You find yourself writing `match result with Ok x -> ... | Error e -> Error e` more than 3 times in a module.
- You need `use!` / `try!/with` semantics for resource management in a custom monad.

### When Not to Build

- One-off Result chaining — use `Result.bind` directly.
- Task-only workflows — `task { }` is already a CE.

---

Quick Reference Card (updated)

| Smell | Fix |
|-------|-----|
| Function > 20 lines | Extract sub-functions |
| `mutable` counter | Accumulator record + `fold` |
| Nested `task { task { } }` | Named function returning `Task<'T>` |
| `match opt with Some x -> ... \| None -> ...` | `Option.map` / `Option.bind` / `Option.defaultValue` |
| Magic string repeated > 1× | DU + companion module |
| Tuple return from public function | Named record |
| Module > 200 lines | Split by concept |
| `try/with` wrapping 50+ lines | Narrow the `try` scope; convert to Result at boundary |
| `while` loop with mutable | Recursive function or `fold` |
| `let x = ...\nlet y = f x\nlet z = g y` | `... \|> f \|> g` |
| Data param before config param | Reorder: stable params first (for partial application) |
| Complex `match` with parsing logic | Active pattern |
| Unitless numeric with domain meaning | Units of measure |
| Interface for swappable capability | Record of functions (Tagless-Final) |
| Concrete dependency in function body | Accept algebra record as parameter |
