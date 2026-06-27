# Worked Example: TypeScript to Python JSON Event Exchange

> **When to read:** You need a concrete end-to-end example of a TypeScript
> service producing domain events consumed by a Python service via JSON over
> a message queue.

## Scenario

A TypeScript service manages taxi request state and publishes `DriverAssigned`
events. A Python service consumes these events to update its read model. The
services communicate via a message queue with JSON payloads.

## Sender: TypeScript

### Domain Event Type

```typescript
// Domain event with camelCase fields and branded IDs
type DriverAssignedEvent = {
  kind: "DriverAssigned";
  eventId: EventId;
  eventAt: Date;
  aggregateId: RequestId;
  payload: {
    driverId: DriverId;
    passengerId: PassengerId;
  };
};
```

### Wire DTO

The outbound DTO uses snake_case fields and snake_case values as required by
the wire-format conventions:

```typescript
type DriverAssignedWireDto = {
  event_name: "driver_assigned";
  event_version: 1;
  event_id: string;
  event_at: string; // ISO 8601 UTC
  aggregate_id: string;
  aggregate_name: "taxi_request";
  payload: {
    driver_id: string;
    passenger_id: string;
  };
};

const toWireDto = (event: DriverAssignedEvent): DriverAssignedWireDto => ({
  event_name: "driver_assigned",
  event_version: 1,
  event_id: event.eventId,
  event_at: event.eventAt.toISOString(),
  aggregate_id: event.aggregateId,
  aggregate_name: "taxi_request",
  payload: {
    driver_id: event.payload.driverId,
    passenger_id: event.payload.passengerId,
  },
});
```

### Publish

```typescript
const publishDriverAssigned = async (
  event: DriverAssignedEvent,
  queue: MessageQueue
): Promise<void> => {
  const wireDto = toWireDto(event);
  const json = JSON.stringify(wireDto);
  await queue.publish("taxi.events.driver_assigned", json);
};
```

## Receiver: Python

### Inbound DTO

The inbound DTO enforces the wire contract with strict validation:

```python
from __future__ import annotations

from datetime import datetime
from typing import Literal
from uuid import UUID

from pydantic import BaseModel, ConfigDict, TypeAdapter


class DriverAssignedPayload(BaseModel):
    model_config = ConfigDict(strict=True, extra="forbid")

    driver_id: UUID
    passenger_id: UUID


class DriverAssignedWireEvent(BaseModel):
    model_config = ConfigDict(strict=True, extra="forbid")

    event_name: Literal["driver_assigned"]
    event_version: Literal[1]
    event_id: str
    event_at: datetime
    aggregate_id: str
    aggregate_name: Literal["taxi_request"]
    payload: DriverAssignedPayload


DriverAssignedWireEventAdapter = TypeAdapter(DriverAssignedWireEvent)
```

### Parse and Convert

```python
def handle_driver_assigned(body: bytes) -> None:
    """Handle a raw DriverAssigned event from the message queue."""
    wire_event = DriverAssignedWireEventAdapter.validate_json(body)

    # Convert to domain event if further processing is needed
    domain_event = DriverAssignedEvent(
        event_id=EventId(wire_event.event_id),
        event_at=wire_event.event_at,
        aggregate_id=RequestId(UUID(wire_event.aggregate_id)),
        driver_id=DriverId(wire_event.payload.driver_id),
        passenger_id=PassengerId(wire_event.payload.passenger_id),
    )

    # Update read model, trigger side effects, etc.
    update_read_model(domain_event)
```

## Contract Test Fixture

Both services share a JSON fixture that pins the wire shape. Store this in a
shared fixtures directory or a contract-testing repository:

```json
{
  "event_name": "driver_assigned",
  "event_version": 1,
  "event_id": "evt-a1b2c3d4",
  "event_at": "2024-03-15T10:30:00Z",
  "aggregate_id": "req-550e8400-e29b-41d4-a716-446655440000",
  "aggregate_name": "taxi_request",
  "payload": {
    "driver_id": "6ba7b810-9dad-11d1-80b4-00c04fd430c8",
    "passenger_id": "7c9e6679-7425-40de-944b-e07fc1f90ae7"
  }
}
```

## Sender Test (TypeScript)

Verify that the sender produces JSON matching the contract fixture:

