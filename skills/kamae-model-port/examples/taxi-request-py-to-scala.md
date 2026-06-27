# Worked Example: Taxi Request -- Python to Scala

> **When to read:** Porting a kamae domain model from Python (Pydantic v2) to
> Scala 3. Walk through the full taxi-request aggregate migration step by step.

<!-- constrained-by ../references/type-mapping.md -->
<!-- constrained-by ../references/state-transition-mapping.md -->

---

## Source Model (Python -- kamae-py)

The taxi-request aggregate in Python uses frozen Pydantic v2 models, a
`Literal`-discriminated union, module-level transition functions, `Protocol`
for repository ports, and early-return `Result` in use cases.

```python
# domain/taxi_request/domain_model.py
from pydantic import BaseModel, ConfigDict

class DomainModel(BaseModel):
    model_config = ConfigDict(frozen=True, extra="forbid")
```

```python
# domain/taxi_request/ids.py
from uuid import UUID
from .domain_model import DomainModel

class RequestId(DomainModel):
    value: UUID

class PassengerId(DomainModel):
    value: UUID

class DriverId(DomainModel):
    value: UUID
```

```python
# domain/taxi_request/states.py
from datetime import datetime
from typing import Annotated, Literal
from uuid import UUID

from pydantic import Field

from .domain_model import DomainModel

class WaitingRequest(DomainModel):
    kind: Literal["waiting"] = "waiting"
    request_id: UUID
    passenger_id: UUID
    created_at: datetime

class EnRouteRequest(DomainModel):
    kind: Literal["en_route"] = "en_route"
    request_id: UUID
    passenger_id: UUID
    driver_id: UUID
    assigned_at: datetime

class InTripRequest(DomainModel):
    kind: Literal["in_trip"] = "in_trip"
    request_id: UUID
    passenger_id: UUID
    driver_id: UUID
    started_at: datetime

class CompletedRequest(DomainModel):
    kind: Literal["completed"] = "completed"
    request_id: UUID
    passenger_id: UUID
    driver_id: UUID
    completed_at: datetime

class CancelledRequest(DomainModel):
    kind: Literal["cancelled"] = "cancelled"
    request_id: UUID
    passenger_id: UUID
    cancelled_at: datetime

type TaxiRequest = Annotated[
    WaitingRequest | EnRouteRequest | InTripRequest | CompletedRequest | CancelledRequest,
    Field(discriminator="kind"),
]
```

```python
# domain/taxi_request/transitions.py
from datetime import datetime
from uuid import UUID

from .states import WaitingRequest, EnRouteRequest, CancelledRequest

def assign_driver(
    waiting: WaitingRequest,
    driver_id: UUID,
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

```python
# domain/taxi_request/errors.py
from typing import Annotated, Literal
from uuid import UUID

from pydantic import Field

from .domain_model import DomainModel

class RequestNotFound(DomainModel):
    kind: Literal["request_not_found"] = "request_not_found"
    request_id: UUID

class InvalidState(DomainModel):
    kind: Literal["invalid_state"] = "invalid_state"
    actual: str

class DriverNotAvailable(DomainModel):
    kind: Literal["driver_not_available"] = "driver_not_available"
    driver_id: UUID

type AssignDriverError = Annotated[
    RequestNotFound | InvalidState | DriverNotAvailable,
    Field(discriminator="kind"),
]
```

```python
# domain/taxi_request/events.py
from datetime import datetime
from typing import Literal
from uuid import UUID

from .domain_model import DomainModel

class DriverAssigned(DomainModel):
    event_name: Literal["driver_assigned"] = "driver_assigned"
    request_id: UUID
    driver_id: UUID
    assigned_at: datetime

class RequestCancelled(DomainModel):
    event_name: Literal["request_cancelled"] = "request_cancelled"
    request_id: UUID
    cancelled_at: datetime

type TaxiRequestEvent = DriverAssigned | RequestCancelled
```

```python
# domain/taxi_request/repository.py
from typing import Protocol

