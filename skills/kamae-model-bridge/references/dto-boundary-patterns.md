# DTO Boundary Patterns

> **When to read:** Designing outbound and inbound DTOs for cross-service model
> exchange. This reference covers the structural patterns for converting domain
> models to wire-compatible DTOs on the sender side and parsing wire data back
> into domain models on the receiver side.

## Core Principle

Domain models never cross the wire directly. Each side maintains its own DTOs
that represent the wire contract. The DTO is a serialization-focused type; the
domain model is a business-logic-focused type. Conversion functions bridge the
two at every service boundary.

## Sender Side: Domain -> Outbound DTO -> Wire

The sender converts its internal domain model into a wire-compatible DTO, then
serializes that DTO. The outbound DTO owns the snake_case field names and
snake_case `kind` values required by the wire format.

### TypeScript

```typescript
// Domain model (camelCase, branded IDs)
type EnRoute = {
  kind: "EnRoute";
  requestId: RequestId;
  passengerId: PassengerId;
  driverId: DriverId;
  assignedAt: Date;
};

// Wire DTO (snake_case fields, snake_case kind)
type EnRouteWireDto = {
  kind: "en_route";
  request_id: string;
  passenger_id: string;
  driver_id: string;
  assigned_at: string;
};

const toWireDto = (enRoute: EnRoute): EnRouteWireDto => ({
  kind: "en_route",
  request_id: enRoute.requestId,
  passenger_id: enRoute.passengerId,
  driver_id: enRoute.driverId,
  assigned_at: enRoute.assignedAt.toISOString(),
});

// Serialize
const json = JSON.stringify(toWireDto(enRoute));
```

### Python

```python
class EnRouteWireDto(BaseModel):
    """Outbound DTO for the EnRoute state."""

    model_config = ConfigDict(strict=True, extra="forbid")

    kind: Literal["en_route"]
    request_id: UUID
    passenger_id: UUID
    driver_id: UUID
    assigned_at: datetime

def to_wire_dto(en_route: EnRoute) -> EnRouteWireDto:
    return EnRouteWireDto(
        kind="en_route",
        request_id=en_route.request_id,
        passenger_id=en_route.passenger_id,
        driver_id=en_route.driver_id,
        assigned_at=en_route.assigned_at,
    )

# Serialize
json_bytes = to_wire_dto(en_route).model_dump_json()
```

Python's snake_case matches the wire format naturally, so no alias or rename
is needed in the DTO.

### Rust

```rust
/// Outbound wire DTO for the EnRoute state.
/// Note: deny_unknown_fields is omitted here because it only affects
/// deserialization. On a Serialize-only outbound DTO it has no effect.
#[derive(serde::Serialize)]
pub struct EnRouteWireDto {
    pub kind: String,
    pub request_id: String,
    pub passenger_id: String,
    pub driver_id: String,
    pub assigned_at: String,
}

impl From<EnRoute> for EnRouteWireDto {
    fn from(en_route: EnRoute) -> Self {
        Self {
            kind: "en_route".to_owned(),
            request_id: en_route.request_id.to_string(),
            passenger_id: en_route.passenger_id.to_string(),
            driver_id: en_route.driver_id.to_string(),
            assigned_at: en_route.assigned_at.to_rfc3339(),
        }
    }
}

// Serialize
let json = serde_json::to_string(&EnRouteWireDto::from(en_route))?;
```

Rust's snake_case matches the wire format naturally, so no `#[serde(rename)]`
is needed.

### Scala

```scala
/** Outbound wire DTO for the EnRoute state. */
final case class EnRouteWireDto(
    kind: String,
    requestId: String,
    passengerId: String,
    driverId: String,
    assignedAt: String
)

object EnRouteWireDto:
  def fromDomain(enRoute: EnRoute): EnRouteWireDto =
    EnRouteWireDto(
      kind = "en_route",
      requestId = enRoute.requestId.value.toString,
      passengerId = enRoute.passengerId.value.toString,
      driverId = enRoute.driverId.value.toString,
      assignedAt = enRoute.assignedAt.toString
    )

  given Encoder[EnRouteWireDto] = (dto: EnRouteWireDto) =>
    Json.obj(
      "kind"         -> dto.kind.asJson,
      "request_id"   -> dto.requestId.asJson,
      "passenger_id" -> dto.passengerId.asJson,
      "driver_id"    -> dto.driverId.asJson,
      "assigned_at"  -> dto.assignedAt.asJson
    )

// Serialize
val json = EnRouteWireDto.fromDomain(enRoute).asJson.noSpaces
```

