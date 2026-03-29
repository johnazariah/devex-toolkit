# Standard: Idiomatic TypeScript

> **Audience:** AI agents writing, reviewing, or refactoring TypeScript code.
> **Prerequisite:** Read `skills/repo-onboard/standards/code-quality.md` for Tagless-Final architecture and general rules.

## Core Principle

TypeScript is JavaScript with a world-class type system. Leverage the type system to **make illegal states unrepresentable**, use **pure functions and immutable data**, and prefer **composition over inheritance**. Code should read as declarative data transformations, not imperative DOM manipulation.

---

## 1. Strict Configuration

**Rule:** Every project starts with the strictest possible `tsconfig.json`. Non-negotiable settings:

```json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "noPropertyAccessFromIndexSignature": true,
    "forceConsistentCasingInFileNames": true,
    "verbatimModuleSyntax": true
  }
}
```

Never weaken these settings. If code doesn't compile under strict mode, fix the code, not the config.

---

## 2. Immutability by Default

**Rule:** Use `readonly` on all type properties. Use `as const` for literal objects. Prefer `ReadonlyArray<T>` or `readonly T[]` for arrays. Never mutate function parameters.

### Bad

```typescript
interface SearchResult {
  title: string;
  path: string;
  score: number;
  tags: string[];
}

function processResults(results: SearchResult[]) {
  results.sort((a, b) => b.score - a.score);  // mutates input!
  results.forEach(r => { r.title = r.title.trim(); });  // mutates items!
  return results;
}
```

### Good

```typescript
interface SearchResult {
  readonly title: string;
  readonly path: string;
  readonly score: number;
  readonly tags: readonly string[];
}

function processResults(results: readonly SearchResult[]): SearchResult[] {
  return [...results]
    .sort((a, b) => b.score - a.score)
    .map(r => ({ ...r, title: r.title.trim() }));
}
```

---

## 3. Discriminated Unions and Exhaustive Matching

**Rule:** Use discriminated unions (tagged unions) for types that can be one of several shapes. Every union must have a `readonly kind` (or `readonly type`) discriminant. Every `switch` on a union must be exhaustive — using `satisfies never` or a `default: assertNever()` helper.

### Bad

```typescript
interface ApiResponse {
  status: 'success' | 'error';
  data?: unknown;
  error?: string;
  retryAfter?: number;
}

function handle(response: ApiResponse) {
  if (response.status === 'success') {
    console.log(response.data);
  } else if (response.error) {
    console.error(response.error);
  }
}
```

### Good

```typescript
type ApiResponse<T> =
  | { readonly kind: 'success'; readonly data: T }
  | { readonly kind: 'error'; readonly message: string }
  | { readonly kind: 'rate-limited'; readonly retryAfterMs: number };

function handle<T>(response: ApiResponse<T>): void {
  switch (response.kind) {
    case 'success':
      console.log(response.data);
      break;
    case 'error':
      console.error(response.message);
      break;
    case 'rate-limited':
      console.warn(`Retry after ${response.retryAfterMs}ms`);
      break;
    default:
      response satisfies never;  // compile-time exhaustiveness check
  }
}
```

### `assertNever` helper

Define once and use in all switch defaults:

```typescript
function assertNever(value: never): never {
  throw new Error(`Unexpected value: ${JSON.stringify(value)}`);
}

// Usage
default:
  assertNever(response);  // runtime safety + compile-time check
```

### Nested Discriminated Unions

```typescript
type PipelineError =
  | { readonly kind: 'file-not-found'; readonly path: string }
  | { readonly kind: 'extraction-failed'; readonly path: string; readonly reason: string }
  | { readonly kind: 'duplicate'; readonly sha256: string }
  | { readonly kind: 'db-error'; readonly operation: string; readonly cause: Error };

function describeError(error: PipelineError): string {
  switch (error.kind) {
    case 'file-not-found': return `File not found: ${error.path}`;
    case 'extraction-failed': return `Extraction failed for ${error.path}: ${error.reason}`;
    case 'duplicate': return `Duplicate: ${error.sha256}`;
    case 'db-error': return `DB error during ${error.operation}: ${error.cause.message}`;
    default: assertNever(error);
  }
}
```

