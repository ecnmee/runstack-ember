# Roadmap

This roadmap describes intent, not shipped functionality. Per ADR-0003, nothing here is announced as a product feature until it has a working implementation and a test that exercises the specific claim.

## Phase 0: Foundation

- Pipeline and pass manager (ADR-0006).
- Source layer passes: minification, symbol renaming, string protection, using `nikic/php-parser` (ADR-0005).
- Asymmetric license signing and offline verification (ADR-0008).
- CI pipeline running pass-level unit tests.

## Phase 1: Runtime and packaging

- Runtime layer: integrity verification, license enforcement checks.
- Packaging layer: encryption of the final artifact, using AES-256-GCM (ADR-0004), with key handling that satisfies the banned patterns in ADR-0002.
- Control flow passes at the source layer.
- **Status**: full-source encryption (`EncryptionPass`) is implemented and activated in the Enterprise edition, running through a self-contained runtime loader (ADR-0011). `IntegrityVerificationPass` and `EncryptionPass` do not run together; editions choose one or the other (ADR-0011's addendum). Premium does not activate encryption, by deliberate product decision, not as a gap still to close (ADR-0011's addendum).

## Phase 2: Intermediate layer

- Bytecode representation and a real dispatch-based virtual machine, replacing the previous implementation's non-functional VM.
- Benchmarks comparing protected and unprotected code, published alongside methodology.

## Phase 3: Platform

- CLI and REST API surface.
- License server for activation limits and revocation, separate from offline signature verification.
- Comparison documentation against other PHP protection tools, without disparaging claims.
- **Status**: the CLI's contract is decided (`bin/ember`, `PassRegistry::withDefaults()`, `ProtectionException`, `ParseException`) but not yet fully implemented; a real entry point that takes a source path and an edition name and produces protected output is the next concrete piece of work, ahead of any further Source-layer techniques (ADR-0012). No decision has been made yet on an HTTP API beyond keeping the underlying composition adapter-agnostic enough to support one later.

Items move out of this file and into release notes only once they are implemented and tested.
