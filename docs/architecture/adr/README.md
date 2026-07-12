# Architecture Decision Records

This folder contains the Architecture Decision Records (ADRs) for RunStack Ember.

An ADR captures a significant architectural decision, the context that led to it, and its consequences. ADRs are immutable once accepted: if a decision changes, a new ADR supersedes the old one instead of editing it in place.

## Index

| ID | Title | Status |
|----|-------|--------|
| [0001](0001-product-is-a-code-protection-platform.md) | Product is a code protection platform, not an obfuscator | Accepted |
| [0002](0002-security-first-banned-patterns.md) | Security first, with explicit banned patterns | Accepted |
| [0003](0003-marketing-follows-implementation.md) | Marketing follows implementation | Accepted |
| [0004](0004-cryptography-standard-primitives-only.md) | Cryptography: standard primitives only | Accepted |
| [0005](0005-ast-before-regex.md) | AST before regex | Accepted |
| [0006](0006-pipeline-of-passes.md) | Pipeline of passes instead of fixed levels | Accepted |
| [0007](0007-protection-model.md) | Protection model, layered by where protection happens | Accepted |
| [0008](0008-asymmetric-license-verification.md) | License verification is asymmetric | Accepted |
| [0009](0009-threat-model.md) | Threat model | Accepted |

## Format

Each ADR follows the same structure:

- **Status**: Proposed, Accepted, Superseded, or Deprecated.
- **Date**: when the ADR was accepted.
- **Authors**: who proposed the decision.
- **Supersedes** / **Superseded by**: links to related ADRs, or `—` if none apply.
- **Context**: what problem or situation motivates the decision.
- **Decision**: what was decided.
- **Consequences**: what becomes easier or harder as a result.

## Immutability

Accepted ADRs are immutable. If the architecture changes, create a new ADR that supersedes the previous one, and update both documents' Supersedes / Superseded by fields. Do not rewrite an accepted ADR in place.

## How to add a new ADR

1. Copy the next sequential number (four digits, zero padded).
2. Use a short, descriptive, kebab case title.
3. Fill in Status, Context, Decision, and Consequences.
4. Add a row to the index table above.
5. Commit with `doc: add ADR-00NN <title>`.