```typescript
import { describe, it, expect } from "vitest";
import contractFixture from "../fixtures/events/driver_assigned_v1.json";

describe("DriverAssigned wire DTO", () => {
  it("serializes to the contract shape", () => {
    const event: DriverAssignedEvent = {
      kind: "DriverAssigned",
      eventId: "evt-a1b2c3d4" as EventId,
      eventAt: new Date("2024-03-15T10:30:00Z"),
      aggregateId: "req-550e8400-e29b-41d4-a716-446655440000" as RequestId,
      payload: {
        driverId: "6ba7b810-9dad-11d1-80b4-00c04fd430c8" as DriverId,
        passengerId: "7c9e6679-7425-40de-944b-e07fc1f90ae7" as PassengerId,
      },
    };

    const wireDto = toWireDto(event);
    const json = JSON.parse(JSON.stringify(wireDto));

    expect(json).toEqual(contractFixture);
  });

  it("uses snake_case for all field names", () => {
    const event = buildDriverAssignedEvent();
    const wireDto = toWireDto(event);
    const json = JSON.parse(JSON.stringify(wireDto));
    const allKeys = collectKeys(json);

    for (const key of allKeys) {
      expect(key).toMatch(/^[a-z][a-z0-9]*(_[a-z0-9]+)*$/);
    }
  });
});

/** Recursively collect all keys from a JSON object. */
function collectKeys(obj: Record<string, unknown>): string[] {
  const keys: string[] = [];
  for (const [key, value] of Object.entries(obj)) {
    keys.push(key);
    if (typeof value === "object" && value !== null && !Array.isArray(value)) {
      keys.push(...collectKeys(value as Record<string, unknown>));
    }
  }
  return keys;
}
```

## Receiver Test (Python)

Verify that the receiver parses the contract fixture correctly:

```python
import json
from pathlib import Path

import pytest


@pytest.fixture
def contract_fixture() -> dict:
    fixture_path = Path(__file__).parent.parent / "fixtures" / "events" / "driver_assigned_v1.json"
    return json.loads(fixture_path.read_text())


def test_driver_assigned_wire_event_parses(contract_fixture: dict) -> None:
    """The inbound DTO successfully parses the contract fixture.

    Uses validate_json because the DTO has strict=True with UUID fields.
    In Pydantic v2 strict mode, validate_python rejects str where UUID is
    expected; validate_json applies JSON-mode coercion (str -> UUID).
    """
    fixture_json = json.dumps(contract_fixture).encode()
    event = DriverAssignedWireEventAdapter.validate_json(fixture_json)

    assert event.event_name == "driver_assigned"
    assert event.event_version == 1
    assert event.aggregate_name == "taxi_request"
    assert str(event.payload.driver_id) == "6ba7b810-9dad-11d1-80b4-00c04fd430c8"
    assert str(event.payload.passenger_id) == "7c9e6679-7425-40de-944b-e07fc1f90ae7"


def test_driver_assigned_rejects_extra_fields(contract_fixture: dict) -> None:
    """Extra fields in the payload are rejected by strict validation."""
    contract_fixture["unexpected_field"] = "should_fail"
    fixture_json = json.dumps(contract_fixture).encode()

    with pytest.raises(Exception):  # ValidationError
        DriverAssignedWireEventAdapter.validate_json(fixture_json)


def test_driver_assigned_rejects_wrong_event_name(contract_fixture: dict) -> None:
    """A mismatched event_name is rejected by the Literal type."""
    contract_fixture["event_name"] = "trip_completed"
    fixture_json = json.dumps(contract_fixture).encode()

    with pytest.raises(Exception):  # ValidationError
        DriverAssignedWireEventAdapter.validate_json(fixture_json)
```

## Key Observations

1. **Snake_case everywhere on the wire.** The TypeScript sender converts
   camelCase domain fields to snake_case in the DTO. The Python receiver
   reads snake_case directly with no alias needed.

2. **Literal types pin the contract.** Both sides use literal types
   (`z.literal` in TS, `Literal` in Python) for `event_name`,
   `event_version`, and `aggregate_name` to catch schema drift at parse time.

3. **Shared fixture, independent tests.** Each side tests against the same
   JSON fixture. If either side's test breaks, the contract has drifted.

4. **Strict validation on both sides.** The TypeScript Zod schema and the
   Python Pydantic model both reject extra or missing fields, catching
   undocumented schema changes early.
