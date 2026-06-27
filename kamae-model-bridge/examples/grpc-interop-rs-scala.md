# Worked Example: Rust to Scala gRPC Command Exchange

> **When to read:** You need a concrete end-to-end example of a Rust gateway
> service sending commands to a Scala backend service via gRPC, with Protobuf
> as the wire format.

## Scenario

A Rust API gateway receives HTTP requests and sends `AssignDriver` commands
to a Scala domain service via gRPC. The Scala service executes the business
logic and returns either a success state or a structured error.

## Shared Proto Definition

Both services generate code from the same `.proto` file:

```protobuf
syntax = "proto3";
package taxi.v1;

import "google/protobuf/timestamp.proto";

service TaxiRequestService {
  rpc AssignDriver (AssignDriverRequest) returns (AssignDriverResponse);
}

message AssignDriverRequest {
  string request_id = 1;
  string driver_id = 2;
}

message AssignDriverResponse {
  oneof result {
    EnRouteState success = 1;
    AssignDriverError error = 2;
  }
}

message EnRouteState {
  string request_id = 1;
  string passenger_id = 2;
  string driver_id = 3;
  google.protobuf.Timestamp assigned_at = 4;
}

message AssignDriverError {
  string code = 1;
  string message = 2;
}
```

The `oneof result` pattern gives the client structured access to both success
and error outcomes without relying solely on gRPC status codes.

## Sender: Rust (tonic)

### Domain Command

```rust
use crate::domain::{DriverId, RequestId};

/// Command to assign a driver to a taxi request.
pub struct AssignDriverCommand {
    pub request_id: RequestId,
    pub driver_id: DriverId,
}
```

### Convert Domain Command to Proto Request

```rust
use crate::proto::taxi::v1 as proto;

impl From<AssignDriverCommand> for proto::AssignDriverRequest {
    fn from(cmd: AssignDriverCommand) -> Self {
        Self {
            request_id: cmd.request_id.to_string(),
            driver_id: cmd.driver_id.to_string(),
        }
    }
}
```

### Send via tonic Client

```rust
use tonic::transport::Channel;
use crate::proto::taxi::v1::taxi_request_service_client::TaxiRequestServiceClient;

pub struct TaxiGateway {
    client: TaxiRequestServiceClient<Channel>,
}

impl TaxiGateway {
    pub async fn assign_driver(
        &mut self,
        cmd: AssignDriverCommand,
    ) -> Result<EnRoute, GatewayError> {
        let request = tonic::Request::new(proto::AssignDriverRequest::from(cmd));

        let response = self
            .client
            .assign_driver(request)
            .await
            .map_err(GatewayError::from_tonic_status)?;

        let inner = response.into_inner();
        EnRoute::try_from_proto(inner)
    }
}
```

### Parse Proto Response to Domain Result

```rust
use crate::domain::{EnRoute, DriverId, PassengerId, RequestId};
use chrono::{DateTime, Utc};

impl EnRoute {
    fn try_from_proto(res: proto::AssignDriverResponse) -> Result<Self, GatewayError> {
        match res.result {
            Some(proto::assign_driver_response::Result::Success(state)) => {
                let assigned_at = state
                    .assigned_at
                    .ok_or(GatewayError::MissingField("assigned_at"))?;

                Ok(EnRoute {
                    request_id: RequestId::parse(&state.request_id)
                        .map_err(GatewayError::InvalidId)?,
                    passenger_id: PassengerId::parse(&state.passenger_id)
                        .map_err(GatewayError::InvalidId)?,
                    driver_id: DriverId::parse(&state.driver_id)
                        .map_err(GatewayError::InvalidId)?,
                    assigned_at: to_chrono_datetime(assigned_at)?,
                })
            }
            Some(proto::assign_driver_response::Result::Error(err)) => {
                Err(GatewayError::DomainError {
                    code: err.code,
                    message: err.message,
                })
            }
            None => Err(GatewayError::EmptyResponse),
        }
    }
}

fn to_chrono_datetime(
    ts: prost_types::Timestamp,
) -> Result<DateTime<Utc>, GatewayError> {
    DateTime::from_timestamp(ts.seconds, ts.nanos as u32)
        .ok_or(GatewayError::InvalidTimestamp)
}
```

### Rust Error Type

```rust
#[derive(Debug)]
pub enum GatewayError {
    /// gRPC transport or protocol error.
    Transport(String),
    /// Domain-level error returned by the server.
    DomainError { code: String, message: String },
    /// Required field was absent in the proto response.
    MissingField(&'static str),
    /// ID string failed branded-type validation.
    InvalidId(IdParseError),
    /// Timestamp could not be converted.
    InvalidTimestamp,
    /// Response oneof was empty.
    EmptyResponse,
}

impl GatewayError {
    fn from_tonic_status(status: tonic::Status) -> Self {
        Self::Transport(format!("{}: {}", status.code(), status.message()))
    }
}
```

## Receiver: Scala (fs2-grpc)

### Receive Proto Request and Convert to Domain Command

