# Standard: Idiomatic Python

> **Audience:** AI agents writing, reviewing, or refactoring Python code.
> **Prerequisite:** Read `skills/repo-onboard/standards/code-quality.md` for Tagless-Final architecture and general rules.

## Core Principle

Python is a multi-paradigm language, but modern Python (3.12+) excels at **typed, functional-flavoured code with Protocol-based abstractions**. Write code that is **explicit, composable, and fully typed** — not dynamic soup with `dict` and `**kwargs` everywhere.

---

## 1. Type Annotations Everywhere

**Rule:** Every function signature, every variable that isn't immediately obvious, every return type. Use `pyright` in strict mode. Zero `type: ignore` comments.

### Bad

```python
def process_documents(docs, config):
    results = []
    for doc in docs:
        result = classify(doc, config)
        if result:
            results.append(result)
    return results
```

### Good

```python
def process_documents(
    docs: Sequence[Document], config: ClassifierConfig
) -> list[ClassificationResult]:
    return [
        result
        for doc in docs
        if (result := classify(doc, config)) is not None
    ]
```

### Type Annotation Guidelines

| Context | Convention |
|---------|-----------|
| Function parameters | Always annotated |
| Return types | Always annotated |
| Local variables | Annotate when type isn't obvious from assignment |
| Constants | `Final[str]`, `Final[int]`, etc. |
| Class attributes | Always annotated in `__init__` or as class vars |
| Generics | Use `TypeVar` or PEP 695 syntax (`def f[T](x: T) -> T`) |

---

## 2. Dataclasses and Named Tuples

**Rule:** `@dataclass(frozen=True)` for domain types. `NamedTuple` for lightweight value types. Never use plain `dict` for structured data. Never use mutable dataclasses for domain objects.

### Bad

```python
def create_search_result(row: dict) -> dict:
    return {
        "title": row.get("title", ""),
        "path": row.get("path", ""),
        "score": row.get("score", 0.0),
        "tags": row.get("tags", []),
    }
```

### Good

```python
@dataclass(frozen=True, slots=True)
class SearchResult:
    title: str
    path: str
    score: float
    tags: tuple[str, ...]  # immutable sequence

    @classmethod
    def from_row(cls, row: Mapping[str, Any]) -> SearchResult:
        return cls(
            title=str(row.get("title", "")),
            path=str(row.get("path", "")),
            score=float(row.get("score", 0.0)),
            tags=tuple(row.get("tags", ())),
        )
```

### Guidelines

- **`frozen=True`** always for domain types — immutability by default.
- **`slots=True`** for performance and preventing accidental attribute creation.
- **`tuple[str, ...]`** over `list[str]` for immutable sequences in frozen dataclasses.
- **`@dataclass`** over hand-written `__init__` / `__eq__` / `__repr__`.
- **`NamedTuple`** for return types with 2–4 fields that are essentially tuples with names.

---

## 3. Protocol-Based Tagless-Final

**Rule:** Define capabilities as `Protocol` classes. Implementations are concrete classes or simple objects that satisfy the protocol. Wire at the composition root. Test with fakes.

### Capability Protocol

```python
from typing import Protocol

class EmailProvider(Protocol):
    async def list_messages(self, since: datetime | None) -> list[EmailMessage]: ...
    async def get_attachments(self, message_id: str) -> list[Attachment]: ...

class DocumentExtractor(Protocol):
    async def extract_pdf(self, content: bytes) -> Result[str]: ...
    async def extract_image(self, content: bytes) -> Result[str]: ...
```

### Concrete Implementation

```python
@dataclass(frozen=True, slots=True)
class GmailProvider:
    """Satisfies EmailProvider protocol structurally."""
    credentials: GmailCredentials

    async def list_messages(self, since: datetime | None) -> list[EmailMessage]:
        # ... Gmail API calls
        ...

    async def get_attachments(self, message_id: str) -> list[Attachment]:
        # ... Gmail API calls
        ...
```

### Fakes for Testing

```python
@dataclass
class FakeEmailProvider:
    messages: list[EmailMessage] = field(default_factory=list)
    attachments: dict[str, list[Attachment]] = field(default_factory=dict)

    async def list_messages(self, since: datetime | None) -> list[EmailMessage]:
        if since is None:
            return self.messages
        return [m for m in self.messages if m.date > since]

    async def get_attachments(self, message_id: str) -> list[Attachment]:
        return self.attachments.get(message_id, [])
```

