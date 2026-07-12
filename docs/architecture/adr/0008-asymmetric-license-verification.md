# ADR-0008: License verification is asymmetric

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

ADR-0002 bans hardcoded, shared secrets for license signing as a general rule, in response to a specific failure: the previous implementation signed and verified licenses with the same HMAC secret, and that secret shipped inside the code every customer received. Licensing is important and specific enough to warrant its own decision, so that the correct scheme is documented once, precisely, instead of being re-derived from a general principle each time someone touches license code.

## Decision

License signing and verification use an asymmetric key pair, not a shared secret:

- **Algorithm**: Ed25519.
- **Private key**: generated once per signing environment, never leaves the license issuing service, never appears in any repository, package, or distributed artifact.
- **Public key**: embedded in the code distributed to every customer, regardless of tier. Its presence in customer-facing code is safe by design, since a public key cannot be used to forge a signature.
- **Verification**: fully offline. A customer's application verifies a license signature locally using the embedded public key, without a network call to a license server, though a license server may exist separately for other purposes (activation limits, telemetry, revocation lists).
- **License format**: a deterministic, versioned structure (for example: license ID, tier, issued-to, expiry, feature flags) that is serialized the same way every time before signing, so that verification is reproducible across PHP versions and platforms.

## Consequences

- License issuance requires access to the private signing key, which means it can only happen from a controlled environment, not from any machine that happens to have the codebase.
- A leaked customer license file lets someone see what a license contains, but not create a new valid one, unlike the previous HMAC scheme where the validator itself was enough.
- Revocation of a compromised private key requires rotating the key pair and reissuing the public key to future releases; existing offline installations with the old public key keep validating old licenses until they update, which is a known and accepted tradeoff of offline verification.
