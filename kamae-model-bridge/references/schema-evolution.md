# Schema Evolution

When to read: Adding, removing, or changing fields in wire-format models that
cross service boundaries between kamae services in different languages.

---

## Backward compatibility rules

### Safe changes

These changes are backward-compatible -- an older receiver can still process
messages from an updated sender (possibly ignoring the new content):

| Change                                | Why it is safe                            |
| ------------------------------------- | ----------------------------------------- |
| Add an optional field with default    | Older receivers ignore it or use default  |
| Add a new `kind` discriminant value   | Older receivers must have unknown-kind handling (see below) |
| Widen a numeric field (int32 to int64)| Older values still fit the wider type     |
| Add a new event type                  | Older consumers ignore unknown event names|

### Breaking changes

These changes require **coordinated deployment** -- all consumers must update
before or simultaneously with the producer:

| Change                                | Why it breaks                             |
| ------------------------------------- | ----------------------------------------- |
| Remove a required field               | Older receivers fail to deserialize       |
| Rename a field                        | Older receivers look for the old name     |
| Change a field type incompatibly      | `string` to `int` breaks all parsers      |
| Remove a `kind` discriminant value    | Consumers matching on that value break    |
| Change a `kind` value's string        | `"en_route"` to `"enroute"` breaks matches|

### Unknown-kind handling

When a new `kind` value is added to a discriminated union, older receivers
will encounter a value they do not recognize. Each language should handle this
explicitly at the DTO boundary:

**TypeScript:**

```typescript
function handleState(wire: unknown): Result<DomainState, DomainError> {
  const parsed = WireStateDto.safeParse(wire);
  if (!parsed.success) {
    // Unknown kind falls here if using z.discriminatedUnion
    return err({ kind: "UnknownWireState", raw: wire });
  }
  return ok(toDomain(parsed.data));
}
```

**Python:**

```python
from pydantic import BaseModel, ConfigDict

class WireStateDto(BaseModel):
    model_config = ConfigDict(extra="allow")
    kind: str  # Accept any string, dispatch manually

def from_wire(dto: WireStateDto) -> DomainState:
    match dto.kind:
        case "waiting":
            return parse_waiting(dto)
        case "en_route":
            return parse_en_route(dto)
        case _:
            raise UnknownKindError(dto.kind)
```

**Rust:**

```rust
#[derive(Debug, Deserialize)]
#[serde(tag = "kind")]
pub enum WireStateDto {
    #[serde(rename = "waiting")]
    Waiting { passenger_id: String },
    #[serde(rename = "en_route")]
    EnRoute { passenger_id: String, driver_id: String },
    #[serde(other)]
    Unknown,
}
```

**Scala:**

```scala
implicit val decoder: Decoder[WireStateDto] =
  Decoder.instance { cursor =>
    cursor.get[String]("kind").flatMap {
      case "waiting"  => cursor.as[Waiting]
      case "en_route" => cursor.as[EnRoute]
      case other      => Right(Unknown(other))
    }
  }
```

---

## Versioning strategies

### Event versioning

Include an `event_version` field in the event envelope. Consumers dispatch on
the version to choose the correct deserialization path.

```json
{
  "event_name": "driver_assigned",
  "event_version": 2,
  "event_id": "evt-abc-123",
  "event_at": "2024-03-15T10:30:00Z",
  "aggregate_id": "req-001",
  "aggregate_name": "taxi_request",
  "payload": {
    "kind": "en_route",
    "driver_id": "drv-001",
    "passenger_id": "psg-001",
    "vehicle_type": "sedan"
  }
}
```

Version 2 adds `vehicle_type` to the payload. Consumers that only know v1
ignore `vehicle_type` (if using tolerant parsing) or dispatch to a v1 handler.

### API versioning

For synchronous APIs (REST, gRPC), version the contract explicitly:

| Strategy            | Example                                       | Trade-off                |
| ------------------- | --------------------------------------------- | ------------------------ |
| URL path            | `/v2/requests`                                | Simple; clutters routes  |
| Accept header       | `Accept: application/vnd.taxi.v2+json`        | Clean URLs; harder debug |
| Content negotiation | Server checks `Accept`, returns best match    | Flexible; complex        |
| Protobuf package    | `package taxi.v2;`                            | Standard for gRPC        |

