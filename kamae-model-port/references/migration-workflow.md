# Migration Workflow

> **When to read:** Starting a porting task. This file defines the recommended
> order of work and common pitfalls for cross-language kamae model migrations.

---

## Recommended Order of Porting

Each step builds on the previous. Do not skip ahead — later steps depend on
types defined in earlier ones.

### Step 1: Port State Types

Translate each state variant and the top-level discriminated union. This is the
foundation everything else depends on.

**What to port:**
- Each state variant (e.g., `Waiting`, `EnRoute`, `InTrip`, `Completed`, `Cancelled`)
- The union type that groups them
- The discriminant mechanism

**Language-specific targets:**

| Source | Target | Translation |
| --- | --- | --- |
| TS `type Waiting = Readonly<{ kind: "Waiting"; ... }>` | Python | `class Waiting(DomainModel): kind: Literal["waiting"] = "waiting"` |
| TS `type Waiting = Readonly<{ kind: "Waiting"; ... }>` | Rust | `pub struct WaitingRequest { /* private fields */ }` |
| TS `type Waiting = Readonly<{ kind: "Waiting"; ... }>` | Scala | `final case class WaitingRequest(...)` |
| TS `type TaxiRequest = Waiting \| EnRoute \| ...` | Python | `TaxiRequest = Annotated[Waiting \| EnRoute \| ..., Field(discriminator="kind")]` |
| TS `type TaxiRequest = Waiting \| EnRoute \| ...` | Rust | `pub enum TaxiRequest { Waiting(WaitingRequest), EnRoute(EnRouteRequest), ... }` |
| TS `type TaxiRequest = Waiting \| EnRoute \| ...` | Scala | `enum TaxiRequest { case Waiting(v: WaitingRequest), ... }` |

**Verification:** Each variant has ONLY the fields relevant to that state. No
optional fields that are "sometimes present" — that defeats the state machine.

### Step 2: Port ID and Value Types

Translate branded types / newtypes / opaque types for domain identifiers and
value objects. These are the building blocks used in state type fields.

**What to port:**
- All branded/newtype ID types (e.g., `RequestId`, `PassengerId`, `DriverId`)
- Value objects with invariants (e.g., `Location`, `Money`)
- Validation logic in constructors/factories

**Language-specific targets:**

| Source | Target | Translation |
| --- | --- | --- |
| TS `z.string().uuid().brand()` | Python | `class RequestId(DomainModel): value: UUID` |
| TS `z.string().uuid().brand()` | Rust | `pub struct RequestId(Uuid); impl RequestId { pub fn new(v: Uuid) -> Result<Self, E> }` |
| TS `z.string().uuid().brand()` | Scala | `opaque type RequestId = String; object RequestId { def apply(v: String): Either[E, RequestId] }` |

**Verification:** Mixing up IDs causes a compile error. `RequestId` cannot be
assigned where `PassengerId` is expected, in every target language.

### Step 3: Port Transition Functions

Translate pure state transition functions. These are the core domain logic.

**What to port:**
- Each transition function (e.g., `assignDriver`, `startTrip`, `complete`, `cancel`)
- Input type narrowing (transition accepts only the valid source state)
- Output type specificity (transition returns the exact target state, not the union)

**Language-specific targets:**

| Source | Target | Translation |
| --- | --- | --- |
| TS Companion Object method | Python | Module-level function beside the type |
| TS Companion Object method | Rust | `impl` method on the source state struct, consuming `self` |
| TS Companion Object method | Scala | Companion `object` method or `extension` method |

**Verification:** Invalid transitions do not compile. You cannot call
`assignDriver` on an `EnRoute` state — the type system rejects it.

### Step 4: Port Error Types

Translate error discriminated unions / enums / ADTs and the Result type usage.

**What to port:**
- Per-use-case error types
- Domain error variants (e.g., `RequestNotFound`, `InvalidState`, `ConcurrencyConflict`)
- Result type wrapping in transition and use case return types
- Exhaustive handling at match sites

**Language-specific targets:**

| Source | Target | Translation |
| --- | --- | --- |
| TS `type AssignError = { kind: "NotFound" } \| { kind: "NotWaiting" }` | Python | `class NotFound(DomainModel): kind: Literal["not_found"]` + union type |
| TS `type AssignError = ...` | Rust | `#[derive(thiserror::Error)] enum AssignError { #[error("...")] NotFound, ... }` |
| TS `type AssignError = ...` | Scala | `enum AssignError { case NotFound, case NotWaiting }` |
| TS `Result<T, E>` (neverthrow) | Python | Explicit checks / custom `Result` type / `returns` library |
| TS `Result<T, E>` (neverthrow) | Rust | `Result<T, E>` (std) |
| TS `Result<T, E>` (neverthrow) | Scala | `Either[E, T]` |

