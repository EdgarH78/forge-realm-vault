---
Summary: AtlasForge's type-safety stance — type safety beats convenience, and the language defaults are too loose. Three hard preferences: no `null`/`undefined` for domain values, no `unknown` past a deserialization boundary, no `any` anywhere. Exceptions are narrow and named. Concrete states (empty array, sentinel, discriminated union) replace nullable optionals; precise types and discriminated unions replace `unknown` bags. The same rules apply to review and implementation — we want them to prevent bad shapes being written, not just catch them after the fact.
Tags: #type-safety #architecture #atlasforge #foundational
---

# Type Discipline

We pay for type safety up front so the system stays predictable downstream. The TypeScript defaults (`T | null`, `unknown`, `any`) are looser than this codebase tolerates. The rules below are stronger than the language's; treat them as foundational, not stylistic.

## 1. No `null` / `undefined` for domain values

Domain values do not opt into "no value". If absence is meaningful, encode it as a concrete state — a sentinel, an empty collection, or a discriminated union variant. The reader should never have to guess whether `null` means "missing", "loading", "not yet fetched", or "the answer is no".

```typescript
// ❌ Wrong — null collapses three different states into one
function findRoom(id: string): Room | null { ... }
interface Stage { artifact?: Artifact }   // optional bag

// ✅ Right — the variant names the state
function findRoom(id: string): Room | { kind: 'not-found' } { ... }
interface Stage { artifact: Artifact | NullArtifact }   // discriminated
```

Empty collections, empty strings, and factory-supplied defaults are usually better than `T | null`. A function that returns `Room[]` for "no matches" beats one that returns `Room[] | null`.

### Allowed exceptions (narrow)

- **Deserialization boundaries.** `JSON.parse`, DB rows, HTTP response bodies — these legitimately lack a value in the wire format. Wrap *immediately* into a domain type that no longer carries `null`/`undefined`. The rule is "one line of nullable, then narrow," not "nullable propagates through the codebase."
- **Third-party type signatures you cannot change.** Adapt them at the import boundary; do not propagate the nullable shape into the domain.

### Smell to flag

Defensive `if (!value) return` early-returns that mask the real invariant. If the value cannot legitimately be absent, type it non-optional and let the caller fail loudly at the contract boundary.

## 2. No `unknown` past a deserialization boundary

`unknown` is acceptable for exactly one line — the result of `JSON.parse` immediately before a validator narrows it to a precise domain type. After that, the codebase carries the narrowed type, not `unknown`.

```typescript
// ❌ Wrong — unknown leaks through the call graph
function handle(payload: unknown) { ... }
const data = JSON.parse(raw) as unknown;
process(data);                          // still unknown three frames in

// ✅ Right — narrow at the boundary, carry the precise type
const data: unknown = JSON.parse(raw);
const parsed = parsePaletteResponse(data);   // returns PaletteResponse
process(parsed);
```

### Allowed exceptions (narrow)

- The `JSON.parse` line itself.
- Generic library plumbing where the constraint is genuinely "any value" — a logger context bag, a serializer's `replacer` callback. These are leaves, not infrastructure to build on.

### Never acceptable

- Carrying `unknown` through three or more call frames.
- `as unknown as X` casts to silence the compiler. Fix the type hierarchy (see [[Architecture-Determinism]] and [[Architecture-Decoupling]] §2 Interface-First).

## 3. No `any`

Hard rule. No exceptions in domain code, no exceptions in adapters, no exceptions in tests. Includes:

- Explicit `: any` annotations.
- `Record<string, any>`.
- `as any`.
- Function parameters typed `any`.
- `@ts-ignore` / `@ts-expect-error` comments that hide an `any`.

The fix is almost always a discriminated union, a small generic, or a branded type — see §4.

## 4. Type-system shapes we prefer

When you reach for a nullable optional or an `unknown` bag, consider these alternatives first:

- **Discriminated unions over optional bags.** `interface X { a?: A; b?: B; c?: C }` where consumers `if (x.a)` everywhere is a union pretending to be a struct. Replace with a discriminated union keyed on `kind`.
- **Branded types for ids.** `function f(roomId: string)` next to `function g(fixtureId: string)` is a swap waiting to happen. Brand them so the type system catches the mistake.
- **`Result` / `Either` for failure.** When a function can fail in a domain-meaningful way, return a discriminated `{ ok: true, value } | { ok: false, error }` instead of `T | null` + a throw.
- **Concrete state variants over `loading: boolean; error: Error | null; data: T | null`.** That three-field shape encodes nine reachable combinations and only four legal ones. A `{ kind: 'idle' } | { kind: 'loading' } | { kind: 'error', error } | { kind: 'data', data }` keeps the state machine honest.
- **`Partial<T>` is almost always wrong as a function parameter.** Define the actual subset of fields the function reads. `Partial<T>` smuggles the same nullable-everywhere shape `null`/`undefined` would.
- **Factories over constructors for foundational types.** A factory can enforce invariants (no malformed ids, no zero-length vectors) that a raw constructor cannot. See [[Architecture-Determinism]] §3.

## 5. Why this lives in the vault, not the skill catalogue

These rules apply to **both** writing code and reviewing it. Encoding them only in a review skill catches bad shapes after the fact; encoding them as a compass atom means the same standard governs the `implement` and `review-atlasforge` skills, and any other surface that consults the vault. Update this atom rather than restating its rules in a skill — the skill should cite [[Type-Discipline]], not duplicate it.

## 6. When the rule is wrong

These preferences are stronger than language defaults, not absolute. A new exception clause belongs *here*, in this atom, with a one-paragraph justification and a named scope — not silently in code. If you find yourself reaching for `null` or `unknown` in domain code, the question is "should this atom be amended?" first, "should I bypass it?" never.
