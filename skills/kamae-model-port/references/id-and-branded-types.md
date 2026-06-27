# ID and Branded Type Mapping

When to read: Porting semantic IDs, branded types, value objects, or
newtype wrappers between languages.

---

## Overview

All kamae languages prevent mix-up of semantically different values that
share the same underlying primitive. The mechanism varies: TypeScript uses
schema brands, Python uses wrapper models, Rust uses newtype structs with
private fields, and Scala uses opaque types. Each approach provides
compile-time or runtime separation (or both), and each has a validated
factory that enforces invariants at the boundary.

---

## 1. ID Patterns

### TypeScript -- `z.brand()` + Symbol + Companion Object

The brand Symbol is unique per type. A Zod schema adds the brand, and a
companion `const` object exposes `schema`, `parse`, and `generate`.

```typescript
// domain/taxi-request/request-id.ts
import { z } from "zod";

const RequestIdBrand = Symbol("RequestId");

const RequestIdSchema = z
  .string()
  .uuid()
  .brand<typeof RequestIdBrand>();

export type RequestId = z.infer<typeof RequestIdSchema>;

export const RequestId = {
  schema: RequestIdSchema,

  parse(value: unknown): RequestId {
    return RequestIdSchema.parse(value);
  },

  generate(): RequestId {
    return RequestIdSchema.parse(crypto.randomUUID());
  },
} as const;
```

Usage:

```typescript
const id: RequestId = RequestId.parse(rawInput);
// const bad: RequestId = "some-string"; // compile error -- not branded
```

### Python -- Frozen wrapper model

A frozen Pydantic model wraps the raw value. Validators enforce format.
For lightweight cases, `NewType` provides static-only separation.

```python
# domain/taxi_request/request_id.py
from uuid import UUID
from pydantic import field_validator
from ..shared.domain_model import DomainModel


class RequestId(DomainModel):
    """Opaque identifier for a taxi request."""
    value: UUID

    @field_validator("value", mode="before")
    @classmethod
    def _parse_uuid(cls, v: object) -> UUID:
        if isinstance(v, str):
            return UUID(v)
        if isinstance(v, UUID):
            return v
        raise ValueError(f"Cannot parse RequestId from {type(v)}")

    def __str__(self) -> str:
        return str(self.value)

    def __hash__(self) -> int:
        return hash(self.value)

    def __eq__(self, other: object) -> bool:
        return isinstance(other, RequestId) and self.value == other.value
```

Lightweight alternative (static-only, no runtime enforcement):

```python
from typing import NewType
from uuid import UUID

RequestId = NewType("RequestId", UUID)
```

Usage:

```python
rid = RequestId(value=uuid4())
# rid = DriverId(value=uuid4())  # type error if function expects RequestId
```

### Rust -- Newtype with private inner field

A tuple struct with a private inner field. The only way to create an
instance is through the validated `new()` constructor.

```rust
// domain/taxi_request/request_id.rs
use serde::{Deserialize, Serialize};
use std::fmt;

#[derive(Clone, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub struct RequestId(String);

impl RequestId {
    pub fn new(value: impl Into<String>) -> Result<Self, IdError> {
        let s = value.into();
        uuid::Uuid::parse_str(&s).map_err(|_| IdError::InvalidUuid(s.clone()))?;
        Ok(Self(s))
    }

    pub fn generate() -> Self {
        Self(uuid::Uuid::new_v4().to_string())
    }

    pub fn as_str(&self) -> &str {
        &self.0
    }
}

impl fmt::Display for RequestId {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{}", self.0)
    }
}

impl fmt::Debug for RequestId {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "RequestId({})", self.0)
    }
}

#[derive(Debug, thiserror::Error)]
pub enum IdError {
    #[error("Invalid UUID: {0}")]
    InvalidUuid(String),
}
```

Usage:

```rust
let id = RequestId::new(raw_input)?;
// let bad: RequestId = RequestId("raw".to_string()); // compile error -- field is private
```

### Scala -- Opaque type

An opaque type alias inside a companion object. The underlying
representation is erased at compile time outside the defining scope.

```scala
// domain/taxirequest/RequestId.scala
package domain.taxirequest

import java.util.UUID

object RequestIds:
  opaque type RequestId = String

  object RequestId:
    def apply(value: String): Either[IdError, RequestId] =
      Either.cond(
        scala.util.Try(UUID.fromString(value)).isSuccess,
        value,
        IdError.InvalidUuid(value),
      )

    def generate(): RequestId =
      UUID.randomUUID().toString

  extension (id: RequestId)
    def value: String = id
    // Note: opaque types inherit the underlying type's toString (here String),
    // so RequestId.toString already returns the underlying string value.
    // Extension methods cannot override inherited methods like toString.
```

Usage:

```scala
import domain.taxirequest.RequestIds.*

val id: RequestId = RequestId("550e8400-e29b-41d4-a716-446655440000") match
  case Right(v) => v
  case Left(e)  => throw new IllegalArgumentException(e.toString)

// val bad: RequestId = "raw-string" // compile error outside RequestIds scope
```

---

## 2. Validation Guarantee Comparison

