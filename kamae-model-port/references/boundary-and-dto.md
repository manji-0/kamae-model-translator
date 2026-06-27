# Boundary Defense and DTO Mapping

> **When to read:** Porting boundary validation, DTO patterns, or external data
> parsing between kamae languages.

The "external data -> DTO -> domain" pipeline is a universal kamae pattern.
Every language enforces it differently, but the principle is the same: untrusted
data never touches domain types without passing through a validated boundary.

---

## The Inbound Pipeline

### TypeScript: Schema -> safeParse -> Result

TypeScript merges schema definition and type extraction. The schema IS the
source of truth; `z.infer` extracts the type.

```typescript
// 1. Define the DTO schema (accepts external data)
const CreateRequestDtoSchema = z.object({
  passengerId: z.string().uuid(),
  pickupLat: z.number(),
  pickupLng: z.number(),
  destinationLat: z.number(),
  destinationLng: z.number(),
});
type CreateRequestDto = z.infer<typeof CreateRequestDtoSchema>;

// 2. Parse at boundary — safeParse returns { success, data } | { success, error }
const parseCreateRequest = (raw: unknown): Result<CreateRequestDto, BoundaryError> =>
  schemaResult(CreateRequestDtoSchema, raw);

// 3. Map DTO to domain type via Companion Object's factory
const TaxiRequest = {
  create: (dto: CreateRequestDto, now: Date): Waiting => ({
    kind: "Waiting",
    requestId: RequestId.generate(),
    passengerId: PassengerId.parse(dto.passengerId),
    pickup: Location.create(dto.pickupLat, dto.pickupLng),
    destination: Location.create(dto.destinationLat, dto.destinationLng),
    createdAt: now,
  }),
} as const;
```

The `schemaResult` factory is a thin wrapper that converts Zod's `safeParse`
into the project's `Result` type (neverthrow, byethrow, etc.), normalizing
errors into a `BoundaryError` variant.

### Python: TypeAdapter / BaseModel -> validate -> mapper

Python separates the model definition (Pydantic `BaseModel`) from the parsing
entry point (`TypeAdapter`). Inbound DTOs are strict and forbid extras.

```python
from pydantic import BaseModel, ConfigDict, TypeAdapter


class CreateRequestDto(BaseModel):
    model_config = ConfigDict(strict=True, extra="forbid")

    passenger_id: str
    pickup_lat: float
    pickup_lng: float
    destination_lat: float
    destination_lng: float


_create_request_adapter = TypeAdapter(CreateRequestDto)


def parse_create_request(raw: bytes) -> CreateRequestDto:
    """Parse raw JSON bytes into a validated DTO.

    Raises ValidationError on invalid input — catch at the HTTP handler level.
    """
    return _create_request_adapter.validate_json(raw)


def to_waiting(dto: CreateRequestDto, now: datetime) -> Waiting:
    """Map validated DTO to domain Waiting state."""
    return Waiting(
        request_id=RequestId.generate(),
        passenger_id=PassengerId(value=UUID(dto.passenger_id)),
        pickup=Location(lat=dto.pickup_lat, lng=dto.pickup_lng),
        destination=Location(lat=dto.destination_lat, lng=dto.destination_lng),
        created_at=now,
    )
```

For `validate_python(raw_dict)` when the source is already a Python dict (e.g.,
from message queues), or `validate_json(body)` when the source is raw bytes
from HTTP.

### Rust: serde Deserialize on DTO -> TryFrom for domain

Rust uses a two-step approach: `serde::Deserialize` on the DTO struct, then
`TryFrom` (or a validated constructor) to convert into the domain type. Domain
types themselves must NOT derive `Deserialize`.

