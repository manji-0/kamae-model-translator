---
name: kamae-model-bridge
description: |
  Kamae cross-language service interoperability guide. Defines wire-format
  conventions, discriminant interop rules, serialization compatibility,
  contract testing, schema evolution, DTO boundary patterns, and
  Protobuf/gRPC mapping for exchanging domain models between
  TypeScript, Python, Rust, and Scala services.

  TRIGGER when: services written in different kamae languages exchange
  domain models via JSON, Protobuf, gRPC, message queues, or event
  streams; defining wire-format contracts between polyglot services;
  asking "how do I send this model from my TS service to the Python
  consumer?" or similar cross-service questions.
  SKIP: single-language work (use the language-specific kamae skill),
  porting a model to rewrite a service in another language (use
  kamae-model-port), pure infrastructure or deployment concerns.
license: MIT
---

# Kamae Model Bridge — Cross-Language Service Interoperability

Exchange kamae domain models between services written in TypeScript, Python,
Rust, and Scala while preserving type safety, schema compatibility, and
domain invariants across service boundaries.

Read only the reference files relevant to the current interop task.

## Step 0: Load Applicable Rules

Before any other step, read matching rule files in priority order:

1. `.claude/rules/*.md` and `.codex/rules/*.md` in the project root
2. `~/.claude/rules/*.md` and `~/.codex/rules/*.md`
3. `../../rules/defaults/*.md` relative to this `SKILL.md`

For each rule:

- Read YAML frontmatter. Skip unless `applies-to` is `kamae-model-bridge` or `*`.
- Group by `name`. The first tier wins; within a tier, lexicographically last wins.
- Apply surviving rules throughout the task.

## Step 1: Identify Communication Protocol

Determine the wire format and transport:

| Protocol | Reference |
| --- | --- |
| JSON over HTTP / REST | [`wire-format-conventions.md`](./references/wire-format-conventions.md) + [`serialization-compatibility.md`](./references/serialization-compatibility.md) |
| JSON over message queue | [`wire-format-conventions.md`](./references/wire-format-conventions.md) + [`schema-evolution.md`](./references/schema-evolution.md) |
| Protobuf / gRPC | [`protobuf-mapping.md`](./references/protobuf-mapping.md) |
| Mixed (JSON events + gRPC commands) | Read both JSON and Protobuf references |

## Step 2: Identify Sender and Receiver Languages

Note which kamae languages are on each side. The wire format is language-neutral;
each side translates between the wire representation and its local domain types.

| Side | Responsibility |
| --- | --- |
| Sender | domain → outbound DTO → serialize to wire format |
| Receiver | deserialize from wire → inbound DTO → domain |

## Step 3: Apply Wire Format Conventions

Read [`references/wire-format-conventions.md`](./references/wire-format-conventions.md)
for the canonical rules that all kamae services must follow when exchanging data.
This is the core reference for every bridge task.

## Step 4: Load Protocol-Specific References

Based on the protocol from Step 1:

| Topic | Reference |
| --- | --- |
| Discriminant value interop | [`discriminant-interop.md`](./references/discriminant-interop.md) |
| JSON serialization details | [`serialization-compatibility.md`](./references/serialization-compatibility.md) |
| DTO design for sender/receiver | [`dto-boundary-patterns.md`](./references/dto-boundary-patterns.md) |
| Contract testing | [`contract-testing.md`](./references/contract-testing.md) |
| Schema versioning | [`schema-evolution.md`](./references/schema-evolution.md) |
| Protobuf / gRPC mapping | [`protobuf-mapping.md`](./references/protobuf-mapping.md) |

## Step 5: Validate Both Sides

After implementing the bridge:

1. Verify the sender's outbound DTO produces conformant wire JSON/Protobuf
2. Verify the receiver's inbound DTO parses the sender's output correctly
3. Write contract tests that pin the wire shape (see [`contract-testing.md`](./references/contract-testing.md))
4. Check schema evolution safety for any new or changed fields

## Examples

Read examples only when a concrete worked interop scenario would clarify the task:

- [`examples/json-interop-ts-py.md`](./examples/json-interop-ts-py.md) — TypeScript ↔ Python JSON event exchange
- [`examples/grpc-interop-rs-scala.md`](./examples/grpc-interop-rs-scala.md) — Rust ↔ Scala gRPC command exchange

## Related: Language Migration

If you are rewriting a service in a different language (not just connecting two
existing services), see **kamae-model-port** for the cross-language type mapping
tables, migration workflow, and idiom translation guidance. Bridging preserves
the wire contract between services; porting translates the domain model itself.

## Applying These Principles

Wire-format conventions are strongest at the boundary. Inside each service,
follow the language-specific kamae skill for local domain patterns. The bridge
skill governs only what crosses the wire.

When a convention conflicts with an established team or organization standard,
follow the standard and document the deviation. The goal is interoperability,
not uniformity for its own sake.
