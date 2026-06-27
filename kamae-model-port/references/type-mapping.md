# Cross-Language Type Mapping

> **When to read:** Every porting task. This is the core reference for
> translating kamae domain constructs between TypeScript, Python, Rust, and Scala.

## Master Correspondence Table

| Concept | TypeScript | Python 3.12+ | Rust | Scala 3 |
| --- | --- | --- | --- | --- |
| **Discriminated union** | `type U = A \| B` with `kind` literal | `type U = Annotated[A \| B, Field(discriminator="kind")]` | `enum U { A(AState), B(BState) }` | `enum U { case A(v: AState), case B(v: BState) }` or sealed trait |
| **State variant** | `type A = Readonly<{ kind: "A"; ... }>` | `class A(DomainModel): kind: Literal["a"] = "a"` | `pub struct AState { ... }` (private fields) | `final case class AState(...)` |
| **Discriminant field** | `kind: "A"` (PascalCase literal) | `kind: Literal["a"] = "a"` (snake_case) | variant name in enum (no explicit field) | case name in enum (no explicit field) |
| **Union type alias** | `type TaxiRequest = Waiting \| EnRoute \| ...` | `type TaxiRequest = Annotated[Waiting \| EnRoute \| ..., Field(discriminator="kind")]` | `pub enum TaxiRequest { Waiting(WaitingState), ... }` | `enum TaxiRequest { case Waiting(v: WaitingState), ... }` |
| **Partial union** | `type Cancellable = Waiting \| EnRoute \| InTrip` | `type Cancellable = Waiting \| EnRoute \| InTrip` | trait bound or dedicated enum | union type `Waiting \| EnRoute \| InTrip` or sealed trait subset |

## Identity and Value Types

| Concept | TypeScript | Python | Rust | Scala |
| --- | --- | --- | --- | --- |
| **Branded ID** | `z.brand()` + Symbol + Companion | Frozen wrapper model `class XId(DomainModel): value: UUID` | Newtype `pub struct XId(String)` + private + `new()` | `opaque type XId = String` inside object + `apply()` |
| **ID validation** | Schema parse (`safeParse`) | `field_validator` on wrapper | `new()` returns `Result<Self, E>` | `apply()` returns `Either[E, XId]` |
| **Companion Object** | Same-name `type` + `const` pair | Module-level functions beside the type | `impl` block on the type | `object` companion or `extension` methods |
| **Value Object** | `Readonly<{ amount: number; currency: string }>` | `@dataclass(frozen=True, slots=True)` or frozen Pydantic | Struct with private fields + `new()` | `final case class` or opaque type |

## Immutability

| Concept | TypeScript | Python | Rust | Scala |
| --- | --- | --- | --- | --- |
| **Enforcement** | `Readonly<{}>` wrapper | `ConfigDict(frozen=True)` / `@dataclass(frozen=True)` | Ownership + private fields (no `pub` on internals) | `final case class` (immutable by default) |
| **State change** | Return new object literal | Construct new model instance | Consume `self`, return new struct | Return new case class instance |

## Error Handling

| Concept | TypeScript | Python | Rust | Scala |
| --- | --- | --- | --- | --- |
| **Result type** | Library: `neverthrow` / `fp-ts` / `byethrow` / `option-t` | Library: `result` / `returns` / local `Ok`/`Err` | `Result<T, E>` (std) | `Either[E, T]` (std or cats) |
| **Error variants** | DU with `kind` field | Pydantic DU with `kind` Literal | `enum` + `#[derive(thiserror::Error)]` | `enum` ADT |
| **Error granularity** | Per-use-case error type | Per-use-case error type | Per-use-case error enum | Per-use-case error enum |
| **Exhaustive check** | `assertNever(x)` in default | `assert_never(x)` in match `_` | Exhaustive `match` (compiler) | Exhaustive `match` (compiler warns) |
| **Forbidden** | `throw` in domain code | `raise` in domain code | `panic!` / `unwrap()` in domain code | `throw` / `.get` / `.head` in domain code |

## Boundary Defense

