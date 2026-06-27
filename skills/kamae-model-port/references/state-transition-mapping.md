# State Transition Mapping

When to read: Porting state transitions, use cases, domain events,
exhaustive branching, or transition outcome types between languages.

---

## Overview

All four kamae languages model state transitions as **pure functions** that
accept a narrow source state and return a narrow target state (or a
transition wrapper carrying events). The differences lie in how each
language groups those functions, handles self-consumption, and composes
results in the use-case layer.

---

## 1. Transition Function Shape

### TypeScript -- Companion Object methods

Functions are grouped in a `const` companion object matching the aggregate
name. Each function is a pure arrow that takes the narrow source state and
injected dependencies as arguments and returns the target state directly.

```typescript
// domain/taxi-request/transitions.ts
import type { WaitingRequest, EnRouteRequest } from "./states";
import type { DriverId } from "../driver/driver-id";

export const TaxiRequest = {
  assignDriver(
    request: WaitingRequest,
    driverId: DriverId,
    now: Date,
  ): EnRouteRequest {
    return {
      kind: "EnRoute",
      requestId: request.requestId,
      passengerId: request.passengerId,
      driverId,
      assignedAt: now,
    };
  },

  cancel(request: WaitingRequest, now: Date): CancelledRequest {
    return {
      kind: "Cancelled",
      requestId: request.requestId,
      passengerId: request.passengerId,
      cancelledAt: now,
    };
  },
} as const;
```

### Python -- Module-level pure functions

Each function lives at module scope. The first positional argument is the
narrow source state (a frozen Pydantic model). Time, IDs, and other
volatile values are injected as explicit arguments.

```python
# domain/taxi_request/transitions.py
from datetime import datetime
from uuid import UUID

from .states import WaitingRequest, EnRouteRequest, CancelledRequest
from ..driver.driver_id import DriverId


def assign_driver(
    waiting: WaitingRequest,
    driver_id: DriverId,
    now: datetime,
) -> EnRouteRequest:
    return EnRouteRequest(
        request_id=waiting.request_id,
        passenger_id=waiting.passenger_id,
        driver_id=driver_id,
        assigned_at=now,
    )


def cancel(
    waiting: WaitingRequest,
    now: datetime,
) -> CancelledRequest:
    return CancelledRequest(
        request_id=waiting.request_id,
        passenger_id=waiting.passenger_id,
        cancelled_at=now,
    )
```

### Rust -- `impl` methods consuming `self`

Transitions are methods on the source state struct. They **consume**
`self` by value, preventing reuse of the old state. They return
`Result<Transition<TState>, DomainError>` so the caller gets both the new
state and any domain events.

```rust
// domain/taxi_request/transitions.rs
use crate::domain::taxi_request::states::{EnRouteRequest, WaitingRequest};
use crate::domain::taxi_request::events::TaxiRequestEvent;
use crate::domain::taxi_request::errors::AssignDriverError;
use crate::domain::driver::DriverAssignment;
use crate::domain::shared::Transition;

impl WaitingRequest {
    pub fn assign_driver(
        self,
        driver: DriverAssignment,
    ) -> Result<Transition<EnRouteRequest>, AssignDriverError> {
        let state = EnRouteRequest {
            request_id: self.request_id,
            passenger_id: self.passenger_id,
            driver_id: driver.driver_id,
            assigned_at: driver.assigned_at,
        };
        let event = TaxiRequestEvent::DriverAssigned {
            request_id: state.request_id.clone(),
            driver_id: state.driver_id.clone(),
        };
        Ok(Transition {
            state,
            events: vec![event],
        })
    }

    pub fn cancel(
        self,
        now: chrono::DateTime<chrono::Utc>,
    ) -> Transition<CancelledRequest> {
        let state = CancelledRequest {
            request_id: self.request_id,
            passenger_id: self.passenger_id,
            cancelled_at: now,
        };
        let event = TaxiRequestEvent::RequestCancelled {
            request_id: state.request_id.clone(),
        };
        Transition {
            state,
            events: vec![event],
        }
    }
}
```

### Scala -- Extension methods

Transitions are `extension` methods on the source state case class. They
return `Either[DomainError, Transition[TState, TEvent]]`.

```scala
// domain/taxirequest/Transitions.scala
package domain.taxirequest

import domain.driver.DriverAssignment
import domain.shared.Transition
import java.time.Instant

extension (request: WaitingRequest)
  def assignDriver(
      driver: DriverAssignment
  ): Either[AssignDriverError, Transition[EnRouteRequest, TaxiRequestEvent]] =
    val state = EnRouteRequest(
      requestId = request.requestId,
      passengerId = request.passengerId,
      driverId = driver.driverId,
      assignedAt = driver.assignedAt,
    )
    val event = TaxiRequestEvent.DriverAssigned(
      requestId = state.requestId,
      driverId = state.driverId,
    )
    Right(Transition(state, List(event)))

  def cancel(
      now: Instant
  ): Transition[CancelledRequest, TaxiRequestEvent] =
    val state = CancelledRequest(
      requestId = request.requestId,
      passengerId = request.passengerId,
      cancelledAt = now,
    )
    val event = TaxiRequestEvent.RequestCancelled(
      requestId = state.requestId
    )
    Transition(state, List(event))
```

---

## 2. Transition Outcome Types

| Language   | Type                                        | Events carried |
| ---------- | ------------------------------------------- | -------------- |
| TypeScript | Plain target state value                    | None -- events emitted separately in use-case layer |
| Python     | `TransitionOutcome[TState, TEvent]`         | `events: list[TEvent]` |
| Rust       | `Transition<TState> { state, events: Vec<ConcreteEvent> }` | `Vec<ConcreteEvent>` |
| Scala      | `Transition[TState, TEvent](state, events: List[TEvent])`   | `List[TEvent]` |