### Composition Root

```python
def create_app(config: AppConfig) -> App:
    provider = GmailProvider(credentials=config.gmail)
    extractor = PdfPigExtractor(config=config.extraction)
    db = SqliteDatabase(path=config.db_path)
    return App(provider=provider, extractor=extractor, db=db)
```

---

## 4. Function Size and Composition

**Rule:** No function exceeds ~20 lines of logic. Compose via comprehensions, `functools`, generator pipelines, and small helper functions.

### Bad

```python
async def sync_account(provider: EmailProvider, db: Database, account: str) -> SyncResult:
    downloaded = 0
    duplicates = 0
    errors: list[str] = []
    try:
        last_sync = await load_sync_state(db, account)
        messages = await provider.list_messages(last_sync)
        for msg in messages:
            try:
                if await message_exists(db, account, msg.provider_id):
                    continue
                await record_message(db, account, msg)
                attachments = await provider.get_attachments(msg.provider_id)
                for att in attachments:
                    sha = compute_sha256(att.content)
                    if await is_duplicate(db, sha):
                        duplicates += 1
                    else:
                        await save_attachment(db, att)
                        downloaded += 1
            except Exception as e:
                errors.append(str(e))
    except Exception as e:
        errors.append(f"Sync failed: {e}")
    return SyncResult(downloaded=downloaded, duplicates=duplicates, errors=errors)
```

### Good

```python
async def _process_attachment(db: Database, att: Attachment) -> Literal["downloaded", "duplicate"]:
    sha = compute_sha256(att.content)
    if await is_duplicate(db, sha):
        return "duplicate"
    await save_attachment(db, att)
    return "downloaded"

async def _process_message(
    provider: EmailProvider, db: Database, account: str, msg: EmailMessage
) -> MessageResult:
    if await message_exists(db, account, msg.provider_id):
        return MessageResult.skipped()
    await record_message(db, account, msg)
    attachments = await provider.get_attachments(msg.provider_id)
    outcomes = await asyncio.gather(*(_process_attachment(db, a) for a in attachments))
    return MessageResult(
        downloaded=outcomes.count("downloaded"),
        duplicates=outcomes.count("duplicate"),
    )

async def sync_account(provider: EmailProvider, db: Database, account: str) -> SyncResult:
    last_sync = await load_sync_state(db, account)
    messages = await provider.list_messages(last_sync)
    results = await asyncio.gather(
        *(_process_message(provider, db, account, m) for m in messages),
        return_exceptions=True,
    )
    return SyncResult.aggregate(results)
```

---

## 5. Result Type

**Rule:** Use a `Result[T, E]` type for operations that can fail expectedly. Never raise exceptions for business logic failures.

```python
@dataclass(frozen=True, slots=True)
class Ok(Generic[T]):
    value: T

@dataclass(frozen=True, slots=True)
class Err(Generic[E]):
    error: E

Result = Ok[T] | Err[E]

def map_result(result: Result[T, E], f: Callable[[T], U]) -> Result[U, E]:
    match result:
        case Ok(value):
            return Ok(f(value))
        case Err() as e:
            return e

def bind_result(result: Result[T, E], f: Callable[[T], Result[U, E]]) -> Result[U, E]:
    match result:
        case Ok(value):
            return f(value)
        case Err() as e:
            return e
```

### Alternative: Use `result` library

If the project adopts the `result` library (`pip install result`), use its `Ok`/`Err` types directly.

---

## 6. Comprehensions and Generators

**Rule:** Prefer comprehensions over accumulation loops. Every `for` loop that builds a new list, dict, or set should be a comprehension. Use generators for lazy pipelines.

### List Comprehension — The Default

```python
# Bad
results = []
for row in rows:
    if row["score"] > 0.5:
        result = SearchResult.from_row(row)
        results.append(result)
return results

# Good
return [SearchResult.from_row(row) for row in rows if row["score"] > 0.5]
```

### Dict Comprehension

```python
# Bad
category_counts: dict[str, int] = {}
for doc in documents:
    cat = doc.category
    if cat not in category_counts:
        category_counts[cat] = 0
    category_counts[cat] += 1

# Good — use Counter for counting
category_counts = Counter(doc.category for doc in documents)

# Dict comprehension for transforms
config_map: dict[str, str] = {
    key: expand_path(value)
    for key, value in raw_config.items()
    if value is not None
}
```

