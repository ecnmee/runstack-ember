# Product Requirements Document: RunStack Ember

* Status: Draft
* Date: 2026-09-09
* Authors: RunStack Team
* Origin: Authored from the current vision, roadmap, and ADRs; not a reconstruction of a historical document

**Note on this document.** Unlike the ADRs, this is not a reconstruction of something that existed before. No prior PRD was found in either repository. This document is a new synthesis, written now, of requirements the project's vision, roadmap, and ADRs already establish. Where a requirement traces to a specific ADR, that ADR is cited; where this document states something not yet backed by an ADR or a test, it says so.

## Purpose

RunStack Ember is a code protection platform for PHP applications: AST transformations, encryption, runtime protection, licensing, and application hardening in a single build pipeline, rather than separate tools bolted onto a source obfuscator (Vision).

## Target users

- PHP application vendors who distribute their code to customers or self-hosted installs and need to raise the cost of casual inspection, redistribution, and reverse engineering.
- Teams currently relying on obfuscation alone, who need licensing and tamper-detection as first-class product concerns rather than external add-ons.

## Product scope

### In scope (What Ember is, per Vision and ADR-0007)

- A pipeline of protection passes organized into four layers: Source, Intermediate, Runtime, Packaging.
- Commercial tiers (editions) that are configuration, not separate codebases: each tier is a named, ordered list of pass names (ADR-0006).
- Offline, asymmetric license verification with no shared secret and no externally-managed key (ADR-0002, ADR-0008).
- Every protection claim stated against an explicit threat model, not as an unqualified percentage (ADR-0003, ADR-0009).

### Out of scope (What Ember is not, per Vision and ADR-0009)

- Stopping a dedicated, technically skilled attacker with unlimited time and root or physical access to the running application (ADR-0009).
- Proprietary or "invented" cryptography; only standard, vetted primitives (ADR-0004).
- A single monolithic "Level N" model; the product is organized by layer and by pass, not by an undifferentiated protection level.

## Editions

Editions activate different subsets of the same passes; there is one engine, not one engine per tier (Vision, ADR-0006).

| Edition | Encryption (`EncryptionPass`) | Notes |
|---|---|---|
| Free | Not activated | Baseline Source-layer protection. |
| Basic | Not activated | |
| Premium | Not activated, by deliberate decision | Full-source encryption carries real operational cost (temp-file materialization, decrypt-on-every-request); Premium's pricing/positioning does not currently absorb that cost (ADR-0011's addendum). |
| Enterprise | Activated | Runs `symbol-rename -> string-protection -> control-flow -> minification -> encryption -> loader-minification`; `integrity-verification` is not used alongside encryption (ADR-0011's addendum). |

This table reflects the current codebase (`src/Editions/*.php`) as of the ADRs cited, not a commitment to keep this exact shape as the product evolves.

## Functional requirements

1. **Protect PHP source without changing its behavior.** Every pass's output must be semantically equivalent PHP to its input, verified by tests, not asserted by documentation.
2. **License a build without a network call.** Verification must succeed fully offline using an embedded public key (ADR-0008).
3. **Detect tampering appropriately for the edition.** Non-encrypted editions use `IntegrityVerificationPass`; encrypted editions rely on AES-256-GCM's built-in authentication instead, since the two mechanisms overlap in what they protect against (ADR-0011's addendum).
4. **Produce protected output through one entry point per interface.** The CLI (and any future API) must use the same `Edition` / `PassRegistry` / `Pipeline` composition; neither may construct a `Pass` directly or hardcode an edition's pass order outside an edition file (ADR-0012).
5. **Fail legibly.** Usage errors, malformed input PHP, and protection-composition failures must be distinguishable to the caller; unexpected failures must surface with their real type and stack trace rather than being masked (ADR-0012).
6. **Never destroy the only copy of a customer's source.** Output defaults to a derived filename; an input/output path collision is refused, not silently overwritten (ADR-0012).

## Non-goals (for this document's current scope)

- Committing to an HTTP API design. Composition must stay adapter-agnostic enough to support one later, without deciding its shape now (ADR-0012).
- A layer-ordered execution scheduler. `layer()` is classification, not a scheduling mechanism; execution order is whatever an edition declares (ADR-0012).
- A bytecode / virtualization Intermediate layer. Planned (Vision, Roadmap Phase 2), not yet designed at the requirements level.

## Open questions

- Whether `--lean` (or an equivalent) is ever exposed as a CLI flag versus purely an edition choice (ADR-0012).
- The shape and scope of a future license server for activation limits and revocation, beyond the offline signature verification already decided (Roadmap Phase 3).

## References

Vision, Roadmap, ADR-0002, ADR-0003, ADR-0004, ADR-0006, ADR-0007, ADR-0008, ADR-0009, ADR-0011, ADR-0012.