from .states import EnRouteRequest, CancelledRequest
from .events import TaxiRequestEvent

class RequestStore(Protocol):
    async def save_en_route(
        self, state: EnRouteRequest, events: tuple[TaxiRequestEvent, ...]
    ) -> None: ...

    async def save_cancelled(
        self, state: CancelledRequest, events: tuple[TaxiRequestEvent, ...]
    ) -> None: ...
```

```python
# use_cases/assign_driver.py
from result import Ok, Err, Result

from domain.taxi_request.states import WaitingRequest, EnRouteRequest
from domain.taxi_request.transitions import assign_driver
from domain.taxi_request.errors import AssignDriverError, RequestNotFound, InvalidState
from domain.taxi_request.events import DriverAssigned
from domain.taxi_request.repository import RequestStore

async def assign_driver_use_case(
    store: RequestStore,
    request_id: UUID,
    driver_id: UUID,
    now: datetime,
) -> Result[EnRouteRequest, AssignDriverError]:
    waiting = await store.find_waiting(request_id)
    if waiting is None:
        return Err(RequestNotFound(request_id=request_id))

    en_route = assign_driver(waiting, driver_id, now)
    event = DriverAssigned(
        request_id=en_route.request_id,
        driver_id=en_route.driver_id,
        assigned_at=en_route.assigned_at,
    )
    await store.save_en_route(en_route, (event,))
    return Ok(en_route)
```

---

## Target Model (Scala 3 -- kamae-scala) -- Step by Step

### Step 1: ID Types

Python frozen wrapper models become Scala opaque types inside companion objects.

**Python source:**

```python
class RequestId(DomainModel):
    value: UUID

class PassengerId(DomainModel):
    value: UUID

class DriverId(DomainModel):
    value: UUID
```

**Scala target:**

```scala
// domain/taxirequest/RequestId.scala
package domain.taxirequest

object RequestId:
  opaque type RequestId = String

  def apply(value: String): Either[String, RequestId] =
    if value.nonEmpty && value.matches(
      "[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}"
    )
    then Right(value)
    else Left(s"Invalid RequestId: $value")

  extension (id: RequestId)
    def value: String = id

export RequestId.RequestId
```

```scala
// domain/taxirequest/PassengerId.scala
package domain.taxirequest

object PassengerId:
  opaque type PassengerId = String

  def apply(value: String): Either[String, PassengerId] =
    if value.nonEmpty && value.matches(
      "[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}"
    )
    then Right(value)
    else Left(s"Invalid PassengerId: $value")

  extension (id: PassengerId)
    def value: String = id

export PassengerId.PassengerId
```

```scala
// domain/driver/DriverId.scala
package domain.driver

object DriverId:
  opaque type DriverId = String

  def apply(value: String): Either[String, DriverId] =
    if value.nonEmpty && value.matches(
      "[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}"
    )
    then Right(value)
    else Left(s"Invalid DriverId: $value")

  extension (id: DriverId)
    def value: String = id

export DriverId.DriverId
```

**What changed and why:**

| Python | Scala | Reason |
| --- | --- | --- |
| `class RequestId(DomainModel): value: UUID` | `opaque type RequestId = String` inside `object` | Scala opaque types give zero-cost type safety at compile time, equivalent to Python's frozen wrapper but with no runtime overhead |
| `UUID` type | `String` with regex validation | Scala typically represents UUIDs as validated strings; `java.util.UUID` is also valid but opaque-over-String is idiomatic kamae-scala |
| Implicit validation via Pydantic's UUID parsing | Explicit `apply()` returning `Either[String, RequestId]` | Boundary validation is explicit -- `apply` is the only way to construct the value |
| `value` field access | `extension (id: RequestId) def value: String` | Opaque types need an extension method to expose the underlying value |

---

### Step 2: State Types

Python frozen Pydantic models become Scala `final case class` instances. The
explicit `kind` discriminant field is dropped because Scala's `enum` case
serves the same role.

**Python source:**

```python
class WaitingRequest(DomainModel):
    kind: Literal["waiting"] = "waiting"
    request_id: UUID
    passenger_id: UUID
    created_at: datetime