### When to Use Discriminated Unions

| Scenario | Use DU? |
|----------|---------|
| Function returns different shapes based on outcome | Yes — `Result<T, E>` |
| API response varies by status | Yes |
| Redux/state machine actions | Yes |
| Configuration variants | Yes |
| Simple boolean flag | No — just `boolean` |
| Optional value | No — `T \| undefined` |
```

---

## 4. Function Size and Composition

**Rule:** No function exceeds ~20 lines of logic. Compose via pure helper functions, `pipe()` chains, or `Array` method chains.

### Bad

```typescript
async function syncAccount(provider: EmailProvider, db: Database, account: string) {
  let downloaded = 0;
  let duplicates = 0;
  const errors: string[] = [];
  try {
    const lastSync = await loadSyncState(db, account);
    const messages = await provider.listMessages(lastSync);
    for (const msg of messages) {
      try {
        const exists = await messageExists(db, account, msg.providerId);
        if (!exists) {
          await recordMessage(db, account, msg);
          const attachments = await provider.getAttachments(msg.providerId);
          for (const att of attachments) {
            const sha = computeSha256(att.content);
            const isDup = await isDuplicate(db, sha);
            if (isDup) { duplicates++; }
            else { await saveAttachment(db, att); downloaded++; }
          }
        }
      } catch (e) { errors.push(String(e)); }
    }
  } catch (e) { errors.push(`Sync failed: ${e}`); }
  return { downloaded, duplicates, errors };
}
```

### Good

```typescript
async function processAttachment(
  db: Database, att: Attachment
): Promise<'downloaded' | 'duplicate'> {
  const sha = computeSha256(att.content);
  if (await isDuplicate(db, sha)) return 'duplicate';
  await saveAttachment(db, att);
  return 'downloaded';
}

async function processMessage(
  provider: EmailProvider, db: Database, account: string, msg: EmailMessage
): Promise<MessageResult> {
  if (await messageExists(db, account, msg.providerId))
    return { kind: 'skipped' };
  await recordMessage(db, account, msg);
  const attachments = await provider.getAttachments(msg.providerId);
  const outcomes = await Promise.all(attachments.map(a => processAttachment(db, a)));
  return {
    kind: 'processed',
    downloaded: outcomes.filter(o => o === 'downloaded').length,
    duplicates: outcomes.filter(o => o === 'duplicate').length,
  };
}

async function syncAccount(
  provider: EmailProvider, db: Database, account: string
): Promise<SyncResult> {
  const lastSync = await loadSyncState(db, account);
  const messages = await provider.listMessages(lastSync);
  const results = await Promise.allSettled(
    messages.map(m => processMessage(provider, db, account, m))
  );
  return aggregateResults(results);
}
```

---

## 5. No `any`, No `as` Casts — Type Narrowing Instead

**Rule:** Never use `any`. Never use `as` type assertions. Use type guards, `satisfies`, `unknown` with narrowing, and discriminated union switches to achieve type safety.

### Bad

```typescript
function parseConfig(raw: any): Config {
  return raw as Config;
}
```

### Good

```typescript
function parseConfig(raw: unknown): Config {
  if (!isValidConfig(raw)) throw new Error('Invalid config');
  return raw;
}

function isValidConfig(raw: unknown): raw is Config {
  return typeof raw === 'object' && raw !== null
    && 'archiveDir' in raw && typeof raw.archiveDir === 'string';
}
```

### `satisfies` for Literal Validation

```typescript
const config = {
  archiveDir: '~/Documents/Hermes',
  pollIntervalMs: 300_000,
} satisfies Config;
```

### Custom Type Guards

Write named type guard functions for any non-trivial narrowing:

```typescript
function isEmailMessage(obj: unknown): obj is EmailMessage {
  return (
    typeof obj === 'object' && obj !== null &&
    'providerId' in obj && typeof obj.providerId === 'string' &&
    'subject' in obj && typeof obj.subject === 'string' &&
    'sender' in obj && typeof obj.sender === 'string'
  );
}

