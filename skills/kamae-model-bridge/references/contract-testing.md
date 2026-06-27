# Contract Testing

When to read: Verifying that sender and receiver services agree on the wire
format for cross-language kamae model exchange.

---

## Why contract tests

Schema validation (JSON Schema, Zod, Pydantic) catches structural errors --
wrong types, missing required fields. But it does not catch **semantic drift**:

- Sender starts using `"en_route"` where receiver expects `"enroute"`.
- Sender adds `fare_cents` but receiver still reads `fare` (a renamed field).
- Sender serializes timestamps as Unix milliseconds; receiver expects ISO 8601.

Contract tests pin the **exact wire shape** that both sides depend on. They
break early and loudly when either side's serialization changes.

---

## Shared fixture approach

The most straightforward contract testing strategy for cross-language services:

1. Create JSON fixture files in a shared location (shared repo, Git submodule,
   or published package).
2. Both sender and receiver include these fixtures in their test suites.
3. Sender tests assert: "our outbound DTO serializes to this JSON shape."
4. Receiver tests assert: "this JSON shape deserializes into our inbound DTO
   correctly."

If either side changes its serialization, the fixture comparison breaks.

### Fixture file layout

```
fixtures/
  events/
    driver_assigned_v1.json
    trip_started_v1.json
    trip_completed_v1.json
  commands/
    request_ride_v1.json
  states/
    taxi_request_waiting_v1.json
    taxi_request_en_route_v1.json
```

### Fixture file example

```json
{
  "event_name": "driver_assigned",
  "event_version": 1,
  "event_id": "test-event-001",
  "event_at": "2024-03-15T10:30:00Z",
  "aggregate_id": "test-request-001",
  "aggregate_name": "taxi_request",
  "payload": {
    "kind": "en_route",
    "driver_id": "test-driver-001",
    "passenger_id": "test-passenger-001"
  }
}
```

Fixtures should use deterministic, readable test values -- not random UUIDs.
This makes diffs reviewable when fixtures change.

---

## Testing patterns per language

### TypeScript

```typescript
import { readFileSync } from "fs";
import { describe, it, expect } from "vitest";
import { DriverAssignedWireDto } from "./wire-dtos";
import { toWireDto, fromWireDto } from "./dto-boundary";

const fixture = JSON.parse(
  readFileSync("fixtures/events/driver_assigned_v1.json", "utf-8"),
);

describe("DriverAssigned contract", () => {
  it("deserializes fixture into inbound DTO", () => {
    const result = DriverAssignedWireDto.safeParse(fixture);
    expect(result.success).toBe(true);
    if (result.success) {
      expect(result.data.payload.kind).toBe("en_route");
      expect(result.data.payload.driver_id).toBe("test-driver-001");
    }
  });

  it("serializes outbound DTO to match fixture shape", () => {
    const domain = makeDomainDriverAssigned(); // test factory
    const wire = toWireDto(domain);
    expect(wire).toEqual(fixture);
  });
});
```

### Python

```python
import json
from pathlib import Path
from pydantic import TypeAdapter

from .wire_dtos import DriverAssignedWireDto
from .dto_boundary import to_wire_dto, from_wire_dto

FIXTURE = json.loads(
    Path("fixtures/events/driver_assigned_v1.json").read_text()
)

def test_deserializes_fixture_into_inbound_dto():
    adapter = TypeAdapter(DriverAssignedWireDto)
    dto = adapter.validate_python(FIXTURE)
    assert dto.payload.kind == "en_route"
    assert dto.payload.driver_id == "test-driver-001"

def test_serializes_outbound_dto_to_match_fixture():
    domain = make_domain_driver_assigned()  # test factory
    wire = to_wire_dto(domain)
    assert wire.model_dump(mode="json") == FIXTURE
```

### Rust

```rust
use serde_json;
use std::fs;

#[cfg(test)]
mod tests {
    use super::*;
    use crate::wire_dtos::DriverAssignedWireDto;
    use crate::dto_boundary::{to_wire_dto, from_wire_dto};

    fn load_fixture() -> serde_json::Value {
        let content = fs::read_to_string(
            "fixtures/events/driver_assigned_v1.json"
        ).unwrap();
        serde_json::from_str(&content).unwrap()
    }

    #[test]
    fn deserializes_fixture_into_inbound_dto() {
        let fixture = load_fixture();
        let dto: DriverAssignedWireDto =
            serde_json::from_value(fixture).unwrap();
        assert_eq!(dto.payload.kind, "en_route");
        assert_eq!(dto.payload.driver_id, "test-driver-001");
    }

    #[test]
    fn serializes_outbound_dto_to_match_fixture() {
        let fixture = load_fixture();
        let domain = make_domain_driver_assigned(); // test factory
        let wire = to_wire_dto(&domain);
        let serialized = serde_json::to_value(&wire).unwrap();
        assert_eq!(serialized, fixture);
    }
}
```

