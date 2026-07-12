# ADR-0006: Pipeline of passes instead of fixed levels

## Status

Accepted

## Date

2026-07-10

## Authors

RunStack Team

## Supersedes

—

## Superseded by

—

## Context

The previous implementation organized protection techniques as numbered levels (Level 1 through Level 6), each implemented as a single, mostly monolithic class tied directly to a commercial tier (Free, Basic, Premium, Enterprise). This made two things hard:

1. Testing a single technique in isolation, since a level class often did several unrelated things at once (for example, Level 6 combined VM compilation, encryption, and anti-debug checks in one place).
2. Fixing or extending one technique without touching the others, since the boundaries between techniques matched commercial tiers rather than engineering concerns.

This is also connected to ADR-0002: the key exposure bug in the previous Level 6 encryption was easier to miss because encryption, compilation, and packaging were not separable units that could be tested and reviewed independently.

## Decision

The core engine is a pipeline of independent, ordered passes, similar to how compiler backends such as LLVM structure optimization and code generation stages. Each pass:

- Takes an AST or intermediate representation as input and produces one as output.
- Has a single, testable responsibility (for example: `IdentifierRenamePass`, `StringObfuscationPass`, `DeadCodeInjectionPass`, `ControlFlowFlatteningPass`, `EncryptionPass`, `VmCompilationPass`).
- Can be unit tested without running the full pipeline.

Commercial tiers (Free, Basic, Premium, Enterprise) are defined as a configuration: an ordered list of which passes are enabled, not as separate code paths. Adding a new tier means changing configuration, not writing a new monolithic class.

## Consequences

- The VM and encryption passes can be rewritten independently, once a real bytecode VM is implemented, without needing to touch the passes handling renaming or dead code.
- Regression tests target individual passes, which makes it possible to have a specific test asserting that `EncryptionPass` never emits key material inside its own output, directly enforcing ADR-0002.
- Slightly more infrastructure code upfront (a pass manager, a shared intermediate representation) compared to independent level classes, which pays off as soon as a second pass needs to reuse work done by an earlier one.
