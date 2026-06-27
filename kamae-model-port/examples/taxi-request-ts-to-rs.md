# Worked Example: Taxi Request -- TypeScript to Rust

This walks through porting the taxi-request domain model from TypeScript to
Rust, step by step. Each section shows the TypeScript source and the Rust
target, with notes on what changed and why.

---

## Step 1: Port ID Types

Branded types in TypeScript become newtypes with private fields and validated
constructors in Rust.

### TypeScript Source

```typescript
const RequestIdBrand = Symbol();
const RequestIdSchema = z.string().uuid().brand<typeof RequestIdBrand>();
type RequestId = z.infer<typeof RequestIdSchema>;

const RequestId = {
  parse: (raw: string): Result<RequestId, BrandError> =>
    schemaResult(RequestIdSchema, raw),
  generate: (): RequestId =>
    RequestIdSchema.parse(crypto.randomUUID()),
} as const;
```

The `Symbol()` brand makes `RequestId` structurally distinct from other string
types. The Companion Object provides `parse` (validated) and `generate`
(infallible) entry points.

### Rust Target

```rust
use uuid::Uuid;

#[derive(Debug, Clone, PartialEq, Eq, Hash)]
pub struct RequestId(Uuid);

impl RequestId {
    pub fn new(value: Uuid) -> Result<Self, IdError> {
        if value.is_nil() {
            return Err(IdError::NilId("RequestId"));
        }
        Ok(Self(value))
    }

    pub fn generate() -> Self {
        Self(Uuid::new_v4())
    }

    pub fn value(&self) -> &Uuid {
        &self.0
    }
}
```

**What changed:**
- `Symbol()` brand becomes a newtype wrapper `struct RequestId(Uuid)`. The
  tuple field is private (no `pub`), so external code cannot construct it
  directly.
- `z.string().uuid()` validation is replaced by using the `Uuid` type directly
  (parsing happens at the boundary via serde).
- `safeParse` becomes `new()` returning `Result<Self, IdError>`.
- `generate()` remains infallible since `Uuid::new_v4()` cannot fail.
- `value()` getter exposes the inner value by reference. TS did not need this
  because the brand is structural, not encapsulated.

Apply the same pattern for `PassengerId`, `DriverId`, and `EventId`.

---

## Step 2: Port State Types

`Readonly<{}>` type aliases with `kind` discriminant become structs with
private fields. The union becomes an enum.

### TypeScript Source

```typescript
type Waiting = Readonly<{
  kind: "Waiting";
  requestId: RequestId;
  passengerId: PassengerId;
  pickup: Location;
  destination: Location;
  createdAt: Date;
}>;

type EnRoute = Readonly<{
  kind: "EnRoute";
  requestId: RequestId;
  passengerId: PassengerId;
  driverId: DriverId;
  pickup: Location;
  destination: Location;
  assignedAt: Date;
}>;

type InTrip = Readonly<{
  kind: "InTrip";
  requestId: RequestId;
  passengerId: PassengerId;
  driverId: DriverId;
  pickup: Location;
  destination: Location;
  startedAt: Date;
}>;

type Completed = Readonly<{
  kind: "Completed";
  requestId: RequestId;
  passengerId: PassengerId;
  driverId: DriverId;
  fare: Money;
  completedAt: Date;
}>;

type Cancelled = Readonly<{
  kind: "Cancelled";
  requestId: RequestId;
  passengerId: PassengerId;
  reason: string;
  cancelledAt: Date;
}>;

type TaxiRequest = Waiting | EnRoute | InTrip | Completed | Cancelled;
```

### Rust Target

```rust
use chrono::{DateTime, Utc};

#[derive(Debug, Clone)]
pub struct WaitingRequest {
    request_id: RequestId,
    passenger_id: PassengerId,
    pickup: Location,
    destination: Location,
    created_at: DateTime<Utc>,
}

#[derive(Debug, Clone)]
pub struct EnRouteRequest {
    request_id: RequestId,
    passenger_id: PassengerId,
    driver_id: DriverId,
    pickup: Location,
    destination: Location,
    assigned_at: DateTime<Utc>,
}

#[derive(Debug, Clone)]
pub struct InTripRequest {
    request_id: RequestId,
    passenger_id: PassengerId,
    driver_id: DriverId,
    pickup: Location,
    destination: Location,
    started_at: DateTime<Utc>,
}

#[derive(Debug, Clone)]
pub struct CompletedRequest {
    request_id: RequestId,
    passenger_id: PassengerId,
    driver_id: DriverId,
    fare: Money,
    completed_at: DateTime<Utc>,
}

#[derive(Debug, Clone)]
pub struct CancelledRequest {
    request_id: RequestId,
    passenger_id: PassengerId,
    reason: String,
    cancelled_at: DateTime<Utc>,
}

#[derive(Debug, Clone)]
pub enum TaxiRequest {
    Waiting(WaitingRequest),
    EnRoute(EnRouteRequest),
    InTrip(InTripRequest),
    Completed(CompletedRequest),
    Cancelled(CancelledRequest),
}
```