Scala maps camelCase local fields to snake_case wire fields explicitly in the
encoder. Alternatively, use `io.circe.derivation.Configuration` with
`derivedConfigured` and a snake_case naming strategy (`circe-generic-extras`
is deprecated).

## Receiver Side: Wire -> Inbound DTO -> Domain

The receiver parses wire data into an inbound DTO, validates it, then converts
to a domain model. The inbound DTO enforces the wire contract shape. The
conversion to domain types applies invariant checks and branded ID parsing.

### TypeScript

```typescript
// Inbound DTO schema (Zod, snake_case fields)
const EnRouteInboundDto = z.object({
  kind: z.literal("en_route"),
  request_id: z.string().uuid(),
  passenger_id: z.string().uuid(),
  driver_id: z.string().uuid(),
  assigned_at: z.string().datetime(),
});

type EnRouteInboundDto = z.infer<typeof EnRouteInboundDto>;

// Convert to domain model
const toDomain = (dto: EnRouteInboundDto): EnRoute => ({
  kind: "EnRoute",
  requestId: RequestId.parse(dto.request_id),
  passengerId: PassengerId.parse(dto.passenger_id),
  driverId: DriverId.parse(dto.driver_id),
  assignedAt: new Date(dto.assigned_at),
});

// Parse wire JSON
const dto = EnRouteInboundDto.parse(JSON.parse(wireJson));
const enRoute = toDomain(dto);
```

### Python

```python
class EnRouteInboundDto(BaseModel):
    """Inbound DTO for the EnRoute state."""

    model_config = ConfigDict(strict=True, extra="forbid")

    kind: Literal["en_route"]
    request_id: UUID
    passenger_id: UUID
    driver_id: UUID
    assigned_at: datetime

EnRouteInboundAdapter = TypeAdapter(EnRouteInboundDto)

def to_domain(dto: EnRouteInboundDto) -> EnRoute:
    return EnRoute(
        request_id=RequestId(dto.request_id),
        passenger_id=PassengerId(dto.passenger_id),
        driver_id=DriverId(dto.driver_id),
        assigned_at=dto.assigned_at,
    )

# Parse wire JSON
dto = EnRouteInboundAdapter.validate_json(wire_bytes)
en_route = to_domain(dto)
```

### Rust

```rust
/// Inbound wire DTO for the EnRoute state.
#[derive(serde::Deserialize)]
#[serde(deny_unknown_fields)]
pub struct EnRouteInboundDto {
    pub kind: String,
    pub request_id: String,
    pub passenger_id: String,
    pub driver_id: String,
    pub assigned_at: String,
}

impl TryFrom<EnRouteInboundDto> for EnRoute {
    type Error = DomainError;

    fn try_from(dto: EnRouteInboundDto) -> Result<Self, Self::Error> {
        Ok(EnRoute {
            request_id: RequestId::parse(&dto.request_id)?,
            passenger_id: PassengerId::parse(&dto.passenger_id)?,
            driver_id: DriverId::parse(&dto.driver_id)?,
            assigned_at: DateTime::parse_from_rfc3339(&dto.assigned_at)
                .map_err(|e| DomainError::InvalidTimestamp(e.to_string()))?,
        })
    }
}

// Parse wire JSON
let dto: EnRouteInboundDto = serde_json::from_str(wire_json)?;
let en_route = EnRoute::try_from(dto)?;
```

### Scala