| Language   | Compile-time guarantee          | Runtime guarantee                  | Bypass risk |
| ---------- | ------------------------------- | ---------------------------------- | ----------- |
| TypeScript | Brand prevents assignment of raw strings | `z.parse()` validates at boundary  | Medium -- brands are structural (erased at runtime); a cast or `as any` bypasses |
| Python     | mypy rejects cross-type assignment (wrapper model) | Pydantic validates on construction | Low for wrapper models; `NewType` is static-only (no runtime check) |
| Rust       | Private field prevents direct construction | `new()` validates                  | Very low -- no way to construct without `new()` from outside the module |
| Scala      | Opaque type prevents assignment outside defining object | `apply()` validates               | Very low -- opaque abstraction is enforced by compiler |

---

## 3. Value Objects

Value objects (Money, DateRange, EmailAddress, etc.) follow the same
pattern but with multiple fields and richer invariants.

### TypeScript

```typescript
// domain/shared/money.ts
import { z } from "zod";

const MoneyBrand = Symbol("Money");

const MoneySchema = z.object({
  amount: z.number().nonnegative(),
  currency: z.enum(["USD", "EUR", "JPY"]),
}).brand<typeof MoneyBrand>();

export type Money = z.infer<typeof MoneySchema>;

export const Money = {
  schema: MoneySchema,
  of(amount: number, currency: "USD" | "EUR" | "JPY"): Money {
    return MoneySchema.parse({ amount, currency });
  },
} as const;
```

### Python

```python
# domain/shared/money.py
from decimal import Decimal
from typing import Literal
from ..shared.domain_model import DomainModel


class Money(DomainModel):
    amount: Decimal
    currency: Literal["USD", "EUR", "JPY"]

    @field_validator("amount")
    @classmethod
    def _nonnegative(cls, v: Decimal) -> Decimal:
        if v < 0:
            raise ValueError("Amount must be non-negative")
        return v
```

### Rust

```rust
// domain/shared/money.rs
use rust_decimal::Decimal;
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
pub struct Money {
    amount: Decimal,
    currency: Currency,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum Currency {
    Usd,
    Eur,
    Jpy,
}

impl Money {
    pub fn new(amount: Decimal, currency: Currency) -> Result<Self, MoneyError> {
        if amount < Decimal::ZERO {
            return Err(MoneyError::NegativeAmount);
        }
        Ok(Self { amount, currency })
    }

    pub fn amount(&self) -> Decimal {
        self.amount
    }

    pub fn currency(&self) -> Currency {
        self.currency
    }
}
```

### Scala

```scala
// domain/shared/Money.scala
package domain.shared

object Moneys:
  opaque type Money = (BigDecimal, Currency)

  enum Currency:
    case Usd, Eur, Jpy

  object Money:
    def apply(amount: BigDecimal, currency: Currency): Either[MoneyError, Money] =
      Either.cond(
        amount >= 0,
        (amount, currency),
        MoneyError.NegativeAmount,
      )

  extension (m: Money)
    def amount: BigDecimal = m._1
    def currency: Currency = m._2
```

---

## 4. ID Collections and Maps

When IDs are used as map keys or set members, ensure the target language
supports equality and hashing.

| Language   | Equality | Hashing | Notes |
| ---------- | -------- | ------- | ----- |
| TypeScript | Structural (brand erased at runtime) | N/A (use `Map<string, V>` with `.toString()`) | Branded types are structurally `string` at runtime |
| Python     | `__eq__` on wrapper model | `__hash__` on wrapper model | Must implement both for use in `dict`/`set` |
| Rust       | `#[derive(PartialEq, Eq)]` | `#[derive(Hash)]` | Derive macros handle this |
| Scala      | Opaque type delegates to underlying `==` | Opaque type delegates to underlying `hashCode` | Works automatically |

---

## 5. Key Porting Notes

| Concern | Guidance |
| ------- | -------- |
| **TS brands are erased at runtime** | A TS `RequestId` is just a `string` at runtime. When porting to Rust/Scala, you gain true type separation. When porting from Rust/Scala to TS, the brand is weaker -- rely on the Zod schema at parse boundaries. |
| **Python dual approach** | Python offers both `NewType` (static-only, zero cost) and wrapper models (runtime-enforced). When porting from TS brands (structural), `NewType` is the closer analog. When porting from Rust newtypes (enforced), use wrapper models. |
| **Private fields (Rust)** | Rust newtypes with private fields are the strongest guarantee -- there is no way to bypass the constructor from outside the module. When porting to TS/Python, the validation schema/model is the closest equivalent but can be bypassed via casts. |
| **Opaque types (Scala)** | Scala opaque types are zero-cost at runtime (no wrapping) but abstract at compile time. This is closest to TS brands conceptually but with stronger enforcement. When porting to Rust, use a newtype struct. |
| **Serialization** | TS brands serialize as the underlying primitive. Python wrappers may need custom serializers (or use `.value`). Rust newtypes need `Serialize`/`Deserialize` derives (consider `#[serde(transparent)]`). Scala opaque types serialize as the underlying type. |
| **`generate()` factory** | All languages provide a factory for creating new IDs (typically UUID v4). Preserve this when porting -- it keeps ID generation testable via dependency injection at the use-case level. |
| **Cross-module visibility** | Rust's module privacy and Scala's opaque type scope enforce that construction only happens through the factory. TS and Python rely on convention and linter rules. When porting to TS/Python, add a comment or lint rule to prevent raw construction. |