**What changed:**
- Each TS type alias becomes a Rust struct. Struct names gain a `Request`
  suffix to avoid colliding with the enum variant names (Rust convention:
  `WaitingRequest` wraps inside `TaxiRequest::Waiting`).
- The `kind` discriminant field disappears — the enum variant IS the
  discriminant in Rust.
- `Readonly<{}>` becomes private fields (no `pub` on struct fields). Add
  getter methods for each field:

```rust
impl WaitingRequest {
    pub fn request_id(&self) -> &RequestId { &self.request_id }
    pub fn passenger_id(&self) -> &PassengerId { &self.passenger_id }
    pub fn pickup(&self) -> &Location { &self.pickup }
    pub fn destination(&self) -> &Location { &self.destination }
    pub fn created_at(&self) -> &DateTime<Utc> { &self.created_at }
}
```

- `Date` becomes `DateTime<Utc>` from the `chrono` crate.
- `camelCase` field names become `snake_case`.

### Partial union types

TS uses type aliases for subsets of the union:

```typescript
type CancellableRequest = Waiting | EnRoute | InTrip;
```

Rust does not have built-in union subsets. Two options:

**Option A: Dedicated enum** (preferred when the subset is used in signatures)

```rust
pub enum CancellableRequest {
    Waiting(WaitingRequest),
    EnRoute(EnRouteRequest),
    InTrip(InTripRequest),
}

impl TryFrom<TaxiRequest> for CancellableRequest {
    type Error = DomainError;

    fn try_from(request: TaxiRequest) -> Result<Self, Self::Error> {
        match request {
            TaxiRequest::Waiting(w) => Ok(CancellableRequest::Waiting(w)),
            TaxiRequest::EnRoute(e) => Ok(CancellableRequest::EnRoute(e)),
            TaxiRequest::InTrip(t) => Ok(CancellableRequest::InTrip(t)),
            _ => Err(DomainError::NotCancellable),
        }
    }
}
```

**Option B: Trait bound** (when behavior is more important than data shape)

```rust
pub trait Cancellable {
    fn request_id(&self) -> &RequestId;
    fn passenger_id(&self) -> &PassengerId;
}

impl Cancellable for WaitingRequest { /* ... */ }
impl Cancellable for EnRouteRequest { /* ... */ }
impl Cancellable for InTripRequest { /* ... */ }
```

---

## Step 3: Port Transition Functions

Companion Object methods become `impl` methods on the source state struct.
The key difference: Rust transitions consume `self`, destroying the old state.

### TypeScript Source

```typescript
const TaxiRequest = {
  assignDriver: (
    waiting: Waiting,
    driverId: DriverId,
    now: Date,
  ): EnRoute => ({
    kind: "EnRoute",
    requestId: waiting.requestId,
    passengerId: waiting.passengerId,
    driverId,
    pickup: waiting.pickup,
    destination: waiting.destination,
    assignedAt: now,
  }),

  startTrip: (
    enRoute: EnRoute,
    now: Date,
  ): InTrip => ({
    kind: "InTrip",
    requestId: enRoute.requestId,
    passengerId: enRoute.passengerId,
    driverId: enRoute.driverId,
    pickup: enRoute.pickup,
    destination: enRoute.destination,
    startedAt: now,
  }),

  complete: (
    inTrip: InTrip,
    fare: Money,
    now: Date,
  ): Completed => ({
    kind: "Completed",
    requestId: inTrip.requestId,
    passengerId: inTrip.passengerId,
    driverId: inTrip.driverId,
    fare,
    completedAt: now,
  }),

  cancel: (
    request: CancellableRequest,
    reason: string,
    now: Date,
  ): Cancelled => ({
    kind: "Cancelled",
    requestId: request.requestId,
    passengerId: request.passengerId,
    reason,
    cancelledAt: now,
  }),
} as const;
```

### Rust Target

```rust
impl WaitingRequest {
    /// Transition: Waiting -> EnRoute
    /// Consumes self — the WaitingRequest no longer exists after assignment.
    pub fn assign_driver(
        self,
        driver_id: DriverId,
        now: DateTime<Utc>,
    ) -> EnRouteRequest {
        EnRouteRequest {
            request_id: self.request_id,
            passenger_id: self.passenger_id,
            driver_id,
            pickup: self.pickup,
            destination: self.destination,
            assigned_at: now,
        }
    }
}

impl EnRouteRequest {
    /// Transition: EnRoute -> InTrip
    pub fn start_trip(self, now: DateTime<Utc>) -> InTripRequest {
        InTripRequest {
            request_id: self.request_id,
            passenger_id: self.passenger_id,
            driver_id: self.driver_id,
            pickup: self.pickup,
            destination: self.destination,
            started_at: now,
        }
    }
}

impl InTripRequest {
    /// Transition: InTrip -> Completed
    pub fn complete(self, fare: Money, now: DateTime<Utc>) -> CompletedRequest {
        CompletedRequest {
            request_id: self.request_id,
            passenger_id: self.passenger_id,
            driver_id: self.driver_id,
            fare,
            completed_at: now,
        }
    }
}
```