**Verification:** Exhaustive handling is required at call sites. Adding a new
error variant forces all callers to handle it (compiler error in Rust/Scala,
`assertNever`/`assert_never` in TS/Python).

### Step 5: Port Boundary DTOs

Create DTO types for external input/output and wire up validation.

**What to port:**
- Inbound DTO types with their schema/serde annotations
- Validation pipeline (schema parse, TryFrom, factory method)
- Outbound response DTOs
- Error mapping for validation failures

See [`boundary-and-dto.md`](./boundary-and-dto.md) for detailed patterns per
language.

**Verification:** Untrusted data cannot bypass validation into domain types. No
`as` cast (TS), no `model_construct` (Python), no direct construction (Rust),
no `asInstanceOf` (Scala).

### Step 6: Port Repository Ports

Translate interfaces / protocols / traits for persistence.

**What to port:**
- Read port (Resolver) and write port (Store)
- Atomic `save(state, events)` signature
- Return types with proper error wrapping

See [`persistence-event-mapping.md`](./persistence-event-mapping.md) for
detailed patterns per language.

**Verification:** Dual-write is structurally impossible. The only way to persist
is through `save(state, events)`, which the repository implementation wraps in
a single transaction.

### Step 7: Port Use Cases

Translate the orchestration layer that ties together resolver, transition, event
creation, and store.

**What to port:**
- Command/query types
- Orchestration flow: load -> validate -> transition -> persist
- Error mapping between domain errors and use-case errors
- Authorization checks (if present in the use case)

**Adapt the composition style:**

| Language | Composition idiom |
| --- | --- |
| TypeScript | `andThen` / `map` chaining on `Result` or `ResultAsync` |
| Python | Sequential `await` with early return or match statements |
| Rust | `?` operator for early return on `Result` |
| Scala | `for`-comprehension on `Either` or `EitherT` |

**Verification:** Authorization checks are preserved. If the source use case
checks `isAuthorized(user, request)`, the target must too.

### Step 8: Port PII Protection

Apply the target language's redaction wrapper to sensitive fields.

**What to port:**
- Fields marked with `Sensitive<T>` (TS) / `Redacted[T]` (Python) /
  `Redacted<T>` (Rust) / `Redacted[T]` (Scala)
- Credential types (`SecretStr`, `SecretString`, etc.)
- Custom `toString` / `Debug` / `__repr__` implementations that redact

**Verification:** Sensitive fields cannot be accidentally logged. Calling
`console.log`, `print`, `dbg!`, or `println` on a domain object containing
PII shows `[REDACTED]`, not the raw value.

### Step 9: Run Quality Gates

Run the target language's full quality gate suite.

| Language | Commands |
| --- | --- |
| TypeScript | `npx tsc --noEmit && npx eslint . && npx vitest run` |
| Python | `uv run ruff check . && uv run ruff format --check . && uv run mypy . && uv run pytest` |
| Rust | `cargo fmt --check && cargo clippy -- -D warnings && cargo test` |
| Scala | `sbt scalafmtCheck scalafixAll compile test` |

Fix issues iteratively. Type errors in step 9 often reveal mistakes made in
earlier steps (e.g., a missing field, a wrong ID type, an incomplete match).

---

## Common Pitfalls

### Rust: Forgetting to make fields private

Public struct fields bypass invariants. If the source language enforced
invariants via a factory/constructor, the Rust struct fields MUST be private
with getter methods.

```rust
// WRONG — fields are public, invariants can be bypassed
pub struct RequestId {
    pub value: Uuid,  // anyone can write RequestId { value: Uuid::nil() }
}

// CORRECT — private field, validated constructor
pub struct RequestId(Uuid);

impl RequestId {
    pub fn new(value: Uuid) -> Result<Self, IdError> {
        if value.is_nil() { return Err(IdError::Nil); }
        Ok(Self(value))
    }

    pub fn value(&self) -> &Uuid { &self.0 }
}
```

### TypeScript: Using interface instead of type

`interface` allows declaration merging — a third-party module or test file can
silently extend your domain type. Always use `type` for domain types.

```typescript
// WRONG — declaration merging risk
interface Waiting {
  kind: "Waiting";
  requestId: RequestId;
}

// CORRECT — no merging possible
type Waiting = Readonly<{
  kind: "Waiting";
  requestId: RequestId;
}>;
```

### Python: Missing frozen=True

Without `frozen=True`, Pydantic models are mutable. State transitions that
return "new" instances can have their fields silently mutated afterward.

```python
# WRONG — mutable
class Waiting(BaseModel):
    kind: Literal["waiting"] = "waiting"
    request_id: RequestId

# CORRECT — immutable
class Waiting(BaseModel):
    model_config = ConfigDict(frozen=True)

    kind: Literal["waiting"] = "waiting"
    request_id: RequestId
```

### Scala: Exposing copy on invariant-bearing case classes

