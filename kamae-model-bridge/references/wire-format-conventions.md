# Wire Format Conventions

> **When to read:** Every bridge task. This defines the canonical rules for
> exchanging kamae domain data between services in different languages.

## Core Principle

The wire format is a shared contract owned by neither sender nor receiver.
Both sides translate between the wire representation and their local domain
types through boundary DTOs.

## Discriminant Field

All kamae languages use `kind` as the discriminant for domain state unions.
On the wire, this convention is preserved:

```json
{
  "kind": "waiting",
  "request_id": "550e8400-e29b-41d4-a716-446655440000",
  "passenger_id": "6ba7b810-9dad-11d1-80b4-00c04fd430c8"
}
```

### Value Convention

Use **snake_case** for `kind` values on the wire:

| Wire value | TS local | Python local | Rust local | Scala local |
| --- | --- | --- | --- | --- |
| `"waiting"` | `"Waiting"` (PascalCase) | `"waiting"` (same) | `Waiting` variant | `Waiting` case |
| `"en_route"` | `"EnRoute"` | `"en_route"` (same) | `EnRoute` variant | `EnRoute` case |
| `"in_trip"` | `"InTrip"` | `"in_trip"` (same) | `InTrip` variant | `InTrip` case |

Each language maps between wire values and local discriminants in its
serialization/deserialization layer. See
[`discriminant-interop.md`](./discriminant-interop.md) for implementation
details.

## Field Naming

Use **snake_case** for all field names on the wire:

```json
{
  "kind": "en_route",
  "request_id": "...",
  "passenger_id": "...",
  "driver_id": "...",
  "assigned_at": "2024-03-15T10:30:00Z"
}
```

Each language converts to its local convention at the DTO boundary:

| Wire (snake_case) | TypeScript (camelCase) | Python (snake_case) | Rust (snake_case) | Scala (camelCase) |
| --- | --- | --- | --- | --- |
| `request_id` | `requestId` | `request_id` | `request_id` | `requestId` |
| `passenger_id` | `passengerId` | `passenger_id` | `passenger_id` | `passengerId` |
| `assigned_at` | `assignedAt` | `assigned_at` | `assigned_at` | `assignedAt` |

### Language-Specific Configuration

**TypeScript (Zod):**
```typescript
const EnRouteWireDto = z.object({
  kind: z.literal("en_route"),
  request_id: z.string().uuid(),
  driver_id: z.string().uuid(),
  assigned_at: z.string().datetime(),
});

const toEnRoute = (dto: z.infer<typeof EnRouteWireDto>): EnRoute => ({
  kind: "EnRoute",
  requestId: RequestId.parse(dto.request_id),
  driverId: DriverId.parse(dto.driver_id),
  assignedAt: new Date(dto.assigned_at),
});
```

**Python (Pydantic):**
```python
class EnRouteWireDto(BaseModel):
    model_config = ConfigDict(strict=True, extra="forbid")

    kind: Literal["en_route"]
    request_id: UUID
    passenger_id: UUID
    driver_id: UUID
    assigned_at: datetime
```

Python snake_case matches the wire format, so no field alias is needed.

**Rust (serde):**
```rust
#[derive(serde::Deserialize)]
#[serde(deny_unknown_fields)]
pub struct EnRouteWireDto {
    kind: String,
    request_id: String,
    passenger_id: String,
    driver_id: String,
    assigned_at: String,
}
```

Rust snake_case matches the wire format. Use `#[serde(rename = "...")]`
only when fields diverge.

**Scala (Circe):**
```scala
final case class EnRouteWireDto(
    kind: String,
    requestId: String,
    passengerId: String,
    driverId: String,
    assignedAt: String
)

object EnRouteWireDto:
  given Decoder[EnRouteWireDto] = (c: HCursor) =>
    for
      kind        <- c.downField("kind").as[String]
      requestId   <- c.downField("request_id").as[String]
      passengerId <- c.downField("passenger_id").as[String]
      driverId    <- c.downField("driver_id").as[String]
      assignedAt  <- c.downField("assigned_at").as[String]
    yield EnRouteWireDto(kind, requestId, passengerId, driverId, assignedAt)
```

Scala maps snake_case wire fields to camelCase local fields in the codec.
Alternatively, use `io.circe.derivation.Configuration` with `derivedConfigured`
and a snake_case naming strategy (`circe-generic-extras` is deprecated).

## Scalar Type Conventions

| Concept | Wire representation | Notes |
| --- | --- | --- |
| **UUID** | Lowercase hyphenated string `"550e8400-e29b-41d4-a716-446655440000"` | 36 characters, RFC 4122 |
| **Timestamp** | ISO 8601 with timezone `"2024-03-15T10:30:00Z"` | Always UTC on the wire; local TZ conversion at the receiver |
| **Date** | ISO 8601 date `"2024-03-15"` | No time component |
| **Money** | Object `{ "amount_cents": 1500, "currency": "USD" }` | Integer cents (or smallest unit); never floating point |
| **Boolean** | JSON `true` / `false` | Never string `"true"` |
| **Enum string** | snake_case `"en_route"` | Matches discriminant convention |
| **Nullable** | JSON `null` or field absent | Document which convention per field; see below |

### Null vs Absent

Two conventions exist; pick one per service boundary and document it:

| Convention | JSON shape | When to use |
| --- | --- | --- |
| **Explicit null** | `{ "phone": null }` | Field is known to have no value |
| **Absent field** | `{ }` (no `phone` key) | Field was not provided or is irrelevant in this state |

Receivers must handle both if the sender's convention is not enforced.
Prefer explicit null for fields that have business meaning when empty.

## Event Envelope

Domain events on the wire follow a standard envelope:

```json
{
  "event_name": "driver_assigned",
  "event_id": "...",
  "event_at": "2024-03-15T10:30:00Z",
  "aggregate_id": "...",
  "aggregate_name": "taxi_request",
  "payload": {
    "driver_id": "...",
    "passenger_id": "..."
  }
}
```

- `event_name`: snake_case, matches the event type discriminant
- `event_at`: ISO 8601 UTC
- `aggregate_name`: snake_case
- `payload`: contains event-specific data, field names in snake_case

## Versioning Header

When schema versions matter, include a version field:

```json
{
  "event_name": "driver_assigned",
  "event_version": 1,
  "payload": { ... }
}
```

See [`schema-evolution.md`](./schema-evolution.md) for versioning strategies.

## PII on the Wire

Never include raw PII in event payloads unless the consuming service
genuinely needs it and both sides document the data classification.
Prefer opaque IDs that the consumer can use to look up contact details
from an identity service when needed.

See each language's kamae PII protection reference for local redaction patterns.
