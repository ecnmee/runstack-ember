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

## Phase 2: Intermediate layer

- Bytecode representation and a real dispatch-based virtual machine, replacing the previous implementation's non-functional VM.
- Benchmarks comparing protected and unprotected code, published alongside methodology.

## Phase 3: Platform

- CLI and REST API surface.
- License server for activation limits and revocation, separate from offline signature verification.
- Comparison documentation against other PHP protection tools, without disparaging claims.

Items move out of this file and into release notes only once they are implemented and tested.