`case class` generates a `copy` method that can bypass factory validation.
If the type has invariants enforced in the companion `apply`, override or
restrict `copy`.

```scala
// RISKY — copy bypasses validation
final case class RequestId private (value: String)
// user can still call: validId.copy(value = "")

// SAFER — use an opaque type (no copy at all)
object RequestId:
  opaque type RequestId = String
  def apply(raw: String): Either[IdError, RequestId] =
    Either.cond(raw.nonEmpty, raw, IdError.Empty)
```

### All languages: Wire compatibility when existing contracts exist

When porting a service that already has live wire contracts (API clients, event
consumers, message queues), changing the discriminant convention can break
compatibility. For example, porting from TypeScript (`"Waiting"`) to Python
(`"waiting"`) changes the wire value downstream consumers expect.

**Rule:** The internal domain discriminant convention follows the target
language's kamae idiom, but the DTO boundary layer MUST preserve the original
wire format. Use explicit serialization aliases (`@field_serializer` in Pydantic,
`#[serde(rename)]` in Rust, Circe custom codecs in Scala) to map between the
internal convention and the wire contract.

See `kamae-model-bridge` for full wire-format conventions and contract testing
patterns.

### All languages: Naming convention translation errors

When porting, field names must change to match the target language convention.
Missing a rename causes confusion and potentially breaks serialization.

| Source (TS) | Python | Rust | Scala |
| --- | --- | --- | --- |
| `requestId` | `request_id` | `request_id` | `requestId` |
| `assignedAt` | `assigned_at` | `assigned_at` | `assignedAt` |
| `driverId` | `driver_id` | `driver_id` | `driverId` |
| `"Waiting"` (discriminant) | `"waiting"` | N/A (variant name) | N/A (case name) |

### All languages: Porting test fixtures incorrectly

TS uses `as const satisfies` for type-safe test fixtures. Each target language
has its own equivalent.

```typescript
// TS fixture
const waitingFixture = {
  kind: "Waiting",
  requestId: "550e8400-e29b-41d4-a716-446655440000",
  passengerId: "660e8400-e29b-41d4-a716-446655440001",
  createdAt: new Date("2024-01-01T00:00:00Z"),
} as const satisfies Waiting;
```

```python
# Python equivalent — use the domain constructor
waiting_fixture = Waiting(
    request_id=RequestId(value=UUID("550e8400-e29b-41d4-a716-446655440000")),
    passenger_id=PassengerId(value=UUID("660e8400-e29b-41d4-a716-446655440001")),
    created_at=datetime(2024, 1, 1, tzinfo=timezone.utc),
)
```

```rust
// Rust equivalent — use the test helper or constructor
fn waiting_fixture() -> WaitingRequest {
    WaitingRequest::new(
        RequestId::new(Uuid::parse_str("550e8400-e29b-41d4-a716-446655440000").unwrap()).unwrap(),
        PassengerId::new(Uuid::parse_str("660e8400-e29b-41d4-a716-446655440001").unwrap()).unwrap(),
        Location::new(35.6812, 139.7671).unwrap(),
        Location::new(35.6595, 139.7006).unwrap(),
        Utc.with_ymd_and_hms(2024, 1, 1, 0, 0, 0).unwrap(),
    )
}
```

```scala
// Scala equivalent — use the domain constructor
val waitingFixture: WaitingRequest = WaitingRequest(
  requestId   = RequestId("550e8400-e29b-41d4-a716-446655440000").getOrElse(fail("bad id")),
  passengerId = PassengerId("660e8400-e29b-41d4-a716-446655440001").getOrElse(fail("bad id")),
  pickup      = Location(35.6812, 139.7671).getOrElse(fail("bad location")),
  destination = Location(35.6595, 139.7006).getOrElse(fail("bad location")),
  createdAt   = Instant.parse("2024-01-01T00:00:00Z"),
)
```

Always use the domain constructor in test fixtures — never bypass validation,
even in tests. Tests that bypass validation are testing a phantom that does not
exist in production.

---

## Checklist

Use this checklist to verify a completed port:

- [ ] Every state variant is present in the target language
- [ ] The discriminated union covers all variants
- [ ] All ID types are distinct (no raw strings/UUIDs used for multiple IDs)
- [ ] Transition functions accept only valid source states
- [ ] Transition functions return the exact target state (not the union)
- [ ] Error types cover all source error variants
- [ ] Exhaustive matching is enforced at all switch/match sites
- [ ] Boundary DTOs exist for all external inputs
- [ ] No `as`/`cast`/`asInstanceOf`/direct construction bypasses validation
- [ ] Repository port has atomic `save(state, events)` signature
- [ ] PII fields are wrapped in redaction types
- [ ] Naming conventions match the target language
- [ ] Quality gates pass (formatter, linter, type checker, tests)
