# ADR-0002: Security first, with explicit banned patterns

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

A code review of the previous implementation found two concrete security failures that a general principle like "prefer security over convenience" would not have caught in code review, because it is not falsifiable:

1. The Level 6 encryption routine (`DynamicDecryptor`) derived a symmetric key from a hardcoded default secret and returned that key alongside the ciphertext in the same payload, which defeats the encryption entirely regardless of the cipher used.
2. The license validator, shipped to every customer including the free tier, and the license generator, meant to stay private, used the same hardcoded HMAC secret to sign and verify licenses. Any customer could extract the secret from the code they legitimately received and forge licenses for any tier.

Both failures happened inside code that looked reasonable at a glance and used a real, standard cipher (AES-256-GCM). The cipher choice was correct; the key handling around it was not. A principle-level ADR would not have prevented either issue, because both authors likely believed they were following "security first" at the time.

## Decision

Security first is enforced through a list of explicitly banned patterns, checked in code review and, where feasible, in CI:

- No decryption key, or any value from which a decryption key can be trivially derived, may be stored, transmitted, or embedded alongside the ciphertext it decrypts.
- No secret used for signing (HMAC, JWT, license signatures) may exist in code that is distributed to a customer. License signing uses an asymmetric key pair (Ed25519); only the public key is embedded in the distributed validator, the private key never leaves the signing service.
- No hardcoded constant of the form `SECRET_KEY`, `default_secret`, or equivalent may act as a silent fallback. Missing secrets fail loudly at startup instead of falling back to a known default.
- Base64 encoding is never described or counted as encryption in code, documentation, or marketing claims.
- Any new cryptographic routine touching keys or secrets requires a second reviewer before merge, specifically checking this list.

## Consequences

- Some short-term convenience is lost: license issuance now requires access to a private key, not just a shared constant, and local development needs a dev-only key pair.
- Code review has an explicit, checkable list instead of relying on reviewer judgment alone.
- The two failures found in the previous implementation become regression tests: one test asserts that no key material appears in an encrypted payload, another asserts that a license signed with a leaked validator secret is rejected.
