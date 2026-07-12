# RunStack Ember

**Beyond obfuscation.**

RunStack Ember is a modern code protection platform for PHP, combining AST transformations, encryption, runtime protection, licensing and application hardening into a single build pipeline.

```
runstack/ember
```

```php
use RunStack\Ember\...;
```

```bash
ember protect app/
ember build
ember verify
ember inspect
```

## What this is

Ember is not a single obfuscation technique. It is a pipeline of protection passes, organized into four layers (source, intermediate, runtime, packaging), that PHP applications run through at build time. Which passes run is controlled by edition (Free, Basic, Premium, Enterprise); the underlying engine is shared.

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

## Status

This is a ground-up rewrite. See [`docs/architecture/roadmap.md`](docs/architecture/roadmap.md) for what is implemented versus planned. Per [ADR-0003](docs/architecture/adr/0003-marketing-follows-implementation.md), a capability is documented here only once it has a working implementation and a test that exercises the specific claim, so this README will grow slowly and deliberately rather than all at once.

## Threat model

Ember is designed to raise the cost of casual and semi-automated attacks: source inspection, generic static analysis tools, casual redistribution, and casual license tampering. It does not claim to stop a dedicated, well-resourced attacker with unlimited time and full access to the machine running the protected code. See [ADR-0009](docs/architecture/adr/0009-threat-model.md) for the full boundary.

## Documentation

- [Vision](docs/architecture/vision.md) — what Ember is and is not.
- [Architecture Decision Records](docs/architecture/adr/README.md) — every significant technical decision, with context.
- [Glossary](docs/architecture/glossary.md) — shared vocabulary.
- [Roadmap](docs/architecture/roadmap.md) — what is planned, phase by phase.

## License

Licensing terms to be added once the licensing model (see [ADR-0008](docs/architecture/adr/0008-asymmetric-license-verification.md)) is implemented.
