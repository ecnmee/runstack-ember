# ADR-0004: Cryptography, standard primitives only

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

Security by obscurity (proprietary ciphers, custom encoding schemes presented as encryption) is a common failure mode in obfuscation and code protection tools, because the market rewards claims that sound novel. The previous implementation already used a correct standard primitive for one of its encryption passes (AES-256-GCM via PHP's `openssl_encrypt`), which shows the team defaults to sound choices when not under pressure to sound impressive.

## Decision

All cryptography uses well reviewed, standard algorithms. Proprietary or "invented" ciphers are never used, and no protection mechanism relies on the algorithm itself being secret.

Approved primitives:

- Ed25519 and X25519 for signing and key exchange.
- AES-256-GCM for authenticated symmetric encryption.
- Argon2id for password or passphrase-derived keys.
- HKDF for key derivation from existing key material.
- PBKDF2 only where compatibility with an existing system requires it, and only with a minimum of 100,000 iterations.

Custom obfuscation techniques (identifier renaming, control flow flattening, dead code injection) are not cryptography and are not marketed as providing cryptographic guarantees.

## Consequences

- No custom cipher implementation is ever written or maintained in-house.
- Security review can focus on key handling and protocol design, since the primitives themselves are already vetted by the broader cryptographic community.
- Some marketing language loses novelty, since "AES-256-GCM" is less exotic sounding than an invented cipher name, but it is verifiable.
