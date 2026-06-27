# Persistence and Event Pattern Mapping

> **When to read:** Porting repository ports, domain events, outbox patterns,
> or atomic persist between kamae languages.

The atomic persist principle is universal across all kamae languages: state and
events must be written in a single transaction. The implementation varies by
language and ecosystem, but the structural guarantee is the same — no dual
writes, no event-then-state or state-then-event ordering bugs.

---

## Repository Port Patterns

### TypeScript: Function-property types, Resolver/Store split

TS repositories split read (Resolver) from write (Store) following Interface
Segregation. Each is a `type` with function properties (not an interface — to
prevent declaration merging).

```typescript
// Read side — thin, query-only
type TaxiRequestResolver = Readonly<{
  findById: (id: RequestId) => ResultAsync<TaxiRequest | null, RepositoryError>;
  findActiveByPassenger: (id: PassengerId) => ResultAsync<readonly TaxiRequest[], RepositoryError>;
}>;

// Write side — always saves state + events atomically
type TaxiRequestStore = Readonly<{
  save: (
    state: TaxiRequest,
    events: readonly TaxiRequestEvent[],
  ) => ResultAsync<void, RepositoryError>;
}>;
```

Key details:
- `Readonly<{}>` prevents mutation of the port itself
- `readonly` array prevents event list mutation
- `ResultAsync` wraps the async Result (from neverthrow/byethrow)
- The `save` method accepts the full union; callers pass the specific state

### Python: Protocol with async methods

Python uses `typing.Protocol` for structural (duck) typing. Repository methods
include `expected_version` for optimistic concurrency and `idempotency_key` for
at-least-once delivery safety.

```python
from typing import Protocol


class TaxiRequestResolver(Protocol):
    async def find_by_id(
        self, request_id: RequestId,
    ) -> TaxiRequest | None: ...

    async def find_active_by_passenger(
        self, passenger_id: PassengerId,
    ) -> tuple[TaxiRequest, ...]: ...


class TaxiRequestStore(Protocol):
    async def save(
        self,
        state: TaxiRequest,
        events: tuple[DomainEvent, ...],
        expected_version: int,
        idempotency_key: str,
    ) -> None: ...
```

Key details:
- `Protocol` gives structural typing — no inheritance required in implementations
- `tuple[...]` for immutable event sequences (not `list`)
- `expected_version` enables optimistic locking at the protocol level
- Raises domain-specific exceptions (not generic ones) on conflict

### Rust: async_trait with borrowed references

Rust uses a trait with `async_trait` (until async fn in traits stabilizes fully).
Domain types are passed by reference to avoid unnecessary cloning.

```rust
use async_trait::async_trait;

#[async_trait]
pub trait TaxiRequestResolver: Send + Sync {
    async fn find_by_id(
        &self,
        request_id: &RequestId,
    ) -> Result<Option<TaxiRequest>, RepositoryError>;

    async fn find_active_by_passenger(
        &self,
        passenger_id: &PassengerId,
    ) -> Result<Vec<TaxiRequest>, RepositoryError>;
}

#[async_trait]
pub trait TaxiRequestStore: Send + Sync {
    async fn save(
        &self,
        state: &TaxiRequest,
        events: &[TaxiRequestEvent],
    ) -> Result<(), RepositoryError>;
}
```

Key details:
- `Send + Sync` bounds are required for async runtime compatibility
- `&self` — the repository is shared; internal mutability (e.g., connection pool)
  is managed via `Arc<Mutex<>>` or pool types
- `&[TaxiRequestEvent]` — borrowed slice avoids cloning events
- `Option<TaxiRequest>` for nullable lookups (not a sentinel value)

### Scala: Higher-kinded trait with F[_]

Scala abstracts over the effect type using `F[_]`, allowing the same port
definition to work with `IO`, `Task`, `Future`, or test stubs.

```scala
trait TaxiRequestResolver[F[_]]:
  def findById(requestId: RequestId): F[Option[TaxiRequest]]
  def findActiveByPassenger(passengerId: PassengerId): F[List[TaxiRequest]]

trait TaxiRequestStore[F[_]]:
  def save(
    state: TaxiRequest,
    events: List[TaxiRequestEvent],
  ): F[Unit]
```

Key details:
- `F[_]` is typically `IO` (Cats Effect) or `Task` (ZIO) in production
- `Option[TaxiRequest]` for nullable lookups
- `List[TaxiRequestEvent]` — immutable by default in Scala
- State-specific save methods (e.g., `saveAssigned`, `saveCompleted`) can
  replace the generic `save` for additional type safety

---

## Domain Event Model Patterns

### TypeScript: Generic type with string discriminant

```typescript
type DomainEvent<TName extends string, TPayload> = Readonly<{
  eventId: EventId;
  eventName: TName;
  occurredAt: Date;
  payload: TPayload;
}>;

type DriverAssigned = DomainEvent<
  "DriverAssigned",
  Readonly<{ requestId: RequestId; driverId: DriverId }>
>;

type TripStarted = DomainEvent<
  "TripStarted",
  Readonly<{ requestId: RequestId; startedAt: Date }>
>;

type TaxiRequestEvent = DriverAssigned | TripStarted | TripCompleted | RequestCancelled;
```