### Set Comprehension

```python
# Bad
unique_senders: set[str] = set()
for msg in messages:
    unique_senders.add(msg.sender)

# Good
unique_senders: set[str] = {msg.sender for msg in messages}
```

### Nested Comprehension (Flattening)

```python
# Bad
all_attachments: list[Attachment] = []
for msg in messages:
    for att in msg.attachments:
        all_attachments.append(att)

# Good
all_attachments: list[Attachment] = [
    att for msg in messages for att in msg.attachments
]
```

### Walrus Operator for Filter-and-Transform

When a filter condition and the transformed value share an intermediate computation:

```python
# Compute once, filter, and keep the result
valid: list[ClassificationResult] = [
    result
    for doc in documents
    if (result := classify(doc, config)) is not None
]

# Also works for expensive operations
matches: list[tuple[str, re.Match[str]]] = [
    (line, m)
    for line in lines
    if (m := date_pattern.search(line)) is not None
]
```

### Generator Expressions for Lazy Pipelines

Use generator expressions when you don't need the full list — saves memory:

```python
# Lazy — only materialised on demand
total_bytes: int = sum(doc.size for doc in documents)
has_invoice: bool = any(doc.category == DocumentCategory.INVOICES for doc in documents)
all_classified: bool = all(doc.category != DocumentCategory.UNCLASSIFIED for doc in documents)
```

### Generator Functions for Complex Lazy Pipelines

```python
def extract_text_chunks(text: str, chunk_size: int = 500) -> Iterator[TextChunk]:
    paragraphs = text.split("\n\n")
    for i, para in enumerate(paragraphs):
        if len(para.strip()) > 0:
            yield TextChunk(text=para.strip(), index=i, start_char=text.index(para))

# Chain generators
def pipeline(documents: Iterable[Document]) -> Iterator[SearchResult]:
    classified = (classify(doc) for doc in documents)
    with_text = (extract(doc) for doc in classified if doc.has_content)
    yield from (to_search_result(doc) for doc in with_text)
```

### Pipeline Length Guideline

- **1-liner comprehension:** Up to ~100 chars.
- **Multi-line comprehension:** 2–3 clauses max.
- **Beyond 3 clauses:** Break into named functions and compose.

### When to Use Comprehensions vs Loops

| Use Comprehension | Use `for` Loop |
|-------------------|----------------|
| Building new list/dict/set | Side effects (I/O, logging) |
| Pure transforms | `await` inside the loop (use `asyncio.gather` instead) |
| Filter + map | Early exit (`break`) |
| Flattening nested collections | Complex branching (>2 `if`) |

---

## 7. Structural Pattern Matching

**Rule:** Use `match/case` (Python 3.10+) as the **primary dispatch mechanism** for discriminated unions, command handling, data parsing, and any multi-branch logic. Always ensure exhaustiveness with `assert_never()`. Never fall back to `isinstance()` chains or `if/elif` cascades when `match/case` works.

### Class Pattern — The Default for DU Dispatch

```python
def describe_error(error: PipelineError) -> str:
    match error:
        case FileNotFound(path=p):
            return f"File not found: {p}"
        case ExtractionFailed(path=p, reason=r):
            return f"Extraction failed for {p}: {r}"
        case Duplicate(sha256=s):
            return f"Duplicate: {s}"
        case _ as unreachable:
            assert_never(unreachable)
```

### Guard Clauses (Conditional Patterns)

```python
def classify_score(score: float) -> str:
    match score:
        case s if s < 0.0:
            raise ValueError(f"Negative score: {s}")
        case s if s < 0.3:
            return "low"
        case s if s < 0.7:
            return "medium"
        case s if s < 0.9:
            return "high"
        case _:
            return "excellent"
```

### Mapping Patterns (for JSON/dict Data)

```python
def parse_event(event: dict[str, Any]) -> Event:
    match event:
        case {"type": "message", "sender": str(sender), "body": str(body)}:
            return MessageEvent(sender=sender, body=body)
        case {"type": "attachment", "filename": str(name), "size": int(size)}:
            return AttachmentEvent(filename=name, size=size)
        case {"type": str(t)}:
            return UnknownEvent(event_type=t)
        case _:
            raise ValueError(f"Invalid event: {event}")
```

### Sequence Patterns (Destructuring)