class EnRouteRequest(DomainModel):
    kind: Literal["en_route"] = "en_route"
    request_id: UUID
    passenger_id: UUID
    driver_id: UUID
    assigned_at: datetime

class InTripRequest(DomainModel):
    kind: Literal["in_trip"] = "in_trip"
    request_id: UUID
    passenger_id: UUID
    driver_id: UUID
    started_at: datetime

class CompletedRequest(DomainModel):
    kind: Literal["completed"] = "completed"
    request_id: UUID
    passenger_id: UUID
    driver_id: UUID
    completed_at: datetime

class CancelledRequest(DomainModel):
    kind: Literal["cancelled"] = "cancelled"
    request_id: UUID
    passenger_id: UUID
    cancelled_at: datetime
```

**Scala target:**

```scala
// domain/taxirequest/States.scala
package domain.taxirequest

import domain.driver.DriverId
import java.time.Instant

final case class WaitingRequest(
    requestId: RequestId,
    passengerId: PassengerId,
    createdAt: Instant,
)

final case class EnRouteRequest(
    requestId: RequestId,
    passengerId: PassengerId,
    driverId: DriverId,
    assignedAt: Instant,
)

final case class InTripRequest(
    requestId: RequestId,
    passengerId: PassengerId,
    driverId: DriverId,
    startedAt: Instant,
)

final case class CompletedRequest(
    requestId: RequestId,
    passengerId: PassengerId,
    driverId: DriverId,
    completedAt: Instant,
)

final case class CancelledRequest(
    requestId: RequestId,
    passengerId: PassengerId,
    cancelledAt: Instant,
)
```

**What changed and why:**

| Python | Scala | Reason |
| --- | --- | --- |
| `class Waiting(DomainModel)` with `ConfigDict(frozen=True)` | `final case class WaitingRequest(...)` | Scala case classes are immutable by default; `final` prevents subclassing |
| `kind: Literal["waiting"] = "waiting"` | No `kind` field | The enum case (Step 3) acts as the discriminant -- no explicit tag needed |
| `request_id: UUID` | `requestId: RequestId` | snake_case to camelCase; raw `UUID` replaced by the opaque ID type for compile-time safety |
| `datetime` | `Instant` | Python's `datetime` maps to `java.time.Instant` for UTC timestamps |
| `extra="forbid"` | Not needed | Case classes do not accept extra fields by construction |

---

### Step 3: Discriminated Union

Python's `Annotated[..., Field(discriminator="kind")]` becomes a Scala `enum`
whose cases wrap the state case classes.

**Python source:**

```python
type TaxiRequest = Annotated[
    WaitingRequest | EnRouteRequest | InTripRequest | CompletedRequest | CancelledRequest,
    Field(discriminator="kind"),
]
```

**Scala target:**

```scala
// domain/taxirequest/TaxiRequest.scala
package domain.taxirequest

enum TaxiRequest:
  case Waiting(value: WaitingRequest)
  case EnRoute(value: EnRouteRequest)
  case InTrip(value: InTripRequest)
  case Completed(value: CompletedRequest)
  case Cancelled(value: CancelledRequest)
