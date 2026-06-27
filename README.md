# kamae-model-converter

Cross-language domain model porting and service interoperability skills for the [kamae](https://github.com/iwasa-kosui/kamae-ts) family.

Two independent skills that bridge the gap between kamae's four language variants:

| Skill | Purpose |
| --- | --- |
| **kamae-model-port** | Port a domain model from one language to another (e.g., rewrite a TS service in Rust) |
| **kamae-model-bridge** | Exchange models between services in different languages via JSON / Protobuf / gRPC |

## Install

```bash
# Claude Code
claude skills add manji-0/agent-skill-modelconverter

# npx
npx @anthropic-ai/skills add manji-0/agent-skill-modelconverter
```

Both skills are installed together. Use only the one relevant to your task.

## Supported Languages

| Language | Kamae skill | Repo |
| --- | --- | --- |
| TypeScript | [kamae](https://github.com/iwasa-kosui/kamae-ts) | iwasa-kosui/kamae-ts |
| Python | [kamae-py](https://github.com/manji-0/kamae-py) | manji-0/kamae-py |
| Rust | [kamae-rs](https://github.com/manji-0/kamae-rs) | manji-0/kamae-rs |
| Scala | [kamae-scala](https://github.com/manji-0/kamae-scala) | manji-0/kamae-scala |

## kamae-model-port

Guides porting kamae domain models between languages while preserving type safety, state-transition correctness, and domain invariants.

### When to use

- Rewriting a service from one language to another
- Translating type-driven domain designs across the kamae family
- Asking "how do I express this TypeScript pattern in Rust?"

### References

| File | Content |
| --- | --- |
| `type-mapping.md` | Master 4-language correspondence table (discriminated unions, branded IDs, Result types, PII, etc.) |
| `state-transition-mapping.md` | Transition function patterns: Companion Object vs `impl` vs `extension` |
| `error-handling-mapping.md` | Error types, Result composition, controller-layer conversion |
| `id-and-branded-types.md` | Branded types, newtypes, opaque types, validation guarantees |
| `pii-and-sensitive.md` | Redaction wrappers, credential types, exposure patterns |
| `boundary-and-dto.md` | External → DTO → domain pipeline per language |
| `persistence-event-mapping.md` | Repository ports, domain events, Transition outcome types |
| `migration-workflow.md` | 9-step porting order with verification criteria and pitfalls |

### Examples

- **TS → Rust**: `examples/taxi-request-ts-to-rs.md`
- **Python → Scala**: `examples/taxi-request-py-to-scala.md`

## kamae-model-bridge

Guides exchanging kamae domain models between polyglot services with shared wire-format conventions.

### When to use

- Services in different languages exchange events via message queues
- Defining JSON or Protobuf contracts between polyglot services
- Setting up contract tests for cross-service boundaries

### Wire conventions (summary)

| Aspect | Convention |
| --- | --- |
| Field names | `snake_case` on the wire |
| Discriminant (`kind`) values | `snake_case` (`"en_route"`, not `"EnRoute"`) |
| Timestamps | ISO 8601 UTC (`"2024-03-15T10:30:00Z"`) |
| UUIDs | Lowercase hyphenated (`"550e8400-..."`) |
| Money | Integer cents + currency code |

Each language converts at the DTO boundary. Python and Rust match the wire format natively; TypeScript and Scala map camelCase ↔ snake\_case in their codecs.

### References

| File | Content |
| --- | --- |
| `wire-format-conventions.md` | Canonical wire rules: fields, scalars, event envelope, null vs absent |
| `discriminant-interop.md` | `kind` and `event_name` value mapping per language |
| `serialization-compatibility.md` | Null handling, strict parsing, numeric precision, date/time |
| `contract-testing.md` | Shared fixtures, CDCT, JSON Schema validation |
| `schema-evolution.md` | Safe vs breaking changes, versioning strategies, migration patterns |
| `dto-boundary-patterns.md` | Sender and receiver DTO design per language |
| `protobuf-mapping.md` | kamae DU → `oneof`, state → `message`, error → gRPC status |

### Examples

- **TS ↔ Python JSON**: `examples/json-interop-ts-py.md`
- **Rust ↔ Scala gRPC**: `examples/grpc-interop-rs-scala.md`

## Rules

Override or extend skill behavior with project-level rules in `.claude/rules/*.md`. See [`rules/README.md`](./rules/README.md) for the format.

## License

MIT