```python
def process_command(args: list[str]) -> Action:
    match args:
        case []:
            return ShowHelp()
        case ["sync", account]:
            return SyncAccount(account=account)
        case ["search", *query_words]:
            return Search(query=" ".join(query_words))
        case [command, *_]:
            return UnknownCommand(command=command)
```

### OR Patterns

```python
def is_image(ext: str) -> bool:
    match ext.lower():
        case ".jpg" | ".jpeg" | ".png" | ".tiff" | ".webp":
            return True
        case _:
            return False
```

### Value Binding with `as`

```python
def process_result(result: Result[Document, PipelineError]) -> str:
    match result:
        case Ok(value=Document(title=t, category=c)) as success:
            logger.info("Processed: %s", success.value.path)
            return f"{c}: {t}"
        case Err(error=FileNotFound() as e):
            return f"Missing: {e.path}"
        case Err(error=e):
            return e.describe()
```

### When to Use Match vs Other Constructs

| Scenario | Use |
|----------|-----|
| Dispatch on dataclass type hierarchy | `match/case` with class patterns |
| Parse dict/JSON into typed data | `match/case` with mapping patterns |
| Command-line argument parsing | `match/case` with sequence patterns |
| Enum dispatch | `match/case` with value patterns |
| Simple boolean check | `if` statement |
| Single type check + use | `if isinstance(x, T)` or `match x: case T() as t:` |
| Range dispatch | `match/case` with guard clauses |

---

## 8. Module Organisation

**Rule:** One concept per module. Modules ≤200 lines. Use `__all__` to control exports.

### Structure

```
src/project_name/
├── domain/          — dataclasses, enums, value objects
│   ├── __init__.py
│   ├── document.py
│   └── email.py
├── capabilities/    — Protocol definitions
├── providers/       — concrete implementations
├── pipeline/        — orchestration
├── infrastructure/  — DB, filesystem, HTTP adapters
├── utils/           — pure utility functions
└── __init__.py      — public API re-exports
```

### Naming

| Entity | Convention | Example |
|--------|-----------|---------|
| Module | `snake_case` | `email_sync.py` |
| Class | `PascalCase` | `EmailSyncService` |
| Function | `snake_case` | `process_message` |
| Constant | `UPPER_SNAKE_CASE` | `MAX_RETRY_COUNT` |
| Private | `_leading_underscore` | `_compute_hash` |
| Type alias | `PascalCase` | `DocumentId = NewType("DocumentId", int)` |

---

## 9. Enums Over Strings

**Rule:** Use `StrEnum` (Python 3.11+) for string-valued enums. Ensure serialization is handled by the enum, not by consumers.

### Bad

```python
source = "email_attachment"
category = "bank-statements"
```

### Good

```python
class SourceType(StrEnum):
    EMAIL_ATTACHMENT = "email_attachment"
    WATCHED_FOLDER = "watched_folder"
    MANUAL_DROP = "manual_drop"

class DocumentCategory(StrEnum):
    UNCLASSIFIED = "unclassified"
    BANK_STATEMENTS = "bank-statements"
    INSURANCE = "insurance"
    INVOICES = "invoices"
    UNSORTED = "unsorted"
```

---

## 10. Error Handling

**Rule:** Use `Result[T, E]` for expected failures. Use exceptions only for truly exceptional conditions. Never bare `except:`. Never `except Exception: pass`.

### try/except Scope

```python
# Bad — too broad
try:
    config = load_config(path)
    results = process_all(config)
    save_results(results)
except Exception:
    logger.error("Something failed")

# Good — narrow
try:
    config = load_config(path)
except FileNotFoundError:
    return Err(f"Config not found: {path}")
except yaml.YAMLError as e:
    return Err(f"Config parse error: {e}")
```

### Error Hierarchy with Dataclasses

```python
@dataclass(frozen=True, slots=True)
class PipelineError:
    """Base for pipeline errors — use match/case to dispatch."""
    pass

@dataclass(frozen=True, slots=True)
class FileNotFound(PipelineError):
    path: str

@dataclass(frozen=True, slots=True)
class ExtractionFailed(PipelineError):
    path: str
    reason: str

@dataclass(frozen=True, slots=True)
class Duplicate(PipelineError):
    sha256: str
```

---

## 11. Async Patterns

### `asyncio.gather` for Independent Work

```python
messages, config, last_sync = await asyncio.gather(
    provider.list_messages(),
    load_config(),
    load_sync_state(db, account),
)
```