### Schema registry

For message queues (Kafka, SQS, EventBridge), use a schema registry that
enforces compatibility rules before a new schema version is accepted:

- **Confluent Schema Registry**: Supports Avro, Protobuf, JSON Schema.
  Enforces backward, forward, or full compatibility.
- **AWS Glue Schema Registry**: Supports Avro and JSON Schema. Integrates
  with MSK and Kinesis.

The registry rejects schema changes that violate the configured compatibility
mode, preventing breaking changes from reaching production.

---

## Migration patterns

### Dual-write

The sender produces **both** v1 and v2 events during the migration window.
Remove v1 after all consumers have upgraded to v2.

```
Timeline:
  1. Sender starts producing v1 + v2     [both versions live]
  2. Consumer A upgrades to read v2       [A reads v2, B reads v1]
  3. Consumer B upgrades to read v2       [all read v2]
  4. Sender stops producing v1            [v1 retired]
```

**When to use:** Multiple consumers upgrading independently. The sender can
afford the overhead of producing two event shapes.

**Risk:** The sender must keep v1 and v2 serialization logic in sync during
the window. Test both paths.

### Consumer-side transform

The consumer accepts v1 on the wire and transforms it to v2 internally. The
sender upgrades to v2 on its own schedule.

```python
def from_wire(raw: dict) -> DriverAssignedV2:
    version = raw.get("event_version", 1)
    if version == 1:
        # v1 has no vehicle_type; default to "standard"
        return DriverAssignedV2(
            driver_id=raw["payload"]["driver_id"],
            passenger_id=raw["payload"]["passenger_id"],
            vehicle_type="standard",
        )
    elif version == 2:
        return DriverAssignedV2(**raw["payload"])
    else:
        raise UnknownVersionError(version)
```

**When to use:** Consumer needs the new shape immediately but the sender is
maintained by a different team with a different release cadence.

**Risk:** Each consumer carries transform logic. If there are many consumers,
the dual-write pattern may be cleaner.

### Expand-and-contract

A three-phase approach for field renames or type changes:

1. **Expand:** Add the new field alongside the old field. Sender writes both.
   ```json
   { "fare": 19.99, "fare_cents": 1999 }
   ```
2. **Migrate:** All consumers switch to reading the new field.
3. **Contract:** Remove the old field. Sender stops writing it.
   ```json
   { "fare_cents": 1999 }
   ```

**When to use:** Renaming a field or changing its type (e.g., float dollars to
integer cents). Each phase is independently deployable.

**Risk:** Phase 2 can take time if consumers are maintained by different teams.
Set a deadline and track progress.

---

## Per-language handling

### TypeScript

```typescript
import { z } from "zod";

// Accept both v1 and v2 with a union
const DriverAssignedV1 = z.object({
  event_version: z.literal(1),
  payload: z.object({
    kind: z.literal("en_route"),
    driver_id: z.string(),
    passenger_id: z.string(),
  }),
});

const DriverAssignedV2 = z.object({
  event_version: z.literal(2),
  payload: z.object({
    kind: z.literal("en_route"),
    driver_id: z.string(),
    passenger_id: z.string(),
    vehicle_type: z.string(),
  }),
});

const DriverAssigned = z.discriminatedUnion("event_version", [
  DriverAssignedV1,
  DriverAssignedV2,
]);

// Normalize to v2 domain model
function toDomain(wire: z.infer<typeof DriverAssigned>): DriverAssignedDomain {
  if (wire.event_version === 1) {
    return { ...wire.payload, vehicleType: "standard" };
  }
  return { ...wire.payload, vehicleType: wire.payload.vehicle_type };
}
```

### Python

```python
from pydantic import BaseModel, ConfigDict

class DriverAssignedWireDto(BaseModel):
    """Tolerant inbound DTO: accepts v1 and v2."""
    model_config = ConfigDict(extra="allow")

    event_version: int
    payload: dict  # raw payload; dispatch on version

def from_wire(dto: DriverAssignedWireDto) -> DriverAssignedDomain:
    match dto.event_version:
        case 1:
            return DriverAssignedDomain(
                driver_id=dto.payload["driver_id"],
                passenger_id=dto.payload["passenger_id"],
                vehicle_type="standard",
            )
        case 2:
            return DriverAssignedDomain(**dto.payload)
        case _:
            raise UnknownVersionError(dto.event_version)
```

