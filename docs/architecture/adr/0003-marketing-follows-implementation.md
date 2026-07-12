# ADR-0003: Marketing follows implementation

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

The previous README advertised claims such as "99% protection", "military-grade encryption", "polymorphic VM execution", and "opcode shuffling". A code review found that the "VM" hardcoded a single opcode at compilation time regardless of the shuffle, and that "polymorphic" execution had no dispatch mechanism behind it. The gap between the README and the implementation was large enough to create real credibility risk, independent of how good the rest of the codebase is.

## Decision

No feature is announced in the README, website, or pricing page before it has:

1. A working implementation.
2. An automated test that specifically exercises the claim, not just the presence of the code path. A claim of "encryption" requires a test that confirms the output cannot be trivially reversed without the key; a claim of "VM execution" requires a test that exercises actual opcode dispatch, not a single hardcoded instruction.

Marketing language follows the code. It is never written first and implemented later.

Superlative or unverifiable claims (percentages of protection, "military-grade", comparisons with no cited methodology) are avoided in favor of specific, falsifiable descriptions of what a given level does.

## Consequences

- Some marketing copy will be less dramatic than the previous README.
- Every advertised feature has a corresponding test that a reader could, in principle, be pointed to.
- Adding a new tier or level requires the test to exist before the feature is announced, which slows down marketing but keeps it accurate.