```rust
use serde::Deserialize;
use uuid::Uuid;

/// Inbound DTO — serde-aware, domain-unaware.
#[derive(Deserialize)]
#[serde(deny_unknown_fields)]
pub struct CreateRequestDto {
    pub passenger_id: Uuid,
    pub pickup_lat: f64,
    pub pickup_lng: f64,
    pub destination_lat: f64,
    pub destination_lng: f64,
}

/// Domain conversion — validates all invariants.
/// `now` is injected rather than calling `Utc::now()` directly,
/// keeping the boundary pure and testable (kamae time-injection principle).
pub fn to_waiting(
    dto: CreateRequestDto,
    now: DateTime<Utc>,
) -> Result<WaitingRequest, BoundaryError> {
    let passenger_id = PassengerId::new(dto.passenger_id)?;
    let pickup = Location::new(dto.pickup_lat, dto.pickup_lng)?;
    let destination = Location::new(dto.destination_lat, dto.destination_lng)?;

    Ok(WaitingRequest::new(
        RequestId::generate(),
        passenger_id,
        pickup,
        destination,
        now,
    ))
}
```

Keep `Serialize` and `Deserialize` off domain types. Only DTOs (inbound and
outbound) should be serde-aware.

### Scala: Circe/Play decoder on DTO -> validated factory

Scala uses a JSON codec on the DTO case class, then a validated factory
returning `Either[BoundaryError, DomainType]`.

```scala
import io.circe.{Decoder, HCursor}
import io.circe.generic.semiauto.deriveDecoder

final case class CreateRequestDto(
  passengerId: String,
  pickupLat:   Double,
  pickupLng:   Double,
  destinationLat: Double,
  destinationLng: Double,
)

object CreateRequestDto:
  given decoder: Decoder[CreateRequestDto] = deriveDecoder

  def toWaiting(dto: CreateRequestDto, now: Instant): Either[BoundaryError, WaitingRequest] =
    for
      pid    <- PassengerId(dto.passengerId)
      pickup <- Location(dto.pickupLat, dto.pickupLng)
      dest   <- Location(dto.destinationLat, dto.destinationLng)
    yield WaitingRequest(
      requestId   = RequestId.generate(),
      passengerId = pid,
      pickup      = pickup,
      destination = dest,
      createdAt   = now,
    )
```

The `for`-comprehension short-circuits on the first `Left`, accumulating all
validation through the `Either` chain. For parallel validation (accumulate all
errors), use `Validated` from Cats.

---

## Forbidden Practices

These are preserved across ALL ports. Violating them in the target language
defeats the boundary guarantee the source language was enforcing.

| Language | Forbidden at Boundary | Why |
| --- | --- | --- |
| TypeScript | `as` cast (except `as const`) | Silently lies to the compiler; untrusted data passes unchecked |
| Python | `typing.cast`, `model_construct()` on untrusted data | Bypasses Pydantic validation entirely |
| Rust | Direct struct construction bypassing `new()` / `TryFrom` | Skips invariant checks; public fields make this easy to miss |
| Scala | `asInstanceOf`, public `copy` on invariant-bearing types | Circumvents factory validation; `copy` can violate invariants |

When porting, verify the target language's boundary is at least as strict as the
source. A TS `safeParse` must not become a Python `model_construct`; a Rust
`TryFrom` must not become a Scala `asInstanceOf`.

---

## Extra Field Handling

| Language | Default Behavior | Configuration |
| --- | --- | --- |
| TypeScript | Zod `.object()` strips unknown fields by default; use `.strict()` to reject | `.passthrough()` to preserve extras (rarely used in domain) |
| Python | Pydantic allows extra fields by default | `extra="forbid"` on domain models; `extra="ignore"` on DTOs when appropriate |
| Rust | serde ignores unknown fields by default | `#[serde(deny_unknown_fields)]` on DTOs to reject |
| Scala | Depends on codec; Circe rejects by default | Configure via `Decoder` options or explicit policy |

Porting note: when moving from TS (rejects by default) to Rust (ignores by
default), you MUST add `#[serde(deny_unknown_fields)]` to preserve the same
strictness. Missing this is a common porting bug.

---

## Environment and Configuration Validation

Configuration is also a boundary. The same DTO-to-domain pipeline applies.

### TypeScript

