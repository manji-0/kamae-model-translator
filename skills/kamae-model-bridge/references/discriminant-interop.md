# Discriminant Interop

When to read: Aligning `kind` discriminant values across services written in
different kamae languages (TypeScript, Python, Rust, Scala).

---

## The problem

Each kamae language has its own idiomatic convention for discriminant values
in domain code:

| Language   | Domain convention        | Example values              |
| ---------- | ------------------------ | --------------------------- |
| TypeScript | PascalCase               | `"Waiting"`, `"EnRoute"`   |
| Python     | snake_case               | `"waiting"`, `"en_route"`  |
| Rust       | Variant names (no string)| `Waiting`, `EnRoute`        |
| Scala      | Case names (no string)   | `Waiting`, `EnRoute`        |

When these services exchange discriminated unions over the wire, they must
agree on a single string representation for each discriminant value.

## Wire convention

**All discriminant values on the wire use snake_case.**

```
"waiting"   "en_route"   "in_trip"   "completed"   "cancelled"
```

Each language converts between its domain convention and the wire convention
at the DTO boundary. Domain code never sees wire strings; wire DTOs never see
domain enums.

---

## Per-language configuration

### TypeScript

Outbound: map PascalCase domain values to snake_case wire values.
Inbound: map snake_case wire values back to PascalCase domain values.

```typescript
// --- Discriminant mapping ---

const domainToWire = {
  Waiting: "waiting",
  EnRoute: "en_route",
  InTrip: "in_trip",
  Completed: "completed",
  Cancelled: "cancelled",
} as const;

const wireToDomain = Object.fromEntries(
  Object.entries(domainToWire).map(([k, v]) => [v, k]),
) as Record<string, keyof typeof domainToWire>;

// --- Outbound DTO (domain -> wire) ---

function toWireKind(domainKind: string): string {
  const wire = domainToWire[domainKind as keyof typeof domainToWire];
  if (!wire) throw new Error(`Unknown domain kind: ${domainKind}`);
  return wire;
}

// --- Inbound DTO (wire -> domain) with Zod ---

import { z } from "zod";

const WireKindSchema = z
  .enum(["waiting", "en_route", "in_trip", "completed", "cancelled"])
  .transform((val) => {
    const domain = wireToDomain[val];
    if (!domain) throw new Error(`Unknown wire kind: ${val}`);
    return domain;
  });
```

### Python

Python kamae domain models already use snake_case discriminant values, so the
wire DTO `kind` field needs no transformation.

```python
from typing import Literal

# Domain model (already snake_case)
class Waiting(FrozenModel):
    kind: Literal["waiting"] = "waiting"

# Wire DTO (same value, no mapping needed)
class WaitingWireDto(BaseModel):
    kind: Literal["waiting"]
```

If the domain uses a different convention (e.g., uppercase), add a mapping in
the DTO conversion functions, not in the model itself.

### Rust

Use `#[serde(rename_all = "snake_case")]` on the wire enum, or explicit
`#[serde(rename = "...")]` per variant when the automatic conversion does not
produce the desired wire string.

```rust
use serde::{Deserialize, Serialize};

/// Wire-format enum for taxi request state.
/// Domain code uses `TaxiRequestState`; this is the DTO boundary.
#[derive(Debug, Serialize, Deserialize)]
#[serde(tag = "kind", rename_all = "snake_case")]
pub enum TaxiRequestStateWireDto {
    Waiting {
        passenger_id: String,
    },
    EnRoute {
        passenger_id: String,
        driver_id: String,
    },
    InTrip {
        passenger_id: String,
        driver_id: String,
        started_at: String,
    },
    Completed {
        passenger_id: String,
        driver_id: String,
        completed_at: String,
    },
    Cancelled {
        passenger_id: String,
        reason: String,
    },
}
```

`rename_all = "snake_case"` converts `EnRoute` to `"en_route"` and `InTrip`
to `"in_trip"` automatically. For non-standard conversions, use explicit
renames:

```rust
#[derive(Debug, Serialize, Deserialize)]
#[serde(tag = "kind")]
pub enum CustomWireDto {
    #[serde(rename = "waiting")]
    Waiting { /* ... */ },
    #[serde(rename = "en_route")]
    EnRoute { /* ... */ },
}
```

### Scala

Map case names to snake_case in the Circe codec (or Play JSON, etc.).

