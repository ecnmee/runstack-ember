# ADR-0002: No externally-managed keys for licensing

* Status: Accepted
* Date: 2026-08-24 (reconstructed; see note below)
* Authors: RunStack Team
* Supersedes: none
* Superseded by: none

**Note on this document's date and origin.** This ADR is cited by number in project discussion (a design conversation about `EncryptionPass` explicitly noted that sourcing an encryption key from "an environment variable, a license server" would "reopen the key management ADR-0002 already bans for the licensing case"), but the original document was never found in either repository. This reconstructs the decision that citation describes, on the date above. The exact original wording is not preserved; only the decision itself, and the reasoning already visible in how the rest of the codebase behaves consistently with it, are.

## Context

Licensing (per ADR-0008, asymmetric license verification) needs some way to distinguish a validly licensed copy of a protected artifact from one that is not. The most flexible way to do that would involve managing keys somewhere outside the protected artifact itself: an environment variable the customer's server must set, a license server the artifact calls at runtime, or an external key store of some kind. That flexibility comes at a cost: it requires the product to operate, secure, and support infrastructure beyond the artifact it produces, and it requires the customer's deployment to trust and depend on that infrastructure being reachable.

## Decision

Licensing verification does not depend on externally-managed keys. No environment variable, no license server call, no external key store is part of the licensing model. Whatever mechanism ADR-0008 specifies for asymmetric verification, the key material involved is embedded in the artifact at build time, the same key-handling shape already used elsewhere in this codebase for a different purpose: `StringProtectionPass` and `EncryptionPass` both generate a fresh key per build and embed it (or, for encryption, generate and embed both key and ciphertext together) rather than reach for anything external.

## Consequences

* A protected artifact is self-contained with respect to licensing verification: it does not need network access, a configured environment, or a reachable service to check whether it is validly licensed.
* This is a scope boundary, not a security claim: it does not mean license verification is unbreakable, only that this product does not take on the operational burden of running or requiring license-server infrastructure to make that verification possible.
* Any future proposal to add server-side license verification, revocation, or environment binding needs its own ADR; it is not compatible with this decision as stated; and it should be weighed against the operational cost this decision was made specifically to avoid.
