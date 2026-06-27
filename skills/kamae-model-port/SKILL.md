---
name: kamae-model-port
description: |
  Kamae cross-language model porting guide. Maps domain model constructs
  (discriminated unions, branded IDs, state transitions, error types,
  PII protection, boundary defense, persistence patterns) between
  TypeScript, Python, Rust, and Scala following each language's kamae idioms.

  TRIGGER when: porting a kamae domain model from one language to another,
  rewriting a service in a different language, translating type-driven
  domain designs across the kamae family, or asking "how do I express
  this TypeScript/Python/Rust/Scala pattern in another language?"
  SKIP: single-language work (use the language-specific kamae skill instead),
  inter-service communication (use kamae-model-bridge), code generation
  from IDL schemas.
license: MIT
---

# Kamae Model Port — Cross-Language Domain Model Migration

Port kamae domain models between TypeScript, Python, Rust, and Scala while
preserving type safety, state-transition correctness, and domain invariants.

Read only the reference files relevant to the current porting task.

## Step 0: Load Applicable Rules

Before any other step, read matching rule files in priority order:

1. `.claude/rules/*.md` and `.codex/rules/*.md` in the project root
2. `~/.claude/rules/*.md` and `~/.codex/rules/*.md`
3. `../../rules/defaults/*.md` relative to this `SKILL.md`

For each rule:

- Read YAML frontmatter. Skip unless `applies-to` is `kamae-model-port` or `*`.
- Group by `name`. The first tier wins; within a tier, lexicographically last wins.
- Apply surviving rules throughout the task.

## Step 1: Identify Source and Target Languages

Determine which of the four kamae languages are involved:

| Language | Kamae skill | Key libraries |
| --- | --- | --- |
| TypeScript | `kamae` | Zod/Valibot/ArkType, neverthrow/fp-ts/byethrow |
| Python | `kamae-py` | Pydantic v2, uv |
| Rust | `kamae-rs` | serde, thiserror, secrecy |
| Scala | `kamae-scala` | Cats/ZIO, Circe, Refined |

## Step 2: Understand the Source Model

Read the source language's kamae skill references to understand the model
structure. Focus on:

- State types and their discriminated union
- ID types and value objects
- Transition functions and their signatures
- Error types
- Repository ports

Use the table below to locate the specific reference files for each language:

| Language | Domain modeling | State transitions | Error handling | Boundary defense |
| --- | --- | --- | --- | --- |
| TypeScript | `kamae/domain-modeling.md` | `kamae/state-modeling.md` | `kamae/error-handling.md` | `kamae/boundary-defense.md` |
| Python | `kamae-py/references/domain-modeling.md` | `kamae-py/references/state-transitions.md` | `kamae-py/references/error-handling.md` | `kamae-py/references/boundary-defense.md` |
| Rust | `kamae-rs/references/domain-modeling.md` | `kamae-rs/references/state-modeling.md` | `kamae-rs/references/error-handling.md` | `kamae-rs/references/boundary-defense.md` |
| Scala | `kamae-scala/references/domain-modeling.md` | `kamae-scala/references/state-transitions.md` | `kamae-scala/references/error-handling.md` | `kamae-scala/references/boundary-defense.md` |

## Step 3: Apply the Type Mapping

Read [`references/type-mapping.md`](./references/type-mapping.md) for the
comprehensive cross-language correspondence table. This is the core reference
for every porting task.

Load additional mapping files based on the concepts being ported:

| Concept | Reference |
| --- | --- |
| State transitions | [`state-transition-mapping.md`](./references/state-transition-mapping.md) |
| Error handling | [`error-handling-mapping.md`](./references/error-handling-mapping.md) |
| IDs and branded types | [`id-and-branded-types.md`](./references/id-and-branded-types.md) |
| PII protection | [`pii-and-sensitive.md`](./references/pii-and-sensitive.md) |
| Boundary defense / DTOs | [`boundary-and-dto.md`](./references/boundary-and-dto.md) |
| Persistence and events | [`persistence-event-mapping.md`](./references/persistence-event-mapping.md) |

## Step 4: Follow the Migration Workflow

Read [`references/migration-workflow.md`](./references/migration-workflow.md) for
the recommended order of porting: state types first, then IDs, transitions,
errors, boundary DTOs, and finally repository ports.

## Step 5: Apply Target Language Idioms

After mechanical translation, read the target language's kamae skill references
to ensure the result follows idiomatic patterns:

- File/module organization conventions
- Naming conventions (camelCase vs snake_case vs PascalCase)
- Library-specific patterns (e.g., Zod brands, Pydantic ConfigDict, serde derives)
- Quality gate commands for the target language

Refer to the skill reference path table in [Step 2](#step-2-understand-the-source-model)
to locate the target language's domain modeling, state transition, error handling,
and boundary defense references.

## Examples

Read examples only when a concrete worked migration would clarify the task:

- [`examples/taxi-request-ts-to-rs.md`](./examples/taxi-request-ts-to-rs.md) — TypeScript to Rust
- [`examples/taxi-request-py-to-scala.md`](./examples/taxi-request-py-to-scala.md) — Python to Scala

## Related: Cross-Service Interop

If the ported service exchanges models with other services over a wire protocol
(JSON, Protobuf, message queues), see **kamae-model-bridge** for wire-format
conventions, discriminant interop rules, contract testing, and schema evolution
guidance. Porting changes the language; bridging preserves the wire contract.

## Applying These Principles

These mappings are strong defaults, not mechanical rules. Each language has
idioms that do not map 1:1. When a direct translation would fight the target
language's type system or ecosystem, adapt the pattern to achieve the same
domain safety guarantee through the target language's native mechanisms.

When deviating from the mapping table, state the reason and verify that the
deviation preserves: (1) invalid states cannot be represented, (2) invalid
transitions cannot compile, (3) external data is validated at boundaries.
