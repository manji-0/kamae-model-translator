# PII and Sensitive Data Mapping

When to read: Porting PII protection, sensitive field handling, redaction
patterns, credential wrappers, or logging guards between languages.

---

## Overview

All kamae languages wrap personally identifiable information (PII) and
credentials in a dedicated type that prevents accidental exposure. The
wrapper redacts by default (in `toString`, `toJSON`, `Debug`, logging)
and requires an explicit call to access the plaintext. Each language also
provides defense-in-depth at the logging/serialization layer.

---

## 1. PII Wrapper Patterns

### TypeScript -- `Sensitive<T>` closure-based wrapper

The raw value is captured in a closure. There is no field to access
directly -- only `unwrap()` reveals the plaintext. `toJSON()` and
`toString()` return `"[REDACTED]"`.

```typescript
// domain/shared/sensitive.ts
import { z, ZodType } from "zod";

export interface Sensitive<T> {
  unwrap(): T;
  toJSON(): string;
  toString(): string;
}

export const Sensitive = {
  of<T>(value: T): Sensitive<T> {
    return {
      unwrap: () => value,
      toJSON: () => "[REDACTED]",
      toString: () => "[REDACTED]",
    };
  },

  schema<T>(inner: ZodType<T>): ZodType<Sensitive<T>> {
    return inner.transform((v) => Sensitive.of(v));
  },
} as const;
```

Usage in a domain model:

```typescript
// domain/passenger/passenger.ts
import { z } from "zod";
import { Sensitive } from "../shared/sensitive";

export const PassengerSchema = z.object({
  passengerId: PassengerId.schema,
  email: Sensitive.schema(z.string().email()),
  phone: Sensitive.schema(z.string()),
  displayName: z.string(),
});

export type Passenger = z.infer<typeof PassengerSchema>;

// Access:
const email: string = passenger.email.unwrap();

// Safe in logs:
console.log(passenger.email);          // "[REDACTED]"
console.log(JSON.stringify(passenger)); // { ..., "email": "[REDACTED]", ... }
```

### Python -- `Redacted[T]` frozen model + `SecretStr`

General PII uses `Redacted[T]`, a generic frozen model. Credentials use
Pydantic's `SecretStr` / `SecretBytes`.

```python
# domain/shared/redacted.py
from typing import Generic, TypeVar
from pydantic import BaseModel

T = TypeVar("T")


class Redacted(BaseModel, Generic[T]):
    """Wrapper that redacts PII in repr/str/JSON."""

    model_config = {"frozen": True}
    _value: T

    def __init__(self, value: T) -> None:
        super().__init__()
        object.__setattr__(self, "_value", value)

    @property
    def value(self) -> T:
        return self._value

    def expose_for_delivery(self) -> T:
        """Named accessor making intent explicit in code review."""
        return self._value

    def __repr__(self) -> str:
        return "Redacted(***)"

    def __str__(self) -> str:
        return "[REDACTED]"

    def model_dump(self, **kwargs) -> dict:
        return {"value": "[REDACTED]"}
```

Usage:

```python
# domain/passenger/passenger.py
from pydantic import SecretStr
from ..shared.redacted import Redacted
from ..shared.domain_model import DomainModel


class Passenger(DomainModel):
    passenger_id: PassengerId
    email: Redacted[str]
    phone: Redacted[str]
    display_name: str
    api_token: SecretStr  # for credentials


# Access:
email_plain: str = passenger.email.value
token_plain: str = passenger.api_token.get_secret_value()

# Safe in logs:
print(passenger.email)      # [REDACTED]
print(passenger.api_token)  # **********
```

### Rust -- `Redacted<T>` with custom `Debug`/`Display`

The wrapper implements `Debug` and `Display` to show `[REDACTED]`. The
inner field is private. Named `expose_for_*()` methods provide access.
For credentials, use `secrecy::SecretString`.

