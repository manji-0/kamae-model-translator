# Serialization Compatibility

When to read: Ensuring JSON output from one kamae language can be parsed
correctly by another kamae language across a service boundary.

---

## Field presence and absence

Languages differ in how they serialize `None`/`null`/`Option::None` for
optional fields. Some include the key with a `null` value; others omit the key
entirely.

| Language   | Default behavior                          | How to change                            |
| ---------- | ----------------------------------------- | ---------------------------------------- |
| TypeScript | `JSON.stringify` includes `null` values   | Filter nulls manually or use a replacer  |
| Python     | `model_dump()` includes `None` values     | Use `exclude_none=True` for absent       |
| Rust       | serde includes `null` for `Option::None`  | `#[serde(skip_serializing_if = "Option::is_none")]` |
| Scala      | Circe drops `None` for `Option` fields    | Use custom encoder to include `null`     |

### Recommendation

Document per-field whether `null` or absent is the canonical representation.
Receivers must handle both forms:

```
// These two payloads should be treated identically by all receivers:
{ "driver_id": "d-001", "note": null }
{ "driver_id": "d-001" }
```

**TypeScript (receiver):**

```typescript
import { z } from "zod";

const WireDto = z.object({
  driver_id: z.string(),
  note: z.string().nullish(), // accepts null, undefined, or absent
});
```

**Python (receiver):**

```python
from pydantic import BaseModel

class WireDto(BaseModel):
    driver_id: str
    note: str | None = None  # accepts null or absent
```

**Rust (receiver):**

```rust
#[derive(Debug, Deserialize)]
pub struct WireDto {
    pub driver_id: String,
    #[serde(default)]
    pub note: Option<String>, // accepts null or absent
}
```

**Scala (receiver):**

```scala
case class WireDto(
  driverId: String,
  note: Option[String] = None // Circe handles null and absent
)
```

---

## Type coercion risks

JSON is loosely typed. A field sent as `"42"` (string) might be accepted as
`42` (integer) by a lenient parser, causing silent data corruption.

| Language   | Default strictness                         | Config                                   |
| ---------- | ------------------------------------------ | ---------------------------------------- |
| TypeScript | Zod rejects by default (strict)            | `.coerce.number()` opts in to leniency   |
| Python     | Pydantic v2 coerces by default (lenient)   | `model_config = ConfigDict(strict=True)` |
| Rust       | serde rejects by default (strict)          | Use `deserialize_with` for leniency      |
| Scala      | Circe rejects by default (strict)          | Custom decoder for leniency              |

### Recommendation

Use strict parsing on all receivers. Wire DTOs are boundary code; they should
reject anything that does not match the expected type exactly.

```python
# Python: enforce strict mode on all wire DTOs
from pydantic import BaseModel, ConfigDict

class WireDto(BaseModel):
    model_config = ConfigDict(strict=True)

    passenger_id: str
    fare_cents: int  # rejects "1500" (string)
```

---

## Extra/unknown fields

When a sender adds a new field that the receiver does not yet know about, the
receiver must decide: reject or ignore.

| Language   | Default for unknown fields    | How to change                           |
| ---------- | ----------------------------- | --------------------------------------- |
| TypeScript | Zod strips unknown fields     | `.passthrough()` to keep them           |
| Python     | Pydantic ignores extras       | `extra="forbid"` to reject              |
| Rust       | serde ignores unknown fields  | `#[serde(deny_unknown_fields)]` to reject |
| Scala      | Circe ignores unknown fields  | Custom decoder to reject                |

### Recommendation

Choose a policy and apply it consistently across all services:

- **Fail-fast (strict):** Reject unknown fields. Catches schema drift early.
  Best for tightly-coupled services deployed together.
- **Tolerant (lenient):** Ignore unknown fields. Supports independent
  deployment where the sender upgrades before the receiver. Best for
  loosely-coupled services and event-driven architectures.

Document the chosen policy in the service contract. For event-driven systems,
tolerant mode is strongly preferred -- it allows schema evolution without
coordinated deployments (see `schema-evolution.md`).

---

## Numeric precision

JSON has a single `number` type with no integer/float distinction. This causes
real problems:

| Risk                           | Example                                   |
| ------------------------------ | ----------------------------------------- |
| Floating point drift           | `0.1 + 0.2` != `0.3`                     |
| Large integer truncation       | `2^53 + 1` loses precision in JavaScript  |
| Currency rounding              | `19.99 * 100` may not equal `1999`        |

### Recommendations

1. **Money:** Use integer cents (or smallest currency unit). `fare_cents: 1999`
   not `fare: 19.99`.
2. **Large integers (>2^53):** Use string representation on the wire.
   ```json
   { "snowflake_id": "1234567890123456789" }
   ```
3. **Decimal values:** Use string representation and parse with a decimal
   library on each side.
   ```json
   { "latitude": "35.6762", "longitude": "139.6503" }
   ```

---

## Date and time

### Wire format

All date/time values on the wire use **ISO 8601 UTC strings**:

```
"2024-03-15T10:30:00Z"
"2024-03-15T10:30:00.000Z"
```

Do not use Unix timestamps (ambiguous seconds vs milliseconds). Do not use
local time without offset.

### Per-language parsing

**TypeScript:**

```typescript
import { z } from "zod";

const WireDto = z.object({
  created_at: z.string().datetime(), // validates ISO 8601
});

// Convert to Date in the domain boundary
function toDomain(wire: WireOutput): DomainModel {
  return { createdAt: new Date(wire.created_at) };
}
```

**Python:**

```python
from datetime import datetime
from pydantic import BaseModel

class WireDto(BaseModel):
    created_at: datetime  # Pydantic auto-parses ISO 8601

# Or use strict string + manual parse:
class StrictWireDto(BaseModel):
    model_config = ConfigDict(strict=True)
    created_at: str  # validate format manually

    def to_domain(self) -> DomainModel:
        return DomainModel(
            created_at=datetime.fromisoformat(self.created_at),
        )
```

**Rust:**

```rust
use chrono::{DateTime, Utc};
use serde::Deserialize;

#[derive(Debug, Deserialize)]
pub struct WireDto {
    pub created_at: DateTime<Utc>, // serde + chrono handles ISO 8601
}
```

**Scala:**

```scala
import java.time.Instant

case class WireDto(createdAt: Instant)

// With Circe:
import io.circe.Decoder
implicit val instantDecoder: Decoder[Instant] =
  Decoder.decodeString.emap { s =>
    scala.util.Try(Instant.parse(s)).toEither.left.map(_.getMessage)
  }
```

### Time zone handling

- Store and transmit in UTC always.
- Convert to local time only at the presentation layer.
- If a business rule depends on local time (e.g., "driver shift ends at 22:00
  JST"), transmit the time zone identifier separately:
  ```json
  { "shift_end": "2024-03-15T13:00:00Z", "time_zone": "Asia/Tokyo" }
  ```

---

## Checklist

Before deploying a new cross-language wire format:

- [ ] Document field presence convention (null vs absent) for each optional field
- [ ] Enable strict parsing on all receiver DTOs
- [ ] Choose and document extra-fields policy (fail-fast or tolerant)
- [ ] Use integer cents for monetary values
- [ ] Use string representation for integers larger than 2^53
- [ ] Use ISO 8601 UTC for all date/time fields
- [ ] Add contract tests for the wire shape (see `contract-testing.md`)