| Concept | TypeScript | Python | Rust | Scala |
| --- | --- | --- | --- | --- |
| **Schema validation** | Zod / Valibot / ArkType | Pydantic `TypeAdapter.validate_python` | `serde::Deserialize` on DTO | Circe / Play JSON decoder on DTO |
| **DTO → Domain** | Schema parse output is domain type (branded) | `TypeAdapter` output or explicit mapper | `TryFrom<Dto>` for domain type | Validated `apply` / `from` factory |
| **Forbidden** | `as` cast (except `as const`) | `typing.cast` / `model_construct` on untrusted data | Direct construction bypassing `new()` | `asInstanceOf` / public copy on invariant types |
| **Extra fields** | Schema rejects by default | `extra="forbid"` on domain; configurable on DTOs | `#[serde(deny_unknown_fields)]` on DTOs | Reject by codec or explicit policy |

## PII Protection

| Concept | TypeScript | Python | Rust | Scala |
| --- | --- | --- | --- | --- |
| **Redaction wrapper** | `Sensitive<T>` (closure-based) | `Redacted[T]` (generic frozen model) | `Redacted<T>` (custom Debug/Display) | `Redacted[T]` or opaque type with safe `toString` |
| **Credential type** | `Sensitive<T>` (same wrapper) | `SecretStr` / `SecretBytes` (Pydantic) | `secrecy::SecretString` / `SecretBox<T>` | Opaque `ApiToken` with hidden `toString` |
| **Serialization** | `toJSON()` returns `"[REDACTED]"` | `__repr__` / `__str__` return masked value | `Debug` impl redacts | `toString` override redacts |
| **Exposure** | `unwrap()` | Named method or adapter access | Named `expose_*` method | Named `exposeFor*` method |

## Persistence and Events

| Concept | TypeScript | Python | Rust | Scala |
| --- | --- | --- | --- | --- |
| **Transition outcome** | Return new state (plain value) | `TransitionOutcome[TState, TEvent]` | `Transition<TState> { state, events: Vec<ConcreteEvent> }` | `Transition[TState, TEvent](state, events)` |
| **Repository port** | `type` with function properties | `typing.Protocol` | `#[async_trait] trait` | `trait[F[_]]` |
| **Event model** | DU with `eventName` discriminant | Frozen Pydantic with `event_name` Literal | Enum variant (struct-like) | Enum ADT case class |
| **Atomic persist** | `save(state, events)` single method | `save(state, events, expected_version)` | `save(state, events)` | `saveAssigned(state, events)` |

> **Rust vs Scala/Python `Transition` type params:** Rust uses one type parameter
> (state only); the event vector uses a concrete enum type. Python and Scala use
> two type parameters (state and event). This is an intentional idiom difference,
> not a porting oversight.

## File Organization

| Concept | TypeScript | Python | Rust | Scala |
| --- | --- | --- | --- | --- |
| **One concept per file** | `request-id.ts`, `taxi-request.ts` | `request_id.py`, `taxi_request.py` | `request_id.rs`, `taxi_request.rs` | `RequestId.scala` or `taxi/request/` package |
| **Barrel/re-export** | `index.ts` (re-exports only) | `__init__.py` | `mod.rs` / `lib.rs` | Package object or top-level exports |
| **Catch-all forbidden** | No `types.ts` / `models.ts` | No `models.py` / `types.py` | No `types.rs` / `models.rs` | No `models/` mixing unrelated concepts |

## Naming Conventions

| Element | TypeScript | Python | Rust | Scala |
| --- | --- | --- | --- | --- |
| Type/class name | PascalCase | PascalCase | PascalCase | PascalCase |
| Field name | camelCase | snake_case | snake_case | camelCase |
| Function name | camelCase | snake_case | snake_case | camelCase |
| Discriminant value | PascalCase (`"Waiting"`) | snake_case (`"waiting"`) | N/A (enum variant) | N/A (enum case) |
| File name | kebab-case (`taxi-request.ts`) | snake_case (`taxi_request.py`) | snake_case (`taxi_request.rs`) | PascalCase or package path |
| Module/package | kebab-case | snake_case | snake_case | dot-separated lowercase |

## Translation Decision Tree

When porting a concept, follow this decision flow:

1. **Does the target language have a direct equivalent?** Use the mapping table above.
2. **Is there a library-specific pattern?** Check the target kamae skill's library guides.
3. **Does the direct translation fight the target language's idioms?** Adapt the pattern while preserving: no invalid states, no invalid transitions, validated boundaries.
4. **Is the concept absent in the target language?** (e.g., branded types in Rust have newtypes, not brands) Use the closest mechanism that provides the same compile-time or runtime guarantee.
