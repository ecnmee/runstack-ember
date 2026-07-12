# ADR-0001: Product is a code protection platform, not an obfuscator

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

The previous implementation grew feature by feature, without a product architecture mature enough to support the commercial claims made in its README. Starting over gives an opportunity to define scope before writing more code, rather than continuing to stack features on top of an unclear foundation.

The product already contains more than obfuscation: licensing, activation, a CLI, a REST API, and enterprise tooling. Scoping it narrowly as "a PHP obfuscator" undersells what it does and constrains future growth to a single word.

## Decision

The product is defined as **RunStack Ember**, a code protection platform for PHP applications. Every component that ships under this name must contribute to one of the following pillars:

- Obfuscation
- Encryption
- Licensing
- Runtime protection
- Integrity verification
- Anti-tamper
- Build pipeline

Features that do not fit one of these pillars belong in a different product.

## Consequences

- The README, pricing tiers, and marketing language describe a platform, not a single tool.
- New features are evaluated against these seven pillars before being accepted into the roadmap.
- The namespace and package name move away from `Obfuscator` as the root term, to avoid re-anchoring the product to a single technique.