The `eventName` string literal serves as the discriminant for the union.

### Python: Frozen Pydantic models with Literal discriminant

```python
from pydantic import ConfigDict, Field
from typing import Annotated, Literal


class DriverAssigned(DomainModel):
    event_name: Literal["driver_assigned"] = "driver_assigned"
    event_id: EventId
    occurred_at: datetime
    request_id: RequestId
    driver_id: DriverId


class TripStarted(DomainModel):
    event_name: Literal["trip_started"] = "trip_started"
    event_id: EventId
    occurred_at: datetime
    request_id: RequestId
    started_at: datetime


TaxiRequestEvent = Annotated[
    DriverAssigned | TripStarted | TripCompleted | RequestCancelled,
    Field(discriminator="event_name"),
]
```

`DomainModel` is a project base class with `ConfigDict(frozen=True)`.

### Rust: Enum with struct-like variants

```rust
use chrono::{DateTime, Utc};

#[derive(Debug, Clone)]
pub enum TaxiRequestEvent {
    DriverAssigned {
        event_id: EventId,
        occurred_at: DateTime<Utc>,
        request_id: RequestId,
        driver_id: DriverId,
    },
    TripStarted {
        event_id: EventId,
        occurred_at: DateTime<Utc>,
        request_id: RequestId,
        started_at: DateTime<Utc>,
    },
    TripCompleted {
        event_id: EventId,
        occurred_at: DateTime<Utc>,
        request_id: RequestId,
        fare: Money,
    },
    RequestCancelled {
        event_id: EventId,
        occurred_at: DateTime<Utc>,
        request_id: RequestId,
        reason: CancelReason,
    },
}
```

No explicit discriminant field — the enum variant name IS the discriminant. The
compiler enforces exhaustive matching.

### Scala: Enum ADT with case classes

```scala
import java.time.Instant

enum TaxiRequestEvent:
  case DriverAssigned(
    eventId:    EventId,
    occurredAt: Instant,
    requestId:  RequestId,
    driverId:   DriverId,
  )
  case TripStarted(
    eventId:    EventId,
    occurredAt: Instant,
    requestId:  RequestId,
    startedAt:  Instant,
  )
  case TripCompleted(
    eventId:    EventId,
    occurredAt: Instant,
    requestId:  RequestId,
    fare:       Money,
  )
  case RequestCancelled(
    eventId:    EventId,
    occurredAt: Instant,
    requestId:  RequestId,
    reason:     CancelReason,
  )
```

Like Rust, the case name is the discriminant. Exhaustive matching is enforced
by the compiler (with warnings).

---

## Transition Outcome Types

The transition outcome bundles the new state with the events it produced.

### TypeScript

TS typically returns the new state directly from the transition function. Events
are created in the use case layer, not in the transition itself.

```typescript
// Transition returns only the new state
const TaxiRequest = {
  assignDriver: (waiting: Waiting, driverId: DriverId, now: Date): EnRoute => ({
    kind: "EnRoute",
    requestId: waiting.requestId,
    passengerId: waiting.passengerId,
    driverId,
    assignedAt: now,
  }),
} as const;

// Use case creates events and persists atomically
const assignDriverUseCase = async (
  cmd: AssignDriverCommand,
  resolver: TaxiRequestResolver,
  store: TaxiRequestStore,
  clock: Clock,
): Promise<Result<void, AssignDriverError>> => {
  const request = await resolver.findById(cmd.requestId);
  if (request?.kind !== "Waiting") return err(notWaiting());

  const now = clock.now();
  const newState = TaxiRequest.assignDriver(request, cmd.driverId, now);
  const event: DriverAssigned = {
    eventId: EventId.generate(),
    eventName: "DriverAssigned",
    occurredAt: now,
    payload: { requestId: request.requestId, driverId: cmd.driverId },
  };

  return store.save(newState, [event]);
};
```

### Python

Python uses a `TransitionOutcome` generic frozen model to bundle state and
events together from the transition function itself.

```python
from dataclasses import dataclass
from typing import Generic, TypeVar

TState = TypeVar("TState")
TEvent = TypeVar("TEvent")


@dataclass(frozen=True, slots=True)
class TransitionOutcome(Generic[TState, TEvent]):
    state: TState
    events: tuple[TEvent, ...]


def assign_driver(
    waiting: Waiting,
    driver_id: DriverId,
    now: datetime,
) -> TransitionOutcome[EnRoute, TaxiRequestEvent]:
    en_route = EnRoute(
        kind="en_route",
        request_id=waiting.request_id,
        passenger_id=waiting.passenger_id,
        driver_id=driver_id,
        assigned_at=now,
    )
    event = DriverAssigned(
        event_id=EventId.generate(),
        occurred_at=now,
        request_id=waiting.request_id,
        driver_id=driver_id,
    )
    return TransitionOutcome(state=en_route, events=(event,))
```