```scala
import io.circe.{Decoder, Encoder, Json}
import io.circe.syntax._

sealed trait TaxiRequestStateWireDto

object TaxiRequestStateWireDto {
  case class Waiting(passengerId: String) extends TaxiRequestStateWireDto
  case class EnRoute(passengerId: String, driverId: String)
      extends TaxiRequestStateWireDto
  case class InTrip(passengerId: String, driverId: String, startedAt: String)
      extends TaxiRequestStateWireDto
  case class Completed(passengerId: String, driverId: String, completedAt: String)
      extends TaxiRequestStateWireDto
  case class Cancelled(passengerId: String, reason: String)
      extends TaxiRequestStateWireDto

  // Snake_case discriminant mapping
  private val kindMap: Map[String, String] = Map(
    "Waiting"  -> "waiting",
    "EnRoute"  -> "en_route",
    "InTrip"   -> "in_trip",
    "Completed" -> "completed",
    "Cancelled" -> "cancelled"
  )
  private val reverseKindMap: Map[String, String] =
    kindMap.map { case (k, v) => (v, k) }

  implicit val encoder: Encoder[TaxiRequestStateWireDto] =
    Encoder.instance { dto =>
      val (caseName, fields) = dto match {
        case Waiting(pid) =>
          ("Waiting", Json.obj("passenger_id" -> pid.asJson))
        case EnRoute(pid, did) =>
          ("EnRoute", Json.obj(
            "passenger_id" -> pid.asJson,
            "driver_id"    -> did.asJson
          ))
        case InTrip(pid, did, at) =>
          ("InTrip", Json.obj(
            "passenger_id" -> pid.asJson,
            "driver_id"    -> did.asJson,
            "started_at"   -> at.asJson
          ))
        case Completed(pid, did, at) =>
          ("Completed", Json.obj(
            "passenger_id" -> pid.asJson,
            "driver_id"    -> did.asJson,
            "completed_at" -> at.asJson
          ))
        case Cancelled(pid, reason) =>
          ("Cancelled", Json.obj(
            "passenger_id" -> pid.asJson,
            "reason"       -> reason.asJson
          ))
      }
      val wireKind = kindMap.getOrElse(caseName, caseName)
      fields.deepMerge(Json.obj("kind" -> wireKind.asJson))
    }

  implicit val decoder: Decoder[TaxiRequestStateWireDto] =
    Decoder.instance { cursor =>
      cursor.get[String]("kind").flatMap { wireKind =>
        reverseKindMap.getOrElse(wireKind, wireKind) match {
          case "Waiting" =>
            cursor.get[String]("passenger_id").map(Waiting.apply)
          case "EnRoute" =>
            for {
              pid <- cursor.get[String]("passenger_id")
              did <- cursor.get[String]("driver_id")
            } yield EnRoute(pid, did)
          case "InTrip" =>
            for {
              pid <- cursor.get[String]("passenger_id")
              did <- cursor.get[String]("driver_id")
              at  <- cursor.get[String]("started_at")
            } yield InTrip(pid, did, at)
          case "Completed" =>
            for {
              pid <- cursor.get[String]("passenger_id")
              did <- cursor.get[String]("driver_id")
              at  <- cursor.get[String]("completed_at")
            } yield Completed(pid, did, at)
          case "Cancelled" =>
            for {
              pid    <- cursor.get[String]("passenger_id")
              reason <- cursor.get[String]("reason")
            } yield Cancelled(pid, reason)
          case other =>
            Left(io.circe.DecodingFailure(
              s"Unknown kind: $other", cursor.history
            ))
        }
      }
    }
}
```

---

## Discriminant mapping table

Taxi-request example with all states:

| Domain (TS)  | Domain (Py)  | Domain (Rust) | Domain (Scala) | Wire value    |
| ------------ | ------------ | -------------- | --------------- | ------------- |
| `"Waiting"`  | `"waiting"`  | `Waiting`      | `Waiting`       | `"waiting"`   |
| `"EnRoute"`  | `"en_route"` | `EnRoute`      | `EnRoute`       | `"en_route"`  |
| `"InTrip"`   | `"in_trip"`  | `InTrip`       | `InTrip`        | `"in_trip"`   |
| `"Completed"`| `"completed"`| `Completed`    | `Completed`     | `"completed"` |
| `"Cancelled"`| `"cancelled"`| `Cancelled`    | `Cancelled`     | `"cancelled"` |

---

## Event name interop

The same snake_case wire convention applies to the `event_name` discriminant
in domain event envelopes. TypeScript domain events typically use PascalCase
event names (`"DriverAssigned"`), but the wire format requires snake_case
(`"driver_assigned"`).

### Conversion rules

| Domain (TS)          | Domain (Py)          | Wire `event_name`      |
| -------------------- | -------------------- | ---------------------- |
| `"DriverAssigned"`   | `"driver_assigned"`  | `"driver_assigned"`    |
| `"TripStarted"`      | `"trip_started"`     | `"trip_started"`       |
| `"TripCompleted"`    | `"trip_completed"`   | `"trip_completed"`     |
| `"TripCancelled"`    | `"trip_cancelled"`   | `"trip_cancelled"`     |

### TypeScript

Convert PascalCase event names to snake_case at the outbound DTO boundary,
exactly as with `kind` values:

```typescript
// PascalCase -> snake_case for event names
const eventNameToWire = (name: string): string =>
  name.replace(/([a-z0-9])([A-Z])/g, "$1_$2").toLowerCase();

// Example: "DriverAssigned" -> "driver_assigned"
const wireDto = {
  event_name: eventNameToWire(event.kind),
  // ...
};
```

Inbound, convert snake_case back to PascalCase:

```typescript
const wireToEventName = (wire: string): string =>
  wire.replace(/_([a-z])/g, (_, c) => c.toUpperCase())
      .replace(/^[a-z]/, (c) => c.toUpperCase());

// Example: "driver_assigned" -> "DriverAssigned"
```

### Python, Rust, Scala

Python and Rust domain events already use snake_case names, so no conversion
is needed. Scala maps PascalCase case class names to snake_case in the
encoder, just as it does for `kind` values.

---

## Deviating from snake_case

If the project already uses a different wire convention (e.g., PascalCase on
the wire because all consumers are TypeScript), document the deviation
explicitly:

1. State the wire convention in the service contract (README, OpenAPI spec, or
   shared schema file).
2. Ensure **all** services -- including any future non-TS service -- agree on
   the same convention.
3. Each language's DTO boundary converts to/from the agreed wire convention,
   not the kamae default.
4. Add a contract test fixture that pins the actual wire values (see
   `contract-testing.md`).

The key invariant: every service produces and consumes the **same** string for
each discriminant value. The string itself matters less than the agreement.