function isNonNullable<T>(value: T): value is NonNullable<T> {
  return value !== null && value !== undefined;
}

// Usage: filter + narrow in one step
const validDocs = documents.map(tryParse).filter(isNonNullable);
```

### `in` Narrowing for Discriminated Unions

```typescript
function processEvent(event: MessageEvent | AttachmentEvent) {
  if ('body' in event) {
    // TypeScript narrows to MessageEvent
    console.log(event.body);
  } else {
    // TypeScript narrows to AttachmentEvent
    console.log(event.filename);
  }
}
```

### When to Use Each Narrowing Technique

| Technique | Use When |
|-----------|----------|
| `switch` on discriminant | DU with `kind` field |
| `typeof` check | Primitive narrowing (`string`, `number`) |
| `in` operator | Object shape differentiation |
| `instanceof` | Class instances (avoid for interfaces) |
| Custom type guard (`is`) | Complex validation, reusable narrowing |
| `satisfies` | Validate literal shape at definition |
| `as const` | Lock down literal types |

---

## 6. Array Methods as Comprehensions

**Rule:** TypeScript has no list comprehension syntax. Array methods (`map`, `filter`, `reduce`, `flatMap`) are your comprehensions. Use them consistently for all data transformations.

### Equivalence Table (Python → TypeScript)

| Python | TypeScript |
|--------|------------|
| `[f(x) for x in items]` | `items.map(x => f(x))` |
| `[x for x in items if pred(x)]` | `items.filter(pred)` |
| `[f(x) for x in items if pred(x)]` | `items.filter(pred).map(f)` |
| `{x.key: x.val for x in items}` | `Object.fromEntries(items.map(x => [x.key, x.val]))` |
| `{x.id for x in items}` | `new Set(items.map(x => x.id))` |
| `[y for x in items for y in x.children]` | `items.flatMap(x => x.children)` |
| `sum(x.score for x in items)` | `items.reduce((acc, x) => acc + x.score, 0)` |
| `any(pred(x) for x in items)` | `items.some(pred)` |
| `all(pred(x) for x in items)` | `items.every(pred)` |

### Filter + Type Guard (Narrowing)

The most powerful pattern — filter and narrow types simultaneously:

```typescript
// Remove nulls and narrow type
const documents: Document[] = rawDocs
  .map(parseDocument)
  .filter(isNonNullable);  // type is narrowed from (Document | undefined)[] to Document[]

// Filter by discriminant and narrow
const errors: PipelineError[] = results
  .filter((r): r is Err<PipelineError> => !r.ok)
  .map(r => r.error);
```

### `flatMap` for Flatten + Transform

```typescript
// Bad
const allAttachments: Attachment[] = [];
for (const msg of messages) {
  for (const att of msg.attachments) {
    allAttachments.push(att);
  }
}

// Good
const allAttachments = messages.flatMap(msg => msg.attachments);
```

### `reduce` for Accumulation

```typescript
// Group by category
const byCategory = documents.reduce(
  (acc, doc) => {
    const key = doc.category;
    return { ...acc, [key]: [...(acc[key] ?? []), doc] };
  },
  {} as Record<string, Document[]>
);

// Or use Map for better performance
const byCategoryMap = documents.reduce(
  (acc, doc) => acc.set(doc.category, [...(acc.get(doc.category) ?? []), doc]),
  new Map<string, Document[]>()
);
```

### Method Chain Length

- **1–4 stages:** Single chain.
- **5–7 stages:** Add comments between logical groups.
- **8+ stages:** Extract into named functions.

### When to Use Loops vs Array Methods

| Use Array Methods | Use `for` Loop |
|-------------------|----------------|
| Pure transforms (map, filter, reduce) | Side effects (I/O, logging) |
| Building new arrays | `await` inside the loop |
| Chaining transformations | Early exit (`break`) |
| Type narrowing via filter + guard | Performance-critical inner loops |

---

## 6. Tagless-Final Architecture

**Rule:** Define capabilities as interfaces with abstract effect types. The TypeScript approach uses generic interfaces or object-of-functions records.

### Interface Approach

```typescript
interface EmailProvider {
  readonly listMessages: (since?: Date) => Promise<readonly EmailMessage[]>;
  readonly getAttachments: (messageId: string) => Promise<readonly Attachment[]>;
}