```

**What changed and why:**

| Python | Scala | Reason |
| --- | --- | --- |
| `type TaxiRequest = Annotated[A \| B \| ..., Field(discriminator="kind")]` | `enum TaxiRequest { case Waiting(value: WaitingRequest), ... }` | Scala enums are sealed ADTs; the compiler enforces exhaustive matching |
| `Field(discriminator="kind")` | Enum case name | Scala uses the case name as the implicit discriminant -- no runtime `kind` string |
| `TypeAdapter(TaxiRequest)` for boundary parsing | Circe codec / Play JSON `Reads` (separate boundary layer) | Serialization is handled by codecs at the boundary, not in the domain |

---

### Step 4: Transition Functions

Python module-level pure functions become Scala extension methods on the
source state. The transition return type wraps both the new state and domain
events in a `Transition`.

**Python source:**

```python
def assign_driver(
    waiting: WaitingRequest,
    driver_id: UUID,
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

**Scala target:**

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

**What changed and why:**

| Python | Scala | Reason |
| --- | --- | --- |
| `def assign_driver(waiting, driver_id, now)` standalone function | `extension (request: WaitingRequest) def assignDriver(driver)` | Scala idiom groups transitions as extension methods on the source state |
| snake_case `assign_driver` | camelCase `assignDriver` | Scala naming convention |
| Returns bare `EnRouteRequest` | Returns `Either[Error, Transition[EnRouteRequest, TaxiRequestEvent]]` | Scala bundles the new state and events together in `Transition`; wraps in `Either` when the transition can fail |
| `driver_id: UUID, now: datetime` as separate args | `driver: DriverAssignment` value object | Scala packs related args into a value object to reduce parameter count and clarify intent |
| Infallible `cancel` returns bare state | Infallible `cancel` returns `Transition` (no `Either`) | When a transition cannot fail, skip the `Either` wrapper -- only wrap when failure is possible |
| Events constructed separately in use-case layer | Events constructed inside the transition and bundled in `Transition` | kamae-scala co-locates event construction with state construction for atomicity |

---

### Step 5: Error Types

Python Pydantic discriminated union errors become a Scala enum ADT. The
`kind` Literal field is replaced by the enum case name.

**Python source:**

```python
class RequestNotFound(DomainModel):
    kind: Literal["request_not_found"] = "request_not_found"
    request_id: UUID

class InvalidState(DomainModel):
    kind: Literal["invalid_state"] = "invalid_state"
    actual: str

class DriverNotAvailable(DomainModel):
    kind: Literal["driver_not_available"] = "driver_not_available"
    driver_id: UUID

type AssignDriverError = Annotated[
    RequestNotFound | InvalidState | DriverNotAvailable,
    Field(discriminator="kind"),
]
```

**Scala target:**

```scala
// domain/taxirequest/Errors.scala
package domain.taxirequest

import domain.driver.DriverId

enum AssignDriverError:
  case RequestNotFound(requestId: RequestId)
  case InvalidState(actual: String)
  case DriverNotAvailable(driverId: DriverId)
```

**What changed and why:**

| Python | Scala | Reason |
| --- | --- | --- |
| Separate classes per error, then `type AssignDriverError = Annotated[...]` | Single `enum AssignDriverError` with cases | Scala enum is the natural ADT; one enum per use-case error keeps exhaustive match at each call site |
| `kind: Literal["request_not_found"] = "request_not_found"` | Case name `RequestNotFound` | Enum case name is the discriminant -- no explicit `kind` field |
| `class RequestNotFound(DomainModel)` inherits frozen config | `case RequestNotFound(requestId: RequestId)` | Enum case classes are immutable and structurally sealed; no base class needed |
| `UUID` in error fields | Opaque ID types (`RequestId`, `DriverId`) | Preserves type safety even in error values |

---

### Step 6: Domain Events

Python frozen Pydantic event models with `event_name: Literal[...]` become a
Scala enum ADT. The pattern mirrors the error type migration.

**Python source:**

```python
class DriverAssigned(DomainModel):
    event_name: Literal["driver_assigned"] = "driver_assigned"
    request_id: UUID
    driver_id: UUID
    assigned_at: datetime

class RequestCancelled(DomainModel):
    event_name: Literal["request_cancelled"] = "request_cancelled"
    request_id: UUID
    cancelled_at: datetime

type TaxiRequestEvent = DriverAssigned | RequestCancelled
```

**Scala target:**

```scala
// domain/taxirequest/Events.scala
package domain.taxirequest

import domain.driver.DriverId
import java.time.Instant

enum TaxiRequestEvent:
  case DriverAssigned(
      requestId: RequestId,
      driverId: DriverId,
  )
  case RequestCancelled(
      requestId: RequestId,
  )
```

**What changed and why:**

| Python | Scala | Reason |
| --- | --- | --- |
| Separate event classes + `type` union alias | Single `enum TaxiRequestEvent` | Same ADT pattern as errors; one sealed enum per aggregate's event family |
| `event_name: Literal["driver_assigned"]` | Enum case name `DriverAssigned` | Case name replaces the explicit discriminant string |
| `assigned_at: datetime` on event | Omitted from event (carried on state) | Scala kamae events carry only the IDs needed for downstream consumers; timestamps live on the state. This is a design choice -- include timestamp fields if consumers need them |

---

### Step 7: Repository Port

Python `Protocol` (structural typing) becomes a Scala `trait[F[_]]`
(higher-kinded type for the effect). This is the largest conceptual shift in
the migration.

**Python source:**

```python
class RequestStore(Protocol):
    async def save_en_route(
        self, state: EnRouteRequest, events: tuple[TaxiRequestEvent, ...]
    ) -> None: ...

    async def save_cancelled(
        self, state: CancelledRequest, events: tuple[TaxiRequestEvent, ...]
    ) -> None: ...
```

**Scala target:**

```scala
// domain/taxirequest/TaxiRequestRepository.scala
package domain.taxirequest

trait TaxiRequestRepository[F[_]]:
  def findWaiting(requestId: RequestId): F[Option[WaitingRequest]]

  def saveAssigned(
      state: EnRouteRequest,
      events: List[TaxiRequestEvent],
  ): F[Unit]

  def saveCancelled(
      state: CancelledRequest,
      events: List[TaxiRequestEvent],
  ): F[Unit]
```

**What changed and why:**

| Python | Scala | Reason |
| --- | --- | --- |
| `class RequestStore(Protocol)` | `trait TaxiRequestRepository[F[_]]` | Python's `Protocol` is structural; Scala uses a higher-kinded trait to abstract over the effect type (IO, Future, Id) |
| `async def` | `F[_]` wraps return type | Python's `async` maps to the effect `F`; concrete implementations bind `F` to `IO`, `Future`, etc. |
| `tuple[TaxiRequestEvent, ...]` | `List[TaxiRequestEvent]` | Scala idiom prefers `List` for ordered event collections |
| `save_en_route` | `saveAssigned` | camelCase + slightly more descriptive name (saves the assigned/en-route state) |
| No find method shown (separate resolver) | `findWaiting` included | Scala repositories typically combine read and write in the same trait per aggregate |

---

### Step 8: Use Case

Python's async function with early-return `Result` becomes a Scala class with
a `for`-comprehension over `Either`. This is where `Ok`/`Err` maps to
`Right`/`Left`.

**Python source:**

```python
async def assign_driver_use_case(
    store: RequestStore,
    request_id: UUID,
    driver_id: UUID,
    now: datetime,
) -> Result[EnRouteRequest, AssignDriverError]:
    waiting = await store.find_waiting(request_id)
    if waiting is None:
        return Err(RequestNotFound(request_id=request_id))

    en_route = assign_driver(waiting, driver_id, now)
    event = DriverAssigned(
        request_id=en_route.request_id,
        driver_id=en_route.driver_id,
        assigned_at=en_route.assigned_at,
    )
    await store.save_en_route(en_route, (event,))
    return Ok(en_route)
```

**Scala target:**

```scala
// usecases/AssignDriver.scala
package usecases

import cats.Monad
import cats.syntax.all.*
import domain.driver.{DriverAssignment, DriverId}
import domain.taxirequest.*
import domain.shared.Transition
import java.time.Instant

class AssignDriver[F[_]: Monad](
    repository: TaxiRequestRepository[F],
):
  def execute(
      requestId: RequestId,
      driverAssignment: DriverAssignment,
  ): F[Either[AssignDriverError, EnRouteRequest]] =
    for
      maybeWaiting <- repository.findWaiting(requestId)
      result = for
        waiting    <- maybeWaiting
                        .toRight(AssignDriverError.RequestNotFound(requestId))
        transition <- waiting.assignDriver(driverAssignment)
      yield transition
      outcome <- result match
        case Right(transition) =>
          repository
            .saveAssigned(transition.state, transition.events)
            .as(Right(transition.state))
        case Left(error) =>
          Monad[F].pure(Left(error))
    yield outcome
```

**What changed and why:**

| Python | Scala | Reason |
| --- | --- | --- |
| `async def assign_driver_use_case(store, ...)` free function | `class AssignDriver[F[_]: Monad](repository)` with `def execute(...)` | Scala use cases are classes that take dependencies via constructor; `F[_]: Monad` abstracts the effect |
| `Result[EnRouteRequest, AssignDriverError]` | `F[Either[AssignDriverError, EnRouteRequest]]` | `F` wraps the effectful computation; `Either` carries the domain success/failure. Note the type parameter order is swapped: Python `Result[Ok, Err]` vs Scala `Either[Left, Right]` |
| `Ok(en_route)` / `Err(RequestNotFound(...))` | `Right(transition.state)` / `Left(AssignDriverError.RequestNotFound(...))` | Direct mapping of Result vocabulary |
| `if waiting is None: return Err(...)` | `.toRight(AssignDriverError.RequestNotFound(...))` | `Option.toRight` converts `None` to `Left`, `Some(v)` to `Right(v)` -- replaces the null-check-and-return pattern |
| Sequential `await` calls | Outer `for`-comprehension over `F`, inner pure `for` over `Either` | The outer `for` sequences effectful operations; the inner `for` composes pure domain logic. This two-layer pattern is the standard kamae-scala use-case shape |
| Event constructed in use case, passed to `save` | Event constructed in transition (Step 4), passed through `Transition` | kamae-scala co-locates event creation with state creation; the use case just forwards `transition.events` |

---

## Summary: Naming Convention Changes

| Element | Python | Scala |
| --- | --- | --- |
| Field names | `snake_case` (`request_id`) | `camelCase` (`requestId`) |
| Function names | `snake_case` (`assign_driver`) | `camelCase` (`assignDriver`) |
| Type names | PascalCase (`WaitingRequest`) | PascalCase (`WaitingRequest`) |
| Discriminant | Explicit `kind: Literal["waiting"]` field | Implicit via enum case name (`TaxiRequest.Waiting`) |
| Event discriminant | Explicit `event_name: Literal["driver_assigned"]` | Implicit via enum case name (`TaxiRequestEvent.DriverAssigned`) |
| File names | `snake_case` (`taxi_request.py`) | PascalCase or package path (`taxirequest/States.scala`) |
| Module/package | `snake_case` (`domain.taxi_request`) | Dot-separated lowercase (`domain.taxirequest`) |

## Summary: Structural Changes

| Concept | Python | Scala |
| --- | --- | --- |
| Immutability | `ConfigDict(frozen=True)` | `final case class` (immutable by default) |
| ID types | Frozen Pydantic wrapper | Opaque type in companion object |
| Union | `Annotated[... Field(discriminator="kind")]` | `enum` with case wrappers |
| Transitions | Module-level pure functions | Extension methods on source state |
| Transition return | Bare new state (events separate) | `Transition[TState, TEvent]` bundling state and events |
| Error types | Pydantic DU with `kind` Literal | Enum ADT |
| Repository port | `Protocol` with `async def` | `trait[F[_]]` with effect-typed returns |
| Use case | `async def` with early-return `Result` | Class with `Monad` context bound and `for`-comprehension |
| Result type | `Result[OkType, ErrType]` (ok first) | `Either[ErrType, OkType]` (error first, by convention) |
| None handling | `if x is None: return Err(...)` | `Option.toRight(...)` |