```scala
import taxi.v1.{AssignDriverRequest as ProtoRequest, *}
import domain.*

def toDomainCommand(
    req: ProtoRequest
): Either[DomainError, AssignDriverCommand] =
  for
    requestId <- RequestId.parse(req.requestId)
    driverId  <- DriverId.parse(req.driverId)
  yield AssignDriverCommand(requestId, driverId)
```

### Execute Use Case

```scala
import cats.effect.IO

trait AssignDriverUseCase:
  def execute(cmd: AssignDriverCommand): IO[Either[DomainError, EnRoute]]

class AssignDriverUseCaseImpl(
    repo: TaxiRequestRepository,
    driverService: DriverService
) extends AssignDriverUseCase:

  def execute(cmd: AssignDriverCommand): IO[Either[DomainError, EnRoute]] =
    for
      request <- repo.find(cmd.requestId)
      result  <- request match
        case None =>
          IO.pure(Left(DomainError.RequestNotFound(cmd.requestId)))
        case Some(state) =>
          state match
            case w: Waiting =>
              for
                available <- driverService.isAvailable(cmd.driverId)
                outcome <- if available then
                    val enRoute = assignDriver(w, cmd.driverId)
                    repo.save(enRoute).as(Right(enRoute))
                  else
                    IO.pure(Left(DomainError.DriverNotAvailable(cmd.driverId)))
              yield outcome
            case _ =>
              IO.pure(Left(DomainError.InvalidState(
                expected = "waiting",
                actual = state.kind
              )))
    yield result
```

### Convert Domain Result to Proto Response

```scala
import com.google.protobuf.timestamp.Timestamp as ProtoTimestamp
import java.time.Instant

def toProtoResponse(
    result: Either[DomainError, EnRoute]
): AssignDriverResponse =
  result match
    case Right(enRoute) =>
      AssignDriverResponse(
        AssignDriverResponse.Result.Success(
          EnRouteState(
            requestId = enRoute.requestId.value.toString,
            passengerId = enRoute.passengerId.value.toString,
            driverId = enRoute.driverId.value.toString,
            assignedAt = Some(toProtoTimestamp(enRoute.assignedAt))
          )
        )
      )
    case Left(err) =>
      AssignDriverResponse(
        AssignDriverResponse.Result.Error(
          AssignDriverError(
            code = toErrorCode(err),
            message = err.message
          )
        )
      )

def toErrorCode(err: DomainError): String = err match
  case _: DomainError.RequestNotFound    => "request_not_found"
  case _: DomainError.InvalidState       => "invalid_state"
  case _: DomainError.DriverNotAvailable => "driver_not_available"

def toProtoTimestamp(instant: Instant): ProtoTimestamp =
  ProtoTimestamp(
    seconds = instant.getEpochSecond,
    nanos = instant.getNano
  )
```

### Wire Up the gRPC Service

```scala
import fs2.grpc.syntax.all.*
import io.grpc.Metadata

class TaxiRequestServiceImpl(
    assignDriver: AssignDriverUseCase
) extends TaxiRequestServiceFs2Grpc[IO, Metadata]:

  override def assignDriver(
      request: ProtoRequest,
      ctx: Metadata
  ): IO[AssignDriverResponse] =
    toDomainCommand(request) match
      case Left(err) =>
        IO.pure(toProtoResponse(Left(err)))
      case Right(cmd) =>
        assignDriver.execute(cmd).map(toProtoResponse)
```

## Error Mapping Summary

The Scala service maps domain errors to both the structured `oneof` response
and gRPC status codes. The client can use either:

| Domain Error | Proto `code` field | gRPC Status (fallback) |
| --- | --- | --- |
| `RequestNotFound` | `"request_not_found"` | `NOT_FOUND` |
| `InvalidState` | `"invalid_state"` | `FAILED_PRECONDITION` |
| `DriverNotAvailable` | `"driver_not_available"` | `UNAVAILABLE` |
| `ValidationError` (from `toDomainCommand`) | `"validation_error"` | `INVALID_ARGUMENT` |

The `oneof` result pattern is preferred because it gives the Rust client
structured access to error details without parsing gRPC status metadata.
The gRPC status code serves as a fallback for generic clients or middleware.

## Key Observations

1. **Proto file is the contract.** Both services generate code from the same
   `.proto` definition. Changes go through the proto file, not ad-hoc
   coordination.

2. **Branded IDs validated at the boundary.** The proto definition uses plain
   `string` for IDs. Both the Rust sender (when parsing the response) and the
   Scala receiver (when parsing the request) validate and convert to branded
   types at their respective boundaries.

3. **Timestamps use Protobuf native type.** `google.protobuf.Timestamp` is
   used instead of string, avoiding ISO 8601 parsing issues. Each language
   converts to its native time type (`chrono::DateTime<Utc>` in Rust,
   `java.time.Instant` in Scala).

4. **Structured errors via `oneof`.** The response contains either a success
   state or a structured error, giving clients type-safe access to error
   details without string parsing.

5. **Domain logic stays inside the service.** The Scala service boundary
   converts proto types to domain types, executes the use case with domain
   types only, then converts the result back to proto types. Proto-generated
   types never appear in the domain layer.