**What changed:**
- Companion Object methods become `impl` blocks on each source state struct.
- Functions consume `self` (by value, not `&self`). This is Rust's ownership
  model enforcing the state machine: after `assign_driver`, the
  `WaitingRequest` is moved and cannot be used again.
- The `kind` field is not set — enum wrapping happens at the call site.
- Field names change to `snake_case`.
- `camelCase` method names become `snake_case`.

**For the `cancel` transition** (which accepts a partial union in TS):

```rust
impl CancellableRequest {
    pub fn cancel(self, reason: String, now: DateTime<Utc>) -> CancelledRequest {
        let (request_id, passenger_id) = match &self {
            CancellableRequest::Waiting(w) => (w.request_id.clone(), w.passenger_id.clone()),
            CancellableRequest::EnRoute(e) => (e.request_id.clone(), e.passenger_id.clone()),
            CancellableRequest::InTrip(t) => (t.request_id.clone(), t.passenger_id.clone()),
        };
        CancelledRequest {
            request_id,
            passenger_id,
            reason,
            cancelled_at: now,
        }
    }
}
```

Or, if using the trait approach:

```rust
pub fn cancel<T: Cancellable>(
    request: T,
    reason: String,
    now: DateTime<Utc>,
) -> CancelledRequest {
    CancelledRequest {
        request_id: request.request_id().clone(),
        passenger_id: request.passenger_id().clone(),
        reason,
        cancelled_at: now,
    }
}
```

---

## Step 4: Port Domain Events

The generic `DomainEvent` type with a string discriminant becomes an enum with
struct-like variants.

### TypeScript Source

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

type TripCompleted = DomainEvent<
  "TripCompleted",
  Readonly<{ requestId: RequestId; fare: Money }>
>;

type RequestCancelled = DomainEvent<
  "RequestCancelled",
  Readonly<{ requestId: RequestId; reason: string }>
>;

type TaxiRequestEvent =
  | DriverAssigned
  | TripStarted
  | TripCompleted
  | RequestCancelled;
```

### Rust Target

```rust
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
        reason: String,
    },
}
```

**What changed:**
- The generic `DomainEvent<TName, TPayload>` wrapper is eliminated. Rust's
  enum variants directly carry the fields — there is no need for a shared
  `eventName` discriminant because pattern matching on the variant serves that
  role.
- `eventId` and `occurredAt` are repeated in each variant. This is idiomatic
  Rust — shared fields on enum variants are accessed via a helper method, not
  a base type:

```rust
impl TaxiRequestEvent {
    pub fn event_id(&self) -> &EventId {
        match self {
            Self::DriverAssigned { event_id, .. } => event_id,
            Self::TripStarted { event_id, .. } => event_id,
            Self::TripCompleted { event_id, .. } => event_id,
            Self::RequestCancelled { event_id, .. } => event_id,
        }
    }

    pub fn occurred_at(&self) -> &DateTime<Utc> {
        match self {
            Self::DriverAssigned { occurred_at, .. } => occurred_at,
            Self::TripStarted { occurred_at, .. } => occurred_at,
            Self::TripCompleted { occurred_at, .. } => occurred_at,
            Self::RequestCancelled { occurred_at, .. } => occurred_at,
        }
    }
}
```

- The `payload` wrapper object is flattened — its fields are promoted to
  top-level variant fields. TS used `payload` to separate metadata from
  content; Rust does not need this separation because the variant structure
  is already clear.

---

## Step 5: Port Transition Outcome

In TS, transitions return a plain state value and events are created in the use
case. In Rust, we bundle state and events together using a `Transition` struct.

### TypeScript Source (use case creates events)

```typescript
// In TS, the transition returns only the new state:
const newState = TaxiRequest.assignDriver(waiting, cmd.driverId, now);

// Events are created separately in the use case:
const event: DriverAssigned = {
  eventId: EventId.generate(),
  eventName: "DriverAssigned",
  occurredAt: now,
  payload: { requestId: waiting.requestId, driverId: cmd.driverId },
};

await store.save(newState, [event]);
```

### Rust Target (transition produces both)

```rust
/// Generic transition outcome — bundles new state with domain events.
pub struct Transition<TState> {
    pub state: TState,
    pub events: Vec<TaxiRequestEvent>,
}

