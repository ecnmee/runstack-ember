# RunStack Ember

RunStack Ember protects PHP applications from casual inspection and reverse engineering. It transforms plain PHP source into hardened output through a pipeline of composable protection techniques, organized into a clear layered architecture rather than a single opaque obfuscation step.

This repository hosts the public architecture documentation for Ember: the decisions behind how it works, why it works that way, and what each edition includes. The engine implementation is private.

## How protection works

Ember applies protection through passes, each one a well defined transformation with a single responsibility. Every pass belongs to exactly one of four layers, based on when and where it acts:

**Source layer.** Transformations applied to PHP source before it runs: minification, symbol renaming, string literal protection, and control flow obfuscation.

**Runtime layer.** Checks that run while the protected application executes: integrity verification, which detects tampering with the distributed file.

**Packaging layer.** How the protected artifact is assembled: full-source encryption, which replaces the protected program with an encrypted payload and a self-contained loader, authenticated with AES-256-GCM.

**Intermediate layer.** Reserved for transformations on a compiled or serialized representation of the code, between the source and packaging stages.

This separation means a technique is never tangled with the commercial tier it happens to ship in. A pass declares its layer and its behavior; an edition declares which passes it includes. The two concerns stay independent.

## Editions

Ember ships as four editions, each a specific combination of passes:

| Edition | Protection |
|---|---|
| Free | Minification |
| Basic | Minification, symbol renaming |
| Premium | Minification, symbol renaming, string protection, control flow obfuscation, integrity verification |
| Enterprise | Minification, symbol renaming, string protection, control flow obfuscation, full-source encryption |

Enterprise trades integrity verification for full-source encryption: an AES-256-GCM authenticated cipher already detects any modification to the encrypted artifact, so a separate integrity check would duplicate a guarantee the encryption already provides. Every other layer of protection in Premium carries over unchanged.

## Architecture decisions

Every significant design decision behind Ember is recorded as an Architecture Decision Record before it is implemented, not after. `docs/architecture/adr/` contains the full history: why passes are organized into four layers, why editions are configuration rather than code, why full-source encryption uses a temporary file and `include()` instead of `eval()`, and the reasoning behind every tradeoff in between.

Reading the ADRs in order shows not just what Ember does, but why it does it that way, including the alternatives that were considered and rejected.

## What lives here

This is the public face of RunStack Ember: architecture documentation, design rationale, and the public interface contracts the engine exposes. The protection engine itself, including every pass implementation, lives in a private repository and ships to customers as a licensed product.
