# Error Handling Mapping

When to read: Porting error types, Result flows, controller-layer error
conversion, or error composition between languages.

---

## Overview

Every kamae language defines **per-use-case error types** as discriminated
unions (or enums). The `kind` discriminant in TS/Python becomes the enum
variant name in Rust/Scala. Errors flow through a Result type and are
converted to HTTP responses at the controller boundary.

---

## 1. Error Type Definitions

### TypeScript -- DU with `kind` field

Each variant is a `Readonly<{...}>` object literal with a `kind` string
literal. The union is a type alias.

```typescript
// domain/taxi-request/errors.ts
import type { RequestId } from "./request-id";

export type AssignDriverError =
  | Readonly<{ kind: "RequestNotFound"; requestId: RequestId }>
  | Readonly<{ kind: "InvalidState"; actual: string }>
  | Readonly<{ kind: "DriverUnavailable"; driverId: string }>;
```

Constructors are plain object literals at call sites:

```typescript
return err({ kind: "RequestNotFound", requestId: cmd.requestId });
```

### Python -- Frozen Pydantic models with `Literal` kind

Each variant is a separate frozen model. The union uses
`Annotated[..., Field(discriminator="kind")]`.

```python
# domain/taxi_request/errors.py
from typing import Annotated, Literal, Union
from pydantic import Field
from ..shared.domain_model import DomainModel
from .request_id import RequestId


class RequestNotFound(DomainModel):
    kind: Literal["request_not_found"] = "request_not_found"
    request_id: RequestId


class InvalidState(DomainModel):
    kind: Literal["invalid_state"] = "invalid_state"
    actual: str


class DriverUnavailable(DomainModel):
    kind: Literal["driver_unavailable"] = "driver_unavailable"
    driver_id: str


AssignDriverError = Annotated[
    Union[RequestNotFound, InvalidState, DriverUnavailable],
    Field(discriminator="kind"),
]
```

Constructors are model instantiation:

```python
return Err(RequestNotFound(request_id=cmd.request_id))
```

### Rust -- `thiserror` enum

Each variant is an enum arm. `#[error("...")]` provides the Display impl.

```rust
// domain/taxi_request/errors.rs
use crate::domain::taxi_request::RequestId;
use thiserror::Error;

#[derive(Debug, Error)]
pub enum AssignDriverError {
    #[error("Request {request_id} not found")]
    RequestNotFound { request_id: RequestId },

    #[error("Invalid state: {actual}")]
    InvalidState { actual: String },

    #[error("Driver {driver_id} unavailable")]
    DriverUnavailable { driver_id: String },
}
```

Constructors use enum variant syntax:

```rust
return Err(AssignDriverError::RequestNotFound {
    request_id: cmd.request_id.clone(),
});
```

### Scala -- Sealed enum

Each variant is a `case` inside an `enum`.

```scala
// domain/taxirequest/Errors.scala
package domain.taxirequest

enum AssignDriverError:
  case RequestNotFound(requestId: RequestId)
  case InvalidState(actual: String)
  case DriverUnavailable(driverId: String)
```

Constructors use enum case syntax:

```scala
Left(AssignDriverError.RequestNotFound(cmd.requestId))
```

---

## 2. Result Type Mapping

| Language   | Result type                          | Ok constructor    | Err constructor   | Library                          |
| ---------- | ------------------------------------ | ----------------- | ----------------- | -------------------------------- |
| TypeScript | `Result<T, E>`                       | `ok(value)`       | `err(error)`      | neverthrow, fp-ts Either, byethrow, option-t |
| Python     | `Result[T, E]`                       | `Ok(value)`       | `Err(error)`      | result (`pip install result`)    |
| Rust       | `Result<T, E>`                       | `Ok(value)`       | `Err(error)`      | std                              |
| Scala      | `Either[E, T]`                       | `Right(value)`    | `Left(error)`     | std / cats                       |

Note the type parameter order: TS and Rust put the success type first
(`Result<T, E>`), Scala's `Either` puts the error first (`Either[E, T]`).
Python's result library follows `Result[T, E]`.

---

## 3. Composition Patterns

### TypeScript -- `andThen` / `map` chains