### Rust

```rust
use serde::Deserialize;

#[derive(Debug, Deserialize)]
pub struct EventEnvelope {
    pub event_version: u32,
    pub payload: serde_json::Value, // raw; dispatch on version
}

#[derive(Debug, Deserialize)]
pub struct DriverAssignedV1Payload {
    pub driver_id: String,
    pub passenger_id: String,
}

#[derive(Debug, Deserialize)]
pub struct DriverAssignedV2Payload {
    pub driver_id: String,
    pub passenger_id: String,
    pub vehicle_type: String,
}

fn from_wire(envelope: &EventEnvelope) -> Result<DriverAssignedDomain, Error> {
    match envelope.event_version {
        1 => {
            let v1: DriverAssignedV1Payload =
                serde_json::from_value(envelope.payload.clone())?;
            Ok(DriverAssignedDomain {
                driver_id: v1.driver_id,
                passenger_id: v1.passenger_id,
                vehicle_type: "standard".into(),
            })
        }
        2 => {
            let v2: DriverAssignedV2Payload =
                serde_json::from_value(envelope.payload.clone())?;
            Ok(DriverAssignedDomain {
                driver_id: v2.driver_id,
                passenger_id: v2.passenger_id,
                vehicle_type: v2.vehicle_type,
            })
        }
        v => Err(Error::UnknownVersion(v)),
    }
}
```

### Scala

```scala
import io.circe.{Decoder, Json}
import io.circe.generic.semiauto._

case class EventEnvelope(eventVersion: Int, payload: Json)

object EventEnvelope {
  implicit val decoder: Decoder[EventEnvelope] = deriveDecoder
}

def fromWire(envelope: EventEnvelope): Either[Error, DriverAssignedDomain] =
  envelope.eventVersion match {
    case 1 =>
      envelope.payload.as[DriverAssignedV1Payload].map { v1 =>
        DriverAssignedDomain(
          driverId = v1.driverId,
          passengerId = v1.passengerId,
          vehicleType = "standard"
        )
      }.left.map(e => DecodingError(e.message))
    case 2 =>
      envelope.payload.as[DriverAssignedV2Payload].map { v2 =>
        DriverAssignedDomain(
          driverId = v2.driverId,
          passengerId = v2.passengerId,
          vehicleType = v2.vehicleType
        )
      }.left.map(e => DecodingError(e.message))
    case v =>
      Left(UnknownVersionError(v))
  }
```

---

## Tombstone events and soft deletes

When removing a state or event type from the system:

1. **Deprecation window:** Announce the deprecation. Set a date after which
   the old type will no longer be produced.
2. **Tombstone event:** Emit a final event that signals "this type is retired."
   Consumers that encounter it can clean up local state.
   ```json
   {
     "event_name": "driver_assigned_deprecated",
     "event_version": 1,
     "payload": {
       "reason": "Replaced by driver_matched in v3 API",
       "successor_event": "driver_matched",
       "deprecation_date": "2024-06-01"
     }
   }
   ```
3. **Consumer cleanup:** Consumers remove handler logic for the old type after
   the deprecation window closes and all in-flight messages have been
   processed.
4. **Producer cleanup:** Producer removes serialization logic for the old type.

### Soft delete for `kind` values

When a `kind` value is removed from a discriminated union:

- Do not remove it from the wire DTO immediately. Keep it in the receiver's
  unknown-kind handler for the deprecation window.
- After the window, the unknown-kind handler can reject it or log a warning,
  depending on the system's tolerance for stale messages in queues.

---

## Decision checklist

Before making a wire format change:

- [ ] Is this change backward-compatible? (Check the safe/breaking table above)
- [ ] If breaking: which migration pattern will you use? (dual-write,
      consumer-side transform, expand-and-contract)
- [ ] Have you bumped `event_version` or API version?
- [ ] Have you updated the shared fixture files? (see `contract-testing.md`)
- [ ] Have you updated all receiver DTOs in all languages?
- [ ] If adding a new `kind` value: do all receivers have unknown-kind
      handling?
- [ ] If removing a type or field: have you set a deprecation window and
      communicated it?
- [ ] If using a schema registry: have you verified the new schema passes
      compatibility checks?