```rust
// domain/shared/redacted.rs
use std::fmt;
use serde::{Serialize, Serializer};

#[derive(Clone)]
pub struct Redacted<T>(T);

impl<T> Redacted<T> {
    pub fn new(value: T) -> Self {
        Self(value)
    }

    /// Named accessor -- makes exposure intent visible in code review.
    pub fn expose_for_persistence(&self) -> &T {
        &self.0
    }

    /// Named accessor for delivery (email, SMS, etc.).
    pub fn expose_for_delivery(&self) -> &T {
        &self.0
    }
}

impl<T> fmt::Debug for Redacted<T> {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "[REDACTED]")
    }
}

impl<T> fmt::Display for Redacted<T> {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "[REDACTED]")
    }
}

impl<T> Serialize for Redacted<T> {
    fn serialize<S: Serializer>(&self, serializer: S) -> Result<S::Ok, S::Error> {
        serializer.serialize_str("[REDACTED]")
    }
}
```

Usage:

```rust
// domain/passenger/mod.rs
use crate::domain::shared::Redacted;
use secrecy::SecretString;

pub struct Passenger {
    pub passenger_id: PassengerId,
    pub email: Redacted<String>,
    pub phone: Redacted<String>,
    pub display_name: String,
    pub api_token: SecretString,   // for credentials
}

// Access:
let email: &String = passenger.email.expose_for_delivery();
let token: &str = passenger.api_token.expose_secret();

// Safe in logs:
tracing::info!(?passenger.email); // [REDACTED]
// #[derive(Debug)] on Passenger will use Redacted's Debug impl
```

Important: Do **not** `#[derive(Debug)]` on structs containing PII fields
unless all PII fields use `Redacted<T>` or `SecretString`. Otherwise the
raw value leaks into debug output.

### Scala -- `Redacted[T]` case class with overridden `toString`

The wrapper overrides `toString` and provides named `exposeFor*` methods.
For credentials, a dedicated `ApiToken` class with safe `toString` is used.

```scala
// domain/shared/Redacted.scala
package domain.shared

final case class Redacted[T] private (private val inner: T):
  override def toString: String = "[REDACTED]"

  /** Named accessor -- intent visible in code review. */
  def exposeForPersistence: T = inner

  /** Named accessor for delivery channels. */
  def exposeForDelivery: T = inner

object Redacted:
  def apply[T](value: T): Redacted[T] = new Redacted(value)
```

Credential wrapper type:

```scala
// domain/shared/ApiToken.scala
package domain.shared

object ApiTokens:
  /** Wraps a credential string with safe toString.
    * A final class (not opaque type) is used so toString can be overridden
    * in the class body — extension methods cannot override inherited methods.
    */
  final class ApiToken private (private val raw: String):
    override def toString: String = "[REDACTED]"
    def exposeForAuth: String = raw

  object ApiToken:
    def apply(raw: String): ApiToken = new ApiToken(raw)
```

Usage:

```scala
// domain/passenger/Passenger.scala
package domain.passenger

import domain.shared.{Redacted, ApiTokens}
import ApiTokens.*

case class Passenger(
    passengerId: PassengerId,
    email: Redacted[String],
    phone: Redacted[String],
    displayName: String,
    apiToken: ApiToken,
)

// Access:
val emailPlain: String = passenger.email.exposeForDelivery
val tokenPlain: String = passenger.apiToken.exposeForAuth

// Safe in logs:
println(passenger.email)    // [REDACTED]
println(passenger.apiToken) // [REDACTED]
```

---

## 2. Exposure Pattern Comparison

| Language   | General PII access         | Credential access             | Named intent accessor           |
| ---------- | -------------------------- | ----------------------------- | ------------------------------- |
| TypeScript | `sensitive.unwrap()`       | `sensitive.unwrap()`          | Same method for both            |
| Python     | `redacted.value`           | `secret.get_secret_value()`   | `expose_for_delivery()` (PII)   |
| Rust       | `redacted.expose_for_*()`  | `secret.expose_secret()`      | `expose_for_persistence()`, `expose_for_delivery()` |
| Scala      | `redacted.exposeFor*`      | `token.exposeForAuth`         | `exposeForPersistence`, `exposeForDelivery` |