```typescript
import { ok, err, Result } from "neverthrow";

function assignDriver(
  deps: Deps,
  cmd: Command,
): Result<void, AssignDriverError> {
  return deps.findRequest(cmd.requestId)
    .mapErr(() => ({ kind: "RequestNotFound" as const, requestId: cmd.requestId }))
    .andThen((request) =>
      request.kind === "Waiting"
        ? ok(request)
        : err({ kind: "InvalidState" as const, actual: request.kind })
    )
    .andThen((waiting) => {
      const next = TaxiRequest.assignDriver(waiting, cmd.driverId, deps.now());
      return deps.save(next);
    });
}
```

### Python -- Early return

```python
from result import Ok, Err, Result

def assign_driver(
    deps: Deps,
    cmd: Command,
) -> Result[None, AssignDriverError]:
    request = deps.find_request(cmd.request_id)
    if request is None:
        return Err(RequestNotFound(request_id=cmd.request_id))

    if request.kind != "waiting":
        return Err(InvalidState(actual=request.kind))

    next_state = transitions.assign_driver(request, cmd.driver_id, deps.now())
    deps.save(next_state)
    return Ok(None)
```

### Rust -- `?` operator

```rust
pub fn assign_driver(
    deps: &mut impl Deps,
    cmd: Command,
) -> Result<(), AssignDriverError> {
    let request = deps
        .find_request(&cmd.request_id)
        .ok_or(AssignDriverError::RequestNotFound {
            request_id: cmd.request_id.clone(),
        })?;

    let waiting = match request {
        TaxiRequestState::Waiting(w) => w,
        other => return Err(AssignDriverError::InvalidState {
            actual: other.kind_name().to_string(),
        }),
    };

    let transition = waiting.assign_driver(cmd.driver_assignment)?;
    deps.save_with_events(transition)?;
    Ok(())
}
```

### Scala -- `for`-comprehension

```scala
def assignDriver(
    deps: Deps,
    cmd: Command,
): Either[AssignDriverError, Unit] =
  for
    request  <- deps.findRequest(cmd.requestId)
                  .toRight(AssignDriverError.RequestNotFound(cmd.requestId))
    waiting  <- request match
                  case w: WaitingRequest => Right(w)
                  case other => Left(AssignDriverError.InvalidState(other.kind))
    transition <- waiting.assignDriver(cmd.driverAssignment)
    _        <- deps.saveWithEvents(transition)
  yield ()
```

---

## 4. Controller-Layer Error Conversion

At the HTTP boundary, each language pattern-matches on the error type and
maps to the appropriate HTTP status code.

### TypeScript

```typescript
// adapters/http/assign-driver-handler.ts
import { match } from "ts-pattern";

export function handleAssignDriverError(
  error: AssignDriverError,
): HttpResponse {
  return match(error)
    .with({ kind: "RequestNotFound" }, (e) => ({
      status: 404,
      body: { message: `Request ${e.requestId} not found` },
    }))
    .with({ kind: "InvalidState" }, (e) => ({
      status: 409,
      body: { message: `Invalid state: ${e.actual}` },
    }))
    .with({ kind: "DriverUnavailable" }, (e) => ({
      status: 422,
      body: { message: `Driver ${e.driverId} unavailable` },
    }))
    .exhaustive();
}
```

Or with a `switch`:

```typescript
switch (error.kind) {
  case "RequestNotFound":
    return { status: 404, body: { message: `Request ${error.requestId} not found` } };
  case "InvalidState":
    return { status: 409, body: { message: `Invalid state: ${error.actual}` } };
  case "DriverUnavailable":
    return { status: 422, body: { message: `Driver ${error.driverId} unavailable` } };
  default:
    assertNever(error);
}
```

### Python

```python
# adapters/http/assign_driver_handler.py
from fastapi.responses import JSONResponse

def handle_assign_driver_error(error: AssignDriverError) -> JSONResponse:
    match error:
        case RequestNotFound(request_id=rid):
            return JSONResponse(
                status_code=404,
                content={"message": f"Request {rid} not found"},
            )
        case InvalidState(actual=actual):
            return JSONResponse(
                status_code=409,
                content={"message": f"Invalid state: {actual}"},
            )
        case DriverUnavailable(driver_id=did):
            return JSONResponse(
                status_code=422,
                content={"message": f"Driver {did} unavailable"},
            )
        case _ as unreachable:
            assert_never(unreachable)
```