### Rust

Rust uses a `Transition` struct generic over state. Events are generated inside
the transition, returned alongside the new state.

```rust
pub struct Transition<TState> {
    pub state: TState,
    pub events: Vec<TaxiRequestEvent>,
}

impl WaitingRequest {
    pub fn assign_driver(
        self,
        driver_id: DriverId,
        now: DateTime<Utc>,
    ) -> Result<Transition<EnRouteRequest>, DomainError> {
        let en_route = EnRouteRequest::new(
            self.request_id().clone(),
            self.passenger_id().clone(),
            driver_id.clone(),
            now,
        );
        let event = TaxiRequestEvent::DriverAssigned {
            event_id: EventId::generate(),
            occurred_at: now,
            request_id: self.request_id().clone(),
            driver_id,
        };
        Ok(Transition {
            state: en_route,
            events: vec![event],
        })
    }
}
```

Note that `self` is consumed (moved), not borrowed. The `WaitingRequest` no
longer exists after the transition — the type system enforces this.

### Scala

Scala uses a `Transition` case class generic over state and event.

```scala
final case class Transition[TState, TEvent](
  state:  TState,
  events: List[TEvent],
)

object WaitingRequest:
  def assignDriver(
    waiting:  WaitingRequest,
    driverId: DriverId,
    now:      Instant,
  ): Either[DomainError, Transition[EnRouteRequest, TaxiRequestEvent]] =
    val enRoute = EnRouteRequest(
      requestId   = waiting.requestId,
      passengerId = waiting.passengerId,
      driverId    = driverId,
      assignedAt  = now,
    )
    val event = TaxiRequestEvent.DriverAssigned(
      eventId    = EventId.generate(),
      occurredAt = now,
      requestId  = waiting.requestId,
      driverId   = driverId,
    )
    Right(Transition(state = enRoute, events = List(event)))
```

---

## Atomic Persist Principle

The same rule applies in every language: **state and events are persisted in a
single transaction**. The repository port enforces this structurally by
accepting both as parameters.

```
save(newState, events)   // single call = single transaction
```

Never do:
```
save(newState)           // state persisted
publishEvents(events)    // <-- crash here = inconsistency
```

### Implementation guidance per language

| Language | Transaction mechanism | Typical pattern |
| --- | --- | --- |
| TypeScript | DB transaction wrapper (e.g., Prisma `$transaction`, Drizzle transaction) | `await db.$transaction(async (tx) => { await tx.requests.update(...); await tx.events.createMany(...); })` |
| Python | SQLAlchemy `async_session.begin()` or framework transaction context | `async with session.begin(): session.add(request_row); session.add_all(event_rows)` |
| Rust | `sqlx::Transaction` or diesel transaction | `let mut tx = pool.begin().await?; sqlx::query!(...).execute(&mut *tx).await?; tx.commit().await?;` |
| Scala | `Transactor[IO]` (doobie) or ZIO transaction | `(updateRequest ++ insertEvents).transact(xa)` |

The outbox pattern (persisting events to a local table, then publishing
asynchronously) is the recommended approach for cross-service event delivery.
The outbox write is inside the same transaction as the state write.

---

## Key Porting Notes

1. **TS splits Resolver/Store by ISP.** When porting to Python, maintain the
   split using two separate `Protocol` classes. Rust and Scala can use two
   separate traits. Do not merge them into a single "repository" trait unless
   the target codebase's convention demands it.

2. **Python uses Protocol for structural typing.** TS `type` with function
   properties achieves the same structural typing. Rust's `trait` and Scala's
   `trait` are nominal but serve the same port abstraction role.

3. **Event generation location varies.** TS tends to create events in the use
   case layer; Python, Rust, and Scala tend to bundle events with the
   transition via `TransitionOutcome`/`Transition`. When porting from TS to
   another language, move event creation into the transition function. When
   porting TO TS, consider keeping events in the use case for consistency with
   the TS kamae idiom.

4. **Rust consumes `self` in transitions.** This is a move, not a copy. The old
   state is gone after the transition. Other languages rely on immutability
   conventions (TS `Readonly`, Python `frozen`, Scala `val`) but do not
   destroy the old reference. When porting TO Rust, ensure transitions take
   `self` (not `&self`) to get the compile-time linearity guarantee.

5. **Scala's `F[_]`** abstracts over the effect type. When porting TO Scala,
   introduce the `F[_]` parameter. When porting FROM Scala, replace `F[_]`
   with the concrete async mechanism: `ResultAsync` in TS, `async def` in
   Python, `async fn` in Rust.

6. **Optimistic concurrency.** Python's protocol includes `expected_version`
   explicitly. The other languages may handle it differently (e.g., in the
   repository implementation rather than the port signature). When porting,
   decide whether version tracking belongs in the port contract or the
   implementation — but it must exist somewhere.