**Porting note:** TS uses a single `unwrap()` for all access. When porting
to Rust/Scala/Python, split into named methods that declare the reason for
exposure (`expose_for_persistence`, `expose_for_delivery`, etc.). When
porting to TS, collapse named accessors into `unwrap()`.

---

## 3. Logging and Serialization Guards (Defense-in-Depth)

The wrapper type is the **primary** defense. Defense-in-depth at the
logging/serialization layer catches cases where a developer accidentally
uses a raw value instead of the wrapper.

### TypeScript -- Pino redaction config

```typescript
// infrastructure/logger.ts
import pino from "pino";

export const logger = pino({
  redact: {
    paths: [
      "*.email",
      "*.phone",
      "*.ssn",
      "*.apiToken",
      "req.headers.authorization",
    ],
    censor: "[REDACTED]",
  },
});
```

This catches raw strings logged under known PII field names even if the
`Sensitive` wrapper was not used.

### Python -- `PiiRedactionFilter` on logging + OTel span scrubbing

```python
# infrastructure/logging_config.py
import logging
import re


class PiiRedactionFilter(logging.Filter):
    """Defense-in-depth: scrub known PII patterns from log messages."""

    _patterns = [
        (re.compile(r"email=\S+"), "email=[REDACTED]"),
        (re.compile(r"phone=\S+"), "phone=[REDACTED]"),
        (re.compile(r"ssn=\S+"), "ssn=[REDACTED]"),
    ]

    def filter(self, record: logging.LogRecord) -> bool:
        msg = record.getMessage()
        for pattern, replacement in self._patterns:
            msg = pattern.sub(replacement, msg)
        record.msg = msg
        record.args = ()
        return True


# Apply to root logger:
logging.getLogger().addFilter(PiiRedactionFilter())
```

OTel span attribute scrubbing:

```python
# infrastructure/otel_config.py
from opentelemetry.sdk.trace.export import SpanExporter

PII_ATTRIBUTES = {"user.email", "user.phone", "user.ssn"}

class SanitizingExporter(SpanExporter):
    def __init__(self, inner: SpanExporter) -> None:
        self._inner = inner

    def export(self, spans):
        for span in spans:
            for attr in PII_ATTRIBUTES:
                if attr in span.attributes:
                    span.attributes[attr] = "[REDACTED]"
        return self._inner.export(spans)
```

### Rust -- Custom `Debug` impl; no `Display` on PII

Rust's defense-in-depth is structural: if `Redacted<T>` is the only way
to hold PII, then `Debug` and `Display` are always safe. The additional
guard is to **never** derive `Debug` on structs that hold raw PII.

```rust
// BAD -- raw PII leaks via derive(Debug):
#[derive(Debug)]
pub struct BadPassenger {
    pub email: String,    // raw PII!
    pub phone: String,    // raw PII!
}

// GOOD -- PII wrapped, Debug is safe:
#[derive(Debug)]
pub struct GoodPassenger {
    pub email: Redacted<String>,    // Debug shows [REDACTED]
    pub phone: Redacted<String>,    // Debug shows [REDACTED]
}
```

For tracing spans, use the field-level redaction:

```rust
tracing::info!(
    email = %passenger.email,  // Display: [REDACTED]
    "Processing passenger"
);
```

### Scala -- Custom `toString`; sanitizing appender

Scala's `Redacted[T]` already overrides `toString`. The defense-in-depth
layer is a log appender that pattern-matches on known PII shapes.

```scala
// infrastructure/SanitizingAppender.scala
package infrastructure

import ch.qos.logback.classic.spi.ILoggingEvent
import ch.qos.logback.core.AppenderBase

class SanitizingAppender extends AppenderBase[ILoggingEvent]:
  private val piiPatterns = List(
    "email=\\S+".r   -> "email=[REDACTED]",
    "phone=\\S+".r   -> "phone=[REDACTED]",
    "ssn=\\S+".r     -> "ssn=[REDACTED]",
  )

  override def append(event: ILoggingEvent): Unit =
    var msg = event.getFormattedMessage
    for (pattern, replacement) <- piiPatterns do
      msg = pattern.replaceAllIn(msg, replacement)
    // forward sanitized message to downstream appender
    ???
```