### `asyncio.TaskGroup` (Python 3.11+) for Structured Concurrency

```python
async with asyncio.TaskGroup() as tg:
    tasks = [tg.create_task(process_message(m)) for m in messages]
results = [t.result() for t in tasks]
```

### Cancellation

Use `asyncio.CancelledError` — don't suppress it:

```python
async def long_running(cancel_event: asyncio.Event) -> None:
    while not cancel_event.is_set():
        await asyncio.sleep(1)
        # ... work
```

---

## 12. Testing

### pytest + Naming

```python
class TestEmailSync:
    async def test_sync_account_new_messages_records_all(self) -> None:
        provider = FakeEmailProvider(messages=[test_email_message(subject="Invoice")])
        db = FakeDatabase()

        result = await sync_account(provider, db, "test@example.com")

        assert result.messages_processed == 1
```

### Test Data Factories

```python
def test_email_message(
    *,
    subject: str = "Test",
    sender: str = "test@example.com",
    body_text: str | None = None,
) -> EmailMessage:
    return EmailMessage(
        provider_id="msg-001",
        subject=subject,
        sender=sender,
        date=datetime.now(UTC),
        body_text=body_text,
    )
```

### Fakes Over Mocks

Never use `unittest.mock.patch` or `MagicMock` for Protocol-based dependencies. Build fakes that satisfy the Protocol structurally:

```python
@dataclass
class FakeDatabase:
    documents: dict[int, Document] = field(default_factory=dict)

    async def get_document(self, doc_id: int) -> Document | None:
        return self.documents.get(doc_id)

    async def save_document(self, doc: Document) -> None:
        self.documents[doc.id] = doc
```

### Property-Based Testing with Hypothesis

```python
from hypothesis import given, strategies as st

@given(st.text())
def test_sanitise_filename_removes_invalid_chars(filename: str) -> None:
    result = sanitise_filename(filename)
    assert all(c not in INVALID_CHARS for c in result)
```

---

## 13. Additional Idioms

### `NewType` for Branded Primitives

```python
from typing import NewType

DocumentId = NewType("DocumentId", int)
MessageId = NewType("MessageId", str)
Sha256 = NewType("Sha256", str)
```

### `functools.reduce` for Accumulation

```python
from functools import reduce

total = reduce(lambda acc, r: acc + r.score, results, 0.0)
```

### Walrus Operator for Filtered Transforms

```python
valid = [
    processed
    for doc in documents
    if (processed := try_process(doc)) is not None
]
```

### `contextlib.asynccontextmanager` for Resources

```python
@asynccontextmanager
async def db_connection(path: str) -> AsyncIterator[Connection]:
    conn = await aiosqlite.connect(path)
    try:
        yield conn
    finally:
        await conn.close()
```

### `Final` for Constants

```python
from typing import Final

MAX_CHUNK_SIZE: Final[int] = 500
STATUS_FILE: Final[str] = "hermes-status.json"
```

---

## Quick Reference Card

| Smell | Fix |
|-------|-----|
| Untyped function | Add all type annotations |
| `dict` for structured data | `@dataclass(frozen=True, slots=True)` |
| Mutable dataclass for domain type | `frozen=True` |
| `list` in frozen dataclass field | `tuple[T, ...]` |
| Accumulation `for` loop | List comprehension |
| `isinstance()` chain | `match/case` with `assert_never` |
| Magic string repeated > 1× | `StrEnum` |
| Bare `except:` or `except Exception: pass` | Narrow except, convert to Result |
| `unittest.mock.patch` for deps | Protocol + fake class |
| Function > 20 lines | Extract helper functions |
| Global mutable state | Module-level `Final` constants |
| `**kwargs: Any` | Explicit parameters or `TypedDict` |
| `type: ignore` | Fix the type error |
| `Optional[X]` / `Union[X, Y]` | `X \| None` / `X \| Y` (3.10+ syntax) |
| `if/elif` on types | `match/case` with class patterns |
| Dict access chain for JSON | `match/case` with mapping patterns |
| `args[0]`, `args[1:]` slicing | `match/case` with sequence patterns |
| `for` loop building dict | Dict comprehension |
| `for` loop building set | Set comprehension |
| `for` loop with `append` | List comprehension |
| Unparameterised `list`, `dict` | `list[str]`, `dict[str, int]` |
| `Callable` without types | `Callable[[ArgType], ReturnType]` |