```scala
/** Inbound wire DTO for the EnRoute state. */
final case class EnRouteInboundDto(
    kind: String,
    requestId: String,
    passengerId: String,
    driverId: String,
    assignedAt: String
)

object EnRouteInboundDto:
  given Decoder[EnRouteInboundDto] = (c: HCursor) =>
    for
      kind        <- c.downField("kind").as[String]
      requestId   <- c.downField("request_id").as[String]
      passengerId <- c.downField("passenger_id").as[String]
      driverId    <- c.downField("driver_id").as[String]
      assignedAt  <- c.downField("assigned_at").as[String]
    yield EnRouteInboundDto(kind, requestId, passengerId, driverId, assignedAt)

  def toDomain(dto: EnRouteInboundDto): Either[DomainError, EnRoute] =
    for
      reqId  <- RequestId.parse(dto.requestId)
      passId <- PassengerId.parse(dto.passengerId)
      drvId  <- DriverId.parse(dto.driverId)
      at     <- parseInstant(dto.assignedAt)
    yield EnRoute(reqId, passId, drvId, at)

// Parse wire JSON
val result = decode[EnRouteInboundDto](wireJson).flatMap(EnRouteInboundDto.toDomain)
```

## Naming Convention Translation

The wire format uses snake_case for all field names. Each language converts
at its boundary:

| Direction | TypeScript | Python | Rust | Scala |
| --- | --- | --- | --- | --- |
| Wire -> Local | snake_case -> camelCase | no change | no change | snake_case -> camelCase |
| Local -> Wire | camelCase -> snake_case | no change | no change | camelCase -> snake_case |

TypeScript and Scala require explicit mapping in their DTO conversion functions
or codec definitions. Python and Rust match the wire format naturally.

## Separate vs Shared DTOs

**Prefer separate inbound and outbound DTOs** when:

- The outbound DTO includes computed or derived fields the receiver ignores.
- The sender and receiver evolve independently (different teams, repos, or
  deployment cadences).
- The inbound side needs stricter validation than the outbound side enforces.

**Shared DTOs are acceptable** when:

- Both sides are tightly coupled (same team, same repo, same deployment).
- The event payload is simple and stable.
- The DTO type is identical for send and receive.

When in doubt, start with separate DTOs. Merging them later is easy; splitting
a shared DTO that has accumulated asymmetric concerns is harder.

## Discriminated Union DTOs

When the wire payload is a discriminated union (multiple possible states),
define a DTO for each variant and a union type or parser that dispatches on
the `kind` field.

**TypeScript:**
```typescript
const TaxiRequestWireDto = z.discriminatedUnion("kind", [
  WaitingWireDto,
  EnRouteWireDto,
  InTripWireDto,
  CompletedWireDto,
  CancelledWireDto,
]);
```

**Python:**
```python
TaxiRequestWireDto = Annotated[
    WaitingWireDto | EnRouteWireDto | InTripWireDto | CompletedWireDto | CancelledWireDto,
    Discriminator("kind"),
]
```

**Rust:**
```rust
// Note: do NOT use `deny_unknown_fields` on an internally-tagged enum.
// serde treats the tag field itself as "unknown" when combined with
// `deny_unknown_fields` on an internally-tagged enum, causing
// deserialization to always fail. Apply `deny_unknown_fields` on
// the individual variant structs (e.g., WaitingWireDto) instead.
#[derive(serde::Deserialize)]
#[serde(tag = "kind")]
pub enum TaxiRequestWireDto {
    #[serde(rename = "waiting")]
    Waiting(WaitingWireDto),
    #[serde(rename = "en_route")]
    EnRoute(EnRouteWireDto),
    #[serde(rename = "in_trip")]
    InTrip(InTripWireDto),
    #[serde(rename = "completed")]
    Completed(CompletedWireDto),
    #[serde(rename = "cancelled")]
    Cancelled(CancelledWireDto),
}
```

**Scala:**
```scala
def decodeTaxiRequestWireDto(json: String): Either[Error, TaxiRequestWireDto] =
  parse(json).flatMap { j =>
    j.hcursor.downField("kind").as[String].flatMap {
      case "waiting"   => j.as[WaitingWireDto]
      case "en_route"  => j.as[EnRouteWireDto]
      case "in_trip"   => j.as[InTripWireDto]
      case "completed" => j.as[CompletedWireDto]
      case "cancelled" => j.as[CancelledWireDto]
      case other       => Left(DecodingFailure(s"Unknown kind: $other", Nil))
    }
  }
```

See [`wire-format-conventions.md`](./wire-format-conventions.md) for the
canonical discriminant and field naming rules.
