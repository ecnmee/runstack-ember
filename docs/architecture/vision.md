# Vision

RunStack Ember is a code protection platform for PHP applications.

Ember combines AST transformations, encryption, runtime protection, licensing, and application hardening into a single build pipeline, rather than treating each of those as a separate tool bolted onto a source obfuscator.

## Definition

> RunStack Ember is a modern code protection platform for PHP, combining AST transformations, encryption, runtime protection, licensing and application hardening into a single build pipeline.

## Tagline

Beyond obfuscation.

## What Ember is

- A pipeline of protection passes (ADR-0006) organized into four layers: source, intermediate, runtime, packaging (ADR-0007).
- A platform whose commercial tiers are configuration, not separate codebases: each tier enables a specific set of passes.
- A product whose public claims are always backed by an implementation and a test (ADR-0003), and whose protection claims are stated against an explicit threat model (ADR-0009).

## What Ember is not

- Not a tool that promises to stop a dedicated, well-resourced reverse engineer. See ADR-0009 for the exact boundary.
- Not a place for proprietary or "invented" cryptography. See ADR-0004.
- Not organized around the phrase "Level N" as a technical concept; that language stays only as historical context for anyone comparing against the previous implementation.

## Structure

```text
RunStack Ember

Engine
├── Parser
├── Pipeline
├── Runtime
├── Cryptography
├── Licensing
└── Packaging

Protection Passes
├── Minification
├── Symbol Renaming
├── String Protection
├── Control Flow
├── Runtime Guards
├── Integrity Verification
├── Encryption
└── Virtualization (planned)
```

Commercial editions (Free, Basic, Premium, Enterprise) activate different subsets of Protection Passes. They are not separate implementations.