---

## 4. Boundary Parsing with PII

When parsing external input (HTTP request, message queue payload), PII
fields should be wrapped **at the boundary** so they never exist as raw
values inside the domain.

### TypeScript

```typescript
// adapters/http/create-passenger-schema.ts
import { z } from "zod";
import { Sensitive } from "../../domain/shared/sensitive";
import { PassengerId } from "../../domain/passenger/passenger-id";

export const CreatePassengerBodySchema = z.object({
  email: Sensitive.schema(z.string().email()),
  phone: Sensitive.schema(z.string()),
  displayName: z.string().min(1),
});
```

### Python

```python
# adapters/http/create_passenger_dto.py
from pydantic import BaseModel, field_validator
from domain.shared.redacted import Redacted


class CreatePassengerBody(BaseModel):
    email: str
    phone: str
    display_name: str

    def to_domain(self) -> "CreatePassengerCommand":
        return CreatePassengerCommand(
            email=Redacted(self.email),
            phone=Redacted(self.phone),
            display_name=self.display_name,
        )
```

### Rust

```rust
// adapters/http/create_passenger_dto.rs
use crate::domain::shared::Redacted;
use serde::Deserialize;

#[derive(Deserialize)]
pub struct CreatePassengerBody {
    pub email: String,
    pub phone: String,
    pub display_name: String,
}

impl CreatePassengerBody {
    pub fn into_command(self) -> CreatePassengerCommand {
        CreatePassengerCommand {
            email: Redacted::new(self.email),
            phone: Redacted::new(self.phone),
            display_name: self.display_name,
        }
    }
}
```

### Scala

```scala
// adapters/http/CreatePassengerDto.scala
package adapters.http

import domain.shared.Redacted

case class CreatePassengerBody(
    email: String,
    phone: String,
    displayName: String,
):
  def toCommand: CreatePassengerCommand =
    CreatePassengerCommand(
      email = Redacted(email),
      phone = Redacted(phone),
      displayName = displayName,
    )
```

---

## 5. Key Porting Notes

| Concern | Guidance |
| ------- | -------- |
| **Closure vs private field** | TS uses a closure to hide the value (no field to access). Python/Rust/Scala use private fields with accessor methods. When porting from TS, replace the closure with a private field. When porting to TS, replace the private field with a closure. |
| **Credential separation** | Python separates credentials (`SecretStr`, `SecretBytes`) from general PII (`Redacted[T]`). Rust uses `secrecy::SecretString`. Scala uses a dedicated opaque type. TS does not distinguish -- both use `Sensitive<T>`. When porting from Python/Rust/Scala to TS, collapse credential and PII wrappers into `Sensitive`. When porting to Python/Rust/Scala, split them. |
| **Named exposure methods** | Rust and Scala use named `expose_for_*()` / `exposeFor*` methods that declare intent. Python provides both `.value` and named methods. TS uses generic `unwrap()`. When porting to Rust/Scala, add intent-declaring method names. |
| **`Debug`/`Display` in Rust** | Rust requires explicit opt-in to `Debug` and `Display`. Never `#[derive(Debug)]` on a struct with raw PII fields. Other languages do not have this concern because `__repr__`/`toString`/`toJSON` are overridden on the wrapper. |
| **Serialization** | TS's `Sensitive.toJSON()` returns `"[REDACTED]"`. Python's `model_dump()` returns `"[REDACTED]"`. Rust's `Serialize` impl returns `"[REDACTED]"`. Scala needs a custom encoder. When porting, ensure the serialization path redacts -- do not assume the default serializer is safe. |
| **Boundary wrapping point** | All languages wrap PII **at the boundary** (HTTP handler, message consumer, DB read). Raw PII should never enter the domain layer. Preserve this when porting -- the DTO-to-domain conversion is where wrapping happens. |
| **Defense-in-depth approach** | TS uses Pino redaction config. Python uses logging filters + OTel scrubbing. Rust relies on structural safety (custom Debug). Scala uses log appender pattern-matching. These are additive -- always port both the wrapper type and the logging guard. |