interface DocumentExtractor {
  readonly extractPdf: (content: Uint8Array) => Promise<Result<string>>;
  readonly extractImage: (content: Uint8Array) => Promise<Result<string>>;
}
```

### Fakes for Testing

```typescript
const fakeEmailProvider: EmailProvider = {
  listMessages: async () => [],
  getAttachments: async () => [],
};

const fakeExtractor: DocumentExtractor = {
  extractPdf: async () => ({ ok: true, value: 'extracted text' }),
  extractImage: async () => ({ ok: true, value: 'ocr text' }),
};
```

### Composition Root

```typescript
function createApp(deps: {
  readonly emailProvider: EmailProvider;
  readonly extractor: DocumentExtractor;
  readonly db: Database;
}) {
  return {
    sync: () => syncAccount(deps.emailProvider, deps.db, 'default'),
    extract: (path: string) => processDocument(deps.extractor, deps.db, path),
  };
}
```

---

## 7. Result Type

**Rule:** Use a `Result<T, E>` discriminated union for operations that can fail. Never throw for expected failures.

```typescript
type Result<T, E = string> =
  | { readonly ok: true; readonly value: T }
  | { readonly ok: false; readonly error: E };

const ok = <T>(value: T): Result<T, never> => ({ ok: true, value });
const err = <E>(error: E): Result<never, E> => ({ ok: false, error });

function map<T, U, E>(result: Result<T, E>, f: (t: T) => U): Result<U, E> {
  return result.ok ? ok(f(result.value)) : result;
}

function bind<T, U, E>(result: Result<T, E>, f: (t: T) => Result<U, E>): Result<U, E> {
  return result.ok ? f(result.value) : result;
}
```

### When to Throw vs Return Result

| Scenario | Approach |
|----------|----------|
| File not found, parse error, validation failure | `Result<T, E>` |
| Network timeout, disk full, OOM | Throw (truly exceptional) |
| Programmer error (assert invariant) | Throw `Error` |

---

## 8. Module Organisation

**Rule:** One concept per file. Files ≤200 lines. Use barrel exports (`index.ts`) sparingly — only at package boundaries.

### Structure

```
src/
├── domain/          — types, discriminated unions, value objects
├── capabilities/    — Tagless-Final interfaces
├── providers/       — concrete implementations
├── pipeline/        — orchestration logic
├── infrastructure/  — DB, HTTP, filesystem adapters
├── utils/           — pure utility functions
└── index.ts         — public API surface
```

### Exports

- **Named exports only.** Never `export default`.
- **Re-export from barrel files** only at package boundaries.
- **Type-only imports** for types: `import type { Config } from './domain/config.js';`

---

## 9. Async Patterns

### Always Return Promises

```typescript
// Bad — mixed sync/async
function getDocument(id: number) {
  if (cache.has(id)) return cache.get(id)!;  // sync return
  return db.query(id);  // Promise return
}

// Good — always async
async function getDocument(id: number): Promise<Document | undefined> {
  return cache.get(id) ?? await db.query(id);
}
```

### `Promise.all` for Independent Work

```typescript
const [messages, config, lastSync] = await Promise.all([
  provider.listMessages(),
  loadConfig(),
  loadSyncState(db, account),
]);
```

### `AbortSignal` for Cancellation

```typescript
async function fetchWithTimeout(
  url: string, signal?: AbortSignal
): Promise<Response> {
  return fetch(url, { signal });
}
```

---

## 10. Branded Types

**Rule:** For IDs, paths, and other primitive-wrapped values, use branded types to prevent accidental mixing.

```typescript
type Brand<T, B extends string> = T & { readonly __brand: B };

