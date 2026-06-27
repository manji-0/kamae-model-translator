# Protobuf and gRPC Mapping

> **When to read:** Using Protocol Buffers or gRPC to exchange kamae domain
> models between services. This reference covers mapping kamae constructs to
> Protobuf message definitions and per-language gRPC integration patterns.

## Core Principle

Protobuf `.proto` files are the shared contract. Each language generates
structs/classes from the proto definition, then converts between generated
types and local domain types at the boundary -- the same DTO boundary pattern
used for JSON, applied to Protobuf-generated code.

## Mapping Kamae Constructs to Protobuf

### Discriminated Union -> `oneof`

A kamae discriminated union maps to a Protobuf `oneof`:

```protobuf
message TaxiRequestState {
  oneof state {
    WaitingState waiting = 1;
    EnRouteState en_route = 2;
    InTripState in_trip = 3;
    CompletedState completed = 4;
    CancelledState cancelled = 5;
  }
}
```

Each variant is a separate message type, and the `oneof` wrapper provides
the discriminant. The field names use snake_case, matching the wire-format
convention for `kind` values.

### State Type -> `message`

Each state variant becomes its own Protobuf message:

```protobuf
message WaitingState {
  string request_id = 1;
  string passenger_id = 2;
  google.protobuf.Timestamp created_at = 3;
}

message EnRouteState {
  string request_id = 1;
  string passenger_id = 2;
  string driver_id = 3;
  google.protobuf.Timestamp assigned_at = 4;
}
```

Field names use snake_case in the proto definition, consistent with the
wire-format conventions.

### Branded ID -> `string` with Code-Level Validation

Protobuf has no concept of branded or opaque types. Branded IDs are
represented as plain `string` fields in proto definitions. The branding
and validation happen in the DTO-to-domain conversion layer of each language:

```protobuf
message EnRouteState {
  string request_id = 1;   // Validated as RequestId in code
  string passenger_id = 2; // Validated as PassengerId in code
  string driver_id = 3;    // Validated as DriverId in code
}
```

### Domain Events -> `message` with `oneof` Payload

Domain events use an envelope message with a `oneof` for the event-specific
payload:

```protobuf
message DomainEvent {
  string event_id = 1;
  google.protobuf.Timestamp event_at = 2;
  string aggregate_id = 3;
  string aggregate_name = 4;
  oneof payload {
    DriverAssignedPayload driver_assigned = 10;
    TripStartedPayload trip_started = 11;
    TripCompletedPayload trip_completed = 12;
    TripCancelledPayload trip_cancelled = 13;
  }
}

message DriverAssignedPayload {
  string driver_id = 1;
  string passenger_id = 2;
}

message TripCompletedPayload {
  string driver_id = 1;
  int64 fare_amount_cents = 2;
  string fare_currency = 3;
}
```

Reserve field numbers 10+ for payload variants so envelope fields can grow
without collision.

### Error Types -> gRPC Status Codes + Details

Map domain error types to gRPC status codes:

| Domain error | gRPC status code | When to use |
| --- | --- | --- |
| `RequestNotFound` | `NOT_FOUND` | Aggregate or entity does not exist |
| `InvalidState` | `FAILED_PRECONDITION` | State transition is not valid from current state |
| `DriverNotAvailable` | `FAILED_PRECONDITION` | Business precondition not met (driver cannot be assigned). `UNAVAILABLE` is reserved for transient infrastructure failures and signals clients to retry, which is wrong for a business logic decision. |
| `ValidationError` | `INVALID_ARGUMENT` | Request fields fail validation |
| `Unauthorized` | `PERMISSION_DENIED` | Caller lacks required permissions |
| `Conflict` | `ALREADY_EXISTS` | Idempotency key collision or duplicate create |

For structured error details, use gRPC's `google.rpc.Status` with
`google.rpc.ErrorInfo` or a custom error message in the response `oneof`:

```protobuf
message AssignDriverResponse {
  oneof result {
    EnRouteState success = 1;
    AssignDriverError error = 2;
  }
}

message AssignDriverError {
  string code = 1;    // "request_not_found", "invalid_state", "driver_not_available"
  string message = 2;
}
```

Prefer the `oneof` response pattern when clients need structured error details
beyond what gRPC status metadata provides.

## Per-Language gRPC Integration

### TypeScript

Use `@grpc/grpc-js` or `nice-grpc` with `ts-proto` or `protobuf-ts` for
type generation.

```typescript
// Generated type (from ts-proto)
interface AssignDriverRequest {
  requestId: string;  // ts-proto generates camelCase by default
  driverId: string;
}

// Convert domain command to proto request
const toProtoRequest = (cmd: AssignDriverCommand): AssignDriverRequest => ({
  requestId: cmd.requestId,
  driverId: cmd.driverId,
});

// Convert proto response to domain result
const fromProtoResponse = (
  res: AssignDriverResponse
): Result<EnRoute, AssignDriverError> => {
  if (res.result?.$case === "success") {
    return ok(toEnRoute(res.result.success));
  }
  if (res.result?.$case === "error") {
    return err(toDomainError(res.result.error));
  }
  return err({ kind: "UnknownResponse" });
};
```

Note: `ts-proto` generates camelCase field names from snake_case proto
definitions by default. The proto-generated types serve as the boundary DTOs.

### Python

Use `grpcio` with `betterproto` or `grpclib` for async support.