### Rust

```rust
// adapters/http/assign_driver_handler.rs
impl axum::response::IntoResponse for AssignDriverError {
    fn into_response(self) -> axum::response::Response {
        let (status, message) = match &self {
            AssignDriverError::RequestNotFound { request_id } => (
                StatusCode::NOT_FOUND,
                format!("Request {} not found", request_id),
            ),
            AssignDriverError::InvalidState { actual } => (
                StatusCode::CONFLICT,
                format!("Invalid state: {}", actual),
            ),
            AssignDriverError::DriverUnavailable { driver_id } => (
                StatusCode::UNPROCESSABLE_ENTITY,
                format!("Driver {} unavailable", driver_id),
            ),
        };
        (status, Json(serde_json::json!({ "message": message }))).into_response()
    }
}
```

### Scala

```scala
// adapters/http/AssignDriverHandler.scala
def toResponse(error: AssignDriverError): Response = error match
  case AssignDriverError.RequestNotFound(rid) =>
    Response(Status.NotFound, s"Request $rid not found")
  case AssignDriverError.InvalidState(actual) =>
    Response(Status.Conflict, s"Invalid state: $actual")
  case AssignDriverError.DriverUnavailable(did) =>
    Response(Status.UnprocessableEntity, s"Driver $did unavailable")
```

---

## 5. Error Nesting and Aggregation

When a use case calls into sub-operations that have their own error types,
the outer error must wrap or flatten the inner errors.

### TypeScript -- Union widening or wrapping variant

```typescript
export type CreateOrderError =
  | AssignDriverError
  | Readonly<{ kind: "PaymentFailed"; reason: string }>;
```

### Python -- Union widening

```python
CreateOrderError = Annotated[
    Union[RequestNotFound, InvalidState, DriverUnavailable, PaymentFailed],
    Field(discriminator="kind"),
]
```

### Rust -- `#[from]` for automatic conversion

```rust
#[derive(Debug, Error)]
pub enum CreateOrderError {
    #[error(transparent)]
    AssignDriver(#[from] AssignDriverError),

    #[error("Payment failed: {reason}")]
    PaymentFailed { reason: String },
}
```

### Scala -- ADT nesting

```scala
enum CreateOrderError:
  case AssignDriver(inner: AssignDriverError)
  case PaymentFailed(reason: String)
```

---

## 6. Key Porting Notes

| Concern | Guidance |
| ------- | -------- |
| **`kind` field presence** | TS and Python errors carry an explicit `kind` string field because they are data values distinguished at runtime by that field. Rust and Scala use enum variant names -- there is no `kind` field. When porting from TS/Python to Rust/Scala, drop the `kind` field and use the variant name. When porting to TS/Python, add the `kind` field. |
| **Error granularity** | All four languages define errors **per use case**, not per aggregate. Preserve this when porting. |
| **Result parameter order** | TS/Rust/Python: `Result<T, E>` (success first). Scala: `Either[E, T]` (error first). Double-check when translating signatures. |
| **Composition style** | TS uses `.andThen()`, Python uses early return, Rust uses `?`, Scala uses `for`-comprehension. These are semantically equivalent -- choose the idiomatic form for the target language. |
| **thiserror (Rust)** | Rust's `thiserror` crate generates `Display` and `Error` impls from `#[error("...")]` annotations. Other languages do not need this -- TS/Python errors are plain data, Scala enums get `toString` for free. |
| **Exhaustive matching** | TS needs `assertNever` in `default`, Python needs `assert_never` in wildcard case, Rust and Scala get compile-time exhaustiveness from `match` on sealed types. Ensure the target language's exhaustiveness mechanism is in place after porting. |
| **Error wrapping depth** | Rust's `#[from]` creates implicit conversions via `?`. Other languages require explicit wrapping. When porting from Rust, make the wrapping explicit in the target language. |