type DocumentId = Brand<number, 'DocumentId'>;
type MessageId = Brand<string, 'MessageId'>;
type Sha256 = Brand<string, 'SHA256'>;

function documentId(id: number): DocumentId { return id as DocumentId; }
function messageId(id: string): MessageId { return id as MessageId; }
```

This prevents passing a `DocumentId` where a `MessageId` is expected.

---

## 11. Testing

### Vitest / Jest

```typescript
describe('EmailSync', () => {
  it('records all new messages', async () => {
    const provider = fakeEmailProvider({
      messages: [testEmailMessage({ subject: 'Invoice' })],
    });
    const db = fakeDatabase();

    const result = await syncAccount(provider, db, 'test@example.com');

    expect(result.messagesProcessed).toBe(1);
  });
});
```

### Test Data Factories

```typescript
function testEmailMessage(overrides?: Partial<EmailMessage>): EmailMessage {
  return {
    providerId: 'msg-001',
    subject: 'Test',
    sender: 'test@example.com',
    date: new Date(),
    ...overrides,
  };
}
```

### Fakes Over Mocks

Never use `jest.mock()` or `vi.mock()` for Tagless-Final interfaces. Build fakes from the interface directly:

```typescript
function fakeEmailProvider(config?: {
  messages?: EmailMessage[];
}): EmailProvider {
  return {
    listMessages: async () => config?.messages ?? [],
    getAttachments: async () => [],
  };
}
```

---

## 12. Additional Idioms

### Const Enums for Fixed Sets

```typescript
const SourceType = {
  EmailAttachment: 'email_attachment',
  WatchedFolder: 'watched_folder',
  ManualDrop: 'manual_drop',
} as const;

type SourceType = (typeof SourceType)[keyof typeof SourceType];
```

### `using` for Resource Management (TC39 Stage 3)

```typescript
{
  using conn = await db.connect();
  const rows = await conn.query('SELECT * FROM documents');
  // conn is disposed at block exit
}
```

### Template Literal Types for Compile-Time Validation

```typescript
type HexColor = `#${string}`;
type EmailAddress = `${string}@${string}.${string}`;

function sendEmail(to: EmailAddress, subject: string): Promise<void> { ... }
```

### `Readonly<T>` Utility

```typescript
function freeze<T extends object>(obj: T): Readonly<T> {
  return Object.freeze(obj);
}
```

### Nullish Coalescing and Optional Chaining

```typescript
// Bad
const name = user && user.profile && user.profile.name ? user.profile.name : 'Anonymous';

// Good
const name = user?.profile?.name ?? 'Anonymous';
```

---

## Quick Reference Card

| Smell | Fix |
|-------|-----|
| `any` type | `unknown` + type guard |
| `as` type assertion | Type guard or `satisfies` |
| `export default` | Named export |
| Mutable interface properties | `readonly` on all properties |
| `let` for non-reassigned variable | `const` |
| `Array.push` in accumulation | `Array.map` / `Array.reduce` |
| `if/else if/else` chain | Discriminated union + `switch` |
| Thrown error for expected failure | `Result<T, E>` return type |
| `jest.mock()` for dependencies | Tagless-Final interface + fake |
| Magic string repeated > 1× | `as const` object + type extraction |
| Mixing sync and async returns | Always `async` / always `Promise<T>` |
| `function` in module scope | `const fn = (...) => ...` for non-hoisted, `function` for hoisted |
| Missing `AbortSignal` on I/O | Add `signal` parameter |
| `.then()` chains | `async/await` |
| `if/else` on string literal | Discriminated union + exhaustive `switch` |
| Missing `assertNever` in `default` | Add `default: assertNever(x)` |
| `Array.push` in `for` loop building list | `.map()` / `.filter()` / `.flatMap()` |
| Nested `for` loops flattening | `.flatMap()` |
| `for` loop with counter | `.reduce()` |
| `.filter().length > 0` | `.some()` |
| Manual null filtering | `.filter(isNonNullable)` with type guard |
| `typeof x === 'object'` chain | Custom type guard `(x): x is T =>` |
| Missing `satisfies` on config object | Add `satisfies Config` |