```python
# Generated type (from betterproto)
@dataclass
class AssignDriverRequest(betterproto.Message):
    request_id: str = betterproto.string_field(1)
    driver_id: str = betterproto.string_field(2)

# Convert domain command to proto request
def to_proto_request(cmd: AssignDriverCommand) -> AssignDriverRequest:
    return AssignDriverRequest(
        request_id=str(cmd.request_id),
        driver_id=str(cmd.driver_id),
    )

# Convert proto response to domain result
def from_proto_response(
    res: AssignDriverResponse,
) -> EnRoute:
    match res:
        case AssignDriverResponse(success=state) if state:
            return to_en_route(state)
        case AssignDriverResponse(error=err) if err:
            raise to_domain_error(err)
        case _:
            raise DomainError("unknown_response", "Unrecognized response shape")
```

### Rust

Use `tonic` with `prost` for code generation.

```rust
// Generated type (from prost)
// Field names are snake_case, matching Rust convention.
pub struct AssignDriverRequest {
    pub request_id: String,
    pub driver_id: String,
}

// Convert domain command to proto request
impl From<AssignDriverCommand> for proto::AssignDriverRequest {
    fn from(cmd: AssignDriverCommand) -> Self {
        Self {
            request_id: cmd.request_id.to_string(),
            driver_id: cmd.driver_id.to_string(),
        }
    }
}

// Convert proto response to domain result
impl TryFrom<proto::AssignDriverResponse> for DomainResult<EnRoute> {
    type Error = DomainError;

    fn try_from(res: proto::AssignDriverResponse) -> Result<Self, Self::Error> {
        match res.result {
            Some(proto::assign_driver_response::Result::Success(state)) => {
                Ok(DomainResult::Ok(EnRoute::try_from(state)?))
            }
            Some(proto::assign_driver_response::Result::Error(err)) => {
                Err(DomainError::from_proto(err))
            }
            None => Err(DomainError::UnknownResponse),
        }
    }
}
```

### Scala

Use `fs2-grpc` or `zio-grpc` with ScalaPB for code generation.

```scala
// Generated type (from ScalaPB)
// Field names are camelCase, matching Scala convention.
final case class AssignDriverRequest(
    requestId: String,
    driverId: String
)

// Convert domain command to proto request
def toProtoRequest(cmd: AssignDriverCommand): AssignDriverRequest =
  AssignDriverRequest(
    requestId = cmd.requestId.value.toString,
    driverId = cmd.driverId.value.toString
  )

// Convert proto response to domain result
def fromProtoResponse(
    res: AssignDriverResponse
): Either[DomainError, EnRoute] =
  res.result match
    case AssignDriverResponse.Result.Success(state) =>
      toEnRoute(state)
    case AssignDriverResponse.Result.Error(err) =>
      Left(toDomainError(err))
    case AssignDriverResponse.Result.Empty =>
      Left(DomainError.UnknownResponse)
```

## Protobuf-Specific Concerns

### Default Values and Absence

In proto3, default values (0 for integers, `""` for strings, `false` for
booleans) are indistinguishable from absent fields. When absence has business
meaning:

- Use `optional` keyword (proto3 syntax):
  ```protobuf
  optional string phone_number = 5;
  ```
- Or use wrapper types:
  ```protobuf
  import "google/protobuf/wrappers.proto";
  google.protobuf.StringValue phone_number = 5;
  ```

Prefer `optional` for new proto3 definitions. Use wrapper types when
interoperating with existing services that already use them.

### Timestamps

Always use `google.protobuf.Timestamp` for time values, never strings:

```protobuf
import "google/protobuf/timestamp.proto";

message EnRouteState {
  string request_id = 1;
  google.protobuf.Timestamp assigned_at = 4;
}
```

Each language's protobuf library handles conversion to native time types:

| Language | Protobuf Timestamp -> Native |
| --- | --- |
| TypeScript | `Timestamp` -> `Date` via `timestamp.toDate()` |
| Python | `Timestamp` -> `datetime` via `timestamp.ToDatetime()` |
| Rust | `prost_types::Timestamp` -> `chrono::DateTime<Utc>` (manual conversion) |
| Scala | `Timestamp` -> `java.time.Instant` via ScalaPB conversions |

### Enums

Protobuf enums are integers. Always include an `UNSPECIFIED = 0` default:

```protobuf
enum VehicleType {
  VEHICLE_TYPE_UNSPECIFIED = 0;
  VEHICLE_TYPE_SEDAN = 1;
  VEHICLE_TYPE_SUV = 2;
  VEHICLE_TYPE_VAN = 3;
}
```

Map Protobuf enum values to kamae domain enum types in the conversion layer.
Treat `UNSPECIFIED` as an error or a sensible default, depending on the domain
semantics -- never silently propagate it as a valid domain value.

## Schema Evolution in Protobuf

### Safe Changes

- Add new fields with new field numbers.
- Add new variants to a `oneof`.
- Add new enum values (receivers must handle unknown values gracefully).
- Add new RPC methods to a service.

### Unsafe Changes

- Remove or reuse field numbers (causes data corruption with old clients).
- Change a field's type or number.
- Rename fields (binary format uses numbers, but JSON mapping uses names).
- Move fields in or out of a `oneof`.

### Reserving Removed Fields

When removing a field, reserve its number and name to prevent accidental reuse:

```protobuf
message WaitingState {
  reserved 4, 5;
  reserved "old_field", "deprecated_field";

  string request_id = 1;
  string passenger_id = 2;
  google.protobuf.Timestamp created_at = 3;
}
```

### Versioning Strategy

For major contract changes, prefer creating a new message type or service
version (e.g., `v2.TaxiRequestService`) over modifying existing messages in
breaking ways. This aligns with the schema evolution strategies described in
[`schema-evolution.md`](./schema-evolution.md).