### Scala

```scala
import io.circe.parser.decode
import io.circe.syntax._
import org.scalatest.flatspec.AnyFlatSpec
import org.scalatest.matchers.should.Matchers

class DriverAssignedContractSpec extends AnyFlatSpec with Matchers {

  val fixture: String =
    scala.io.Source.fromFile(
      "fixtures/events/driver_assigned_v1.json"
    ).mkString

  "DriverAssignedWireDto" should "deserialize from fixture" in {
    val result = decode[DriverAssignedWireDto](fixture)
    result shouldBe a[Right[_, _]]
    val dto = result.toOption.get
    dto.payload.kind shouldBe "en_route"
    dto.payload.driverId shouldBe "test-driver-001"
  }

  it should "serialize to match fixture shape" in {
    val domain = makeDomainDriverAssigned() // test factory
    val wire = toWireDto(domain)
    val serialized = wire.asJson.noSpaces
    val fixtureJson = io.circe.parser.parse(fixture).toOption.get
    wire.asJson shouldBe fixtureJson
  }
}
```

---

## Consumer-Driven Contract Testing (CDCT)

In CDCT, the **consumer** (receiver) defines the contract -- the subset of
fields it actually reads -- and the **provider** (sender) verifies it produces
conformant output.

### How it works

1. Consumer team writes a contract: "I expect `kind`, `driver_id`, and
   `passenger_id` in the payload. I do not read `note`."
2. Contract is stored in a shared location or contract broker.
3. Provider CI runs the contract against its serialization output.
4. If the provider breaks a field the consumer depends on, the build fails.
5. If the provider adds a field the consumer does not use, the build passes.

### Tools

- **Pact**: Language-agnostic CDCT framework. Has libraries for all four kamae
  languages. Good for HTTP APIs and message queues.
- **Hand-rolled fixtures in a shared repo**: Simpler, no extra tooling. Each
  consumer adds fixture files; providers run all fixtures in CI.

Choose Pact when there are many consumers with different field subsets. Choose
hand-rolled fixtures when there are few consumers and the team prefers
simplicity.

---

## JSON Schema as contracts

Generate JSON Schema from one language's DTO definition, then validate the
other language's output against that schema.

| Language   | Tool                        | Example                                  |
| ---------- | --------------------------- | ---------------------------------------- |
| TypeScript | `zod-to-json-schema`        | `zodToJsonSchema(WireDto)`               |
| Python     | Pydantic `model_json_schema`| `WireDto.model_json_schema()`            |
| Rust       | `schemars` crate            | `schema_for!(WireDto)`                   |
| Scala      | Manual or library-specific  | Circe does not generate schema natively  |

### Workflow

1. Designate one language as the schema source of truth (often the one with
   the richest type system or the most constraints).
2. Generate JSON Schema in CI and commit it to the shared contract location.
3. Other languages validate their serialization output against the schema in
   their own test suites.
4. Schema changes trigger contract test failures in downstream services.

### Limitations

JSON Schema validates structure but not semantics. It cannot express:

- Discriminant value mappings (only that `kind` is a string enum).
- Field co-occurrence rules ("if `kind` is `en_route`, `driver_id` must be
  present").
- Business invariants.

Use JSON Schema as a first line of defense. Use fixture-based contract tests
for semantic correctness.

---

## When to update contract tests

Update fixtures and contracts whenever any of these changes:

- [ ] A field is added, removed, or renamed
- [ ] A discriminant value is added, removed, or changed
- [ ] A field's type changes (e.g., `string` to `int`, `required` to `optional`)
- [ ] A new event or command type is introduced
- [ ] The envelope structure changes (new metadata fields, version bump)
- [ ] The serialization convention changes (e.g., null vs absent)

### Process for updating

1. Update the fixture file to reflect the new wire shape.
2. Update the sender's serialization test to match the new fixture.
3. Update the receiver's deserialization test to match the new fixture.
4. If using schema-based contracts, regenerate the JSON Schema.
5. Deploy sender and receiver in the correct order based on the compatibility
   of the change (see `schema-evolution.md`).

---

## Anti-patterns

| Anti-pattern                                  | Why it fails                              |
| --------------------------------------------- | ----------------------------------------- |
| Testing only the sender or only the receiver  | Drift on the other side goes undetected   |
| Using random/generated test data in fixtures  | Diffs are unreadable; flaky comparisons   |
| Fixture files not version-controlled          | No history of wire format changes         |
| Contract tests run only locally, not in CI    | Regressions slip through                  |
| Comparing serialized JSON as strings          | Field ordering differences cause failures; compare parsed JSON objects instead |
