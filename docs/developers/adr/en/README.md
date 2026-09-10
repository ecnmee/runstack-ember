# Architecture Decision Records

This index lists every ADR for RunStack Ember. "Origin" marks whether the decision was reconstructed from the implementation and existing documentation (no original document survived) or is an original document written at the time the decision was made. See each ADR's own header for detail.

| ID | Title | Origin |
|---|---|---|
| [ADR-0001](0001-code-protection-platform-not-an-obfuscator.md) | Product is a code protection platform, not an obfuscator | Reconstructed |
| [ADR-0002](0002-no-external-key-management-for-licensing.md) | No externally-managed keys for licensing | Reconstructed |
| [ADR-0003](0003-marketing-follows-implementation.md) | Marketing follows implementation | Reconstructed |
| [ADR-0004](0004-aes-256-gcm-encryption-primitive.md) | AES-256-GCM as this project's encryption primitive | Reconstructed |
| [ADR-0005](0005-ast-required-for-structural-transformations.md) | An AST is required for structural transformations | Reconstructed |
| [ADR-0006](0006-editions-are-configuration.md) | Editions are configuration, not code | Reconstructed |
| [ADR-0007](0007-protection-model-layered-by-where-it-happens.md) | Protection model, layered by where protection happens | Original, with a 2026-08-24 addendum resolving the `loader-minification` classification |
| [ADR-0008](0008-asymmetric-license-verification.md) | License verification is asymmetric | Original |
| [ADR-0009](0009-threat-model.md) | Threat model | Original |
| [ADR-0010](0010-second-real-case-not-first-speculative-one.md) | Build for the second real case, not the first speculative one | Reconstructed |
| [ADR-0011](0011-full-source-encryption-runtime-execution-strategy.md) | Full-Source Encryption / Runtime Execution Strategy | Original, with two 2026-08-21 addenda (integrity/ordering, edition activation, and the production-entry-point blocker) |
| [ADR-0012](0012-production-pipeline-entry-point.md) | Production Pipeline Entry Point | Original |

Reconstructed ADRs state, in their own header, that historical wording was not preserved and separate what is supported by evidence from what is reconstructed rationale. They should not be read as literal historical documents.