```typescript
const AppConfigSchema = z.object({
  DATABASE_URL: z.string().url(),
  PORT: z.coerce.number().int().min(1).max(65535),
  LOG_LEVEL: z.enum(["debug", "info", "warn", "error"]),
});
type AppConfig = z.infer<typeof AppConfigSchema>;

const config = AppConfigSchema.parse(process.env);
```

### Python

```python
from pydantic_settings import BaseSettings


class AppConfig(BaseSettings):
    model_config = ConfigDict(extra="forbid")

    database_url: str
    port: int = Field(ge=1, le=65535)
    log_level: Literal["debug", "info", "warn", "error"] = "info"

config = AppConfig()  # reads from environment automatically
```

### Rust

```rust
use config::Config;
use serde::Deserialize;

#[derive(Deserialize)]
#[serde(deny_unknown_fields)]
struct AppConfigDto {
    database_url: String,
    port: u16,
    log_level: String,
}

impl TryFrom<AppConfigDto> for AppConfig {
    type Error = ConfigError;

    fn try_from(dto: AppConfigDto) -> Result<Self, Self::Error> {
        let log_level = LogLevel::from_str(&dto.log_level)?;
        Ok(AppConfig { database_url: dto.database_url, port: dto.port, log_level })
    }
}

let raw: AppConfigDto = Config::builder()
    .add_source(config::Environment::default())
    .build()?
    .try_deserialize()?;
let config = AppConfig::try_from(raw)?;
```

### Scala

```scala
import pureconfig.*
import pureconfig.generic.semiauto.*

final case class AppConfigDto(
  databaseUrl: String,
  port: Int,
  logLevel: String,
)

object AppConfigDto:
  given ConfigReader[AppConfigDto] = deriveReader

  def toDomain(dto: AppConfigDto): Either[ConfigError, AppConfig] =
    for
      level <- LogLevel.fromString(dto.logLevel)
      _     <- Either.cond(dto.port > 0 && dto.port <= 65535, (), ConfigError.InvalidPort(dto.port))
    yield AppConfig(databaseUrl = dto.databaseUrl, port = dto.port, logLevel = level)
```

---

## Outbound DTOs (Domain -> External)

The reverse direction also uses a DTO layer. Domain types are never directly
serialized.

| Language | Pattern |
| --- | --- |
| TypeScript | `toResponse(domain: EnRoute): EnRouteResponse` — plain mapper function returning a serializable object |
| Python | `class EnRouteResponse(BaseModel)` with a `@classmethod from_domain(cls, state: EnRoute)` factory |
| Rust | `#[derive(Serialize)] struct EnRouteResponse` with `impl From<&EnRouteRequest> for EnRouteResponse` |
| Scala | `final case class EnRouteResponse(...)` with `object EnRouteResponse { def from(state: EnRouteRequest): EnRouteResponse }` |

PII fields MUST be excluded or redacted in outbound DTOs. The mapper function
is the enforcement point.

---

## Key Porting Notes

1. **TS merges schema and type** via `z.infer`. When porting to Python, you will
   separate them into a `BaseModel` class and a `TypeAdapter`. When porting to
   Rust, you will separate them into a DTO struct (with `Deserialize`) and a
   domain struct (without).

2. **Python's `TypeAdapter`** is the closest equivalent to TS's `schemaResult`
   wrapper. Both take raw data and return a validated result. Rust's `serde`
   deserialization is the equivalent step but requires an explicit `TryFrom` to
   reach the domain type.

3. **Rust's two-step (DTO + TryFrom)** is the most explicit pattern. When
   porting FROM Rust, you can sometimes collapse the two steps in TS (schema
   parse) or Python (single model with validators), but keep the conceptual
   separation clear.

4. **Scala's `for`-comprehension** naturally chains `Either` validations. The TS
   equivalent is `andThen` chaining on `Result`; the Python equivalent is
   sequential checks with early return; the Rust equivalent is `?` operator on
   `Result`.

5. **Naming conventions shift at the boundary.** External APIs typically use
   `camelCase` or `snake_case` JSON. Configure serde rename, Pydantic aliases,
   or Circe key mappers to bridge between wire format and language convention.