> **Note:** Rust's `Transition` takes only one type parameter (the state type);
> the event type in `events: Vec<...>` is a concrete enum, not a generic parameter.
> Python and Scala use two type parameters (state and event). This reflects an
> idiomatic difference: Rust enum variants are already closed, so a second generic
> parameter adds no value.

TypeScript keeps transitions lightweight -- the companion object method
returns only the new state. Event construction happens in the use-case
layer or an explicit `emitEvents` helper. When porting **from** TS, you
must decide which events to bundle into the `Transition` wrapper.

When porting **to** TS, extract event construction from the transition and
move it to the use-case function.

---

## 3. Use-Case Layer Composition

### TypeScript -- `andThen` chains (neverthrow)

```typescript
// use-cases/assign-driver.ts
import { ok, err, Result } from "neverthrow";

export function assignDriver(
  deps: AssignDriverDeps,
  cmd: AssignDriverCommand,
): Result<void, AssignDriverError> {
  return deps.findRequest(cmd.requestId)
    .andThen((request) => {
      if (request.kind !== "Waiting") {
        return err({ kind: "InvalidState" as const, actual: request.kind });
      }
      return ok(request);
    })
    .andThen((waiting) => {
      const next = TaxiRequest.assignDriver(waiting, cmd.driverId, deps.now());
      return deps.save(next);
    });
}
```

### Python -- Early return with `Result`

```python
# use_cases/assign_driver.py
from result import Ok, Err, Result

def assign_driver(
    deps: AssignDriverDeps,
    cmd: AssignDriverCommand,
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
// use_cases/assign_driver.rs
pub fn assign_driver(
    deps: &mut impl AssignDriverDeps,
    cmd: AssignDriverCommand,
) -> Result<(), AssignDriverError> {
    let request = deps
        .find_request(&cmd.request_id)
        .ok_or(AssignDriverError::RequestNotFound {
            request_id: cmd.request_id.clone(),
        })?;

    let waiting = match request {
        TaxiRequestState::Waiting(w) => w,
        other => return Err(AssignDriverError::InvalidState {
            actual: other.kind_name(),
        }),
    };

    let transition = waiting.assign_driver(cmd.driver_assignment)?;
    deps.save_with_events(transition)?;
    Ok(())
}
```

### Scala -- `for`-comprehension

```scala
// usecases/AssignDriver.scala
def assignDriver(
    deps: AssignDriverDeps,
    cmd: AssignDriverCommand,
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

## 4. Exhaustive Checking

When matching on the aggregate union to route to the correct transition,
every language enforces exhaustiveness differently.

### TypeScript -- `assertNever`

```typescript
function handleRequest(request: TaxiRequestState): void {
  switch (request.kind) {
    case "Waiting":
      // ...
      break;
    case "EnRoute":
      // ...
      break;
    case "Completed":
      // ...
      break;
    case "Cancelled":
      // ...
      break;
    default:
      assertNever(request); // compile error if a case is missing
  }
}

function assertNever(value: never): never {
  throw new Error(`Unhandled value: ${JSON.stringify(value)}`);
}
```

### Python -- `assert_never`

```python
from typing import assert_never

def handle_request(request: TaxiRequestState) -> None:
    match request:
        case WaitingRequest():
            ...
        case EnRouteRequest():
            ...
        case CompletedRequest():
            ...
        case CancelledRequest():
            ...
        case _ as unreachable:
            assert_never(unreachable)  # type error if a case is missing
```

### Rust -- Exhaustive `match`

```rust
fn handle_request(request: TaxiRequestState) {
    match request {
        TaxiRequestState::Waiting(w) => { /* ... */ }
        TaxiRequestState::EnRoute(e) => { /* ... */ }
        TaxiRequestState::Completed(c) => { /* ... */ }
        TaxiRequestState::Cancelled(x) => { /* ... */ }
        // Compiler error if a variant is missing -- no helper needed
    }
}
```

### Scala -- Sealed `match`

```scala
def handleRequest(request: TaxiRequestState): Unit = request match
  case w: WaitingRequest   => ???
  case e: EnRouteRequest   => ???
  case c: CompletedRequest => ???
  case x: CancelledRequest => ???
  // Compiler warning (or error with -Werror) if a case is missing
```

---

## 5. Key Porting Notes

| Concern | Guidance |
| ------- | -------- |
| **Self-consumption (Rust)** | Rust transitions consume `self` by value, preventing reuse of the old state. When porting to TS/Python/Scala, rely on immutability (frozen models, readonly types) instead -- the old binding is still reachable but must not be mutated. |
| **Companion object grouping (TS)** | TS groups all transitions in a single companion `const`. When porting to Rust, distribute them as `impl` blocks on each source state. When porting to Scala, use `extension` blocks. When porting to Python, use module-level functions. |
| **Event bundling** | TS typically does not bundle events in transition returns. When porting from TS to Rust/Scala/Python, decide which events to wrap in `Transition`. When porting to TS, extract events to the use-case layer. |
| **Time and ID injection** | All languages inject `now` and generated IDs as arguments rather than calling clocks/generators inside transitions. Preserve this when porting. |
| **Result wrapping** | Infallible transitions in Rust still often return `Transition` without `Result`. Match this -- do not wrap in `Result` unless the transition can actually fail. |
| **Extension vs impl** | Scala `extension` methods and Rust `impl` methods look similar but differ: Scala extensions are resolved at compile time by import scope, Rust `impl` blocks are inherent. When porting between them, verify visibility and import paths. |