impl WaitingRequest {
    pub fn assign_driver(
        self,
        driver_id: DriverId,
        now: DateTime<Utc>,
    ) -> Transition<EnRouteRequest> {
        let event = TaxiRequestEvent::DriverAssigned {
            event_id: EventId::generate(),
            occurred_at: now,
            request_id: self.request_id.clone(),
            driver_id: driver_id.clone(),
        };

        let state = EnRouteRequest {
            request_id: self.request_id,
            passenger_id: self.passenger_id,
            driver_id,
            pickup: self.pickup,
            destination: self.destination,
            assigned_at: now,
        };

        Transition {
            state,
            events: vec![event],
        }
    }
}
```

**What changed:**
- Event creation moves INTO the transition function. In TS, the use case is
  responsible for creating events; in Rust, the transition itself produces
  them. This is the idiomatic Rust kamae pattern — it keeps event generation
  co-located with the state change that produces it.
- `Transition<TState>` replaces the plain return value. The use case
  destructures it:

```rust
let transition = waiting.assign_driver(cmd.driver_id, now);
store.save(&transition.state, &transition.events).await?;
```

---

## Step 6: Port Error Types and Result

TS discriminated union errors become Rust enums with `thiserror`.

### TypeScript Source

```typescript
type AssignDriverError =
  | Readonly<{ kind: "RequestNotFound" }>
  | Readonly<{ kind: "NotWaiting"; currentState: string }>
  | Readonly<{ kind: "DriverUnavailable"; driverId: DriverId }>;

// Used in the use case:
const assignDriver = (
  cmd: AssignDriverCommand,
): ResultAsync<void, AssignDriverError> => { ... };
```

### Rust Target

```rust
use thiserror::Error;

#[derive(Debug, Error)]
pub enum AssignDriverError {
    #[error("request not found")]
    RequestNotFound,

    #[error("request is not in Waiting state, current state: {current_state}")]
    NotWaiting { current_state: String },

    #[error("driver {driver_id:?} is unavailable")]
    DriverUnavailable { driver_id: DriverId },

    #[error("repository error: {0}")]
    Repository(#[from] RepositoryError),
}

// Used in the use case:
pub async fn assign_driver(
    cmd: AssignDriverCommand,
    resolver: &dyn TaxiRequestResolver,
    store: &dyn TaxiRequestStore,
) -> Result<(), AssignDriverError> {
    let request = resolver
        .find_by_id(&cmd.request_id)
        .await
        .map_err(AssignDriverError::Repository)?
        .ok_or(AssignDriverError::RequestNotFound)?;

    let waiting = match request {
        TaxiRequest::Waiting(w) => w,
        other => return Err(AssignDriverError::NotWaiting {
            current_state: format!("{:?}", other),
        }),
    };

    let transition = waiting.assign_driver(cmd.driver_id, Utc::now());

    store
        .save(&TaxiRequest::EnRoute(transition.state), &transition.events)
        .await
        .map_err(AssignDriverError::Repository)?;

    Ok(())
}
```

**What changed:**
- TS `kind`-discriminated error type becomes a Rust `enum` with `thiserror`
  derive for automatic `Display` implementation.
- `ResultAsync<void, E>` becomes `Result<(), E>` (Rust uses `()` for void,
  `Result` for sync, and the function is `async`).
- `#[from]` attribute on `Repository` variant enables `?` operator to
  automatically convert `RepositoryError` into `AssignDriverError`.
- The `?` operator replaces `andThen`/`map` chaining. Each `?` is an early
  return on error — structurally equivalent to TS's `andThen` but more concise.
- `.ok_or(...)` converts `Option<T>` to `Result<T, E>`, replacing the TS
  pattern of `if (request === null) return err(...)`.

---

## Summary of Structural Differences

| Aspect | TypeScript | Rust |
| --- | --- | --- |
| ID types | `z.brand()` + Symbol | Newtype struct, private field, `new()` |
| Immutability | `Readonly<{}>` | Private fields + getters |
| Discriminant | `kind` string literal field | Enum variant name |
| State transition | Companion Object function, returns new value | `impl` method consuming `self` |
| Events | Created in use case | Created in transition, returned via `Transition<T>` |
| Error type | DU with `kind` | `enum` + `thiserror` |
| Result type | `Result`/`ResultAsync` (library) | `Result<T, E>` (std) |
| Composition | `andThen` / `map` chaining | `?` operator |
| Null handling | `\| null` union | `Option<T>` |
| Partial union | Type alias of subset | Dedicated enum or trait bound |
| Repository port | `type` with function properties | `trait` with `async_trait` |
| Effect handling | `ResultAsync` wraps Promise+Result | `async fn` returns `Result` |
