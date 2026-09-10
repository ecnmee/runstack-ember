# ADR-0004: AES-256-GCM as this project's encryption primitive

* Status: Accepted
* Date: 2026-08-24 (reconstructed; see note below)
* Authors: RunStack Team
* Supersedes: none
* Superseded by: none

**Historical note.** The original ADR document was not preserved. Unlike some other reconstructed decisions in this set, the pairing of this number with this decision is not inferred: `StringLiteralEncryptor.php` cites it directly in its own docblock ("a self-invoking closure that decrypts it (AES-256-GCM, ADR-0004)"). What is not preserved is the original historical wording, only the decision and its direct citation.

## Decision supported by evidence

Every encryption operation in this codebase, without exception, uses AES-256-GCM via PHP's `openssl` extension:

* `StringLiteralEncryptor` (used by `StringProtectionPass`) encrypts each protected string literal with `openssl_encrypt($plaintext, 'aes-256-gcm', $this->key, OPENSSL_RAW_DATA, $iv, $tag, '', 16)`, a fresh key generated per build.
* `EncryptionPass` later reuses the exact same cipher and the same key-per-build model for whole-file encryption, on top of `gzdeflate` compression.

No other cipher, mode, or encryption library appears anywhere in `src/`.

## Reconstructed rationale

AES-256-GCM is a standard authenticated-encryption cipher, available through PHP's `openssl` extension without adding a dependency, and its authentication tag provides tamper-evidence as a side effect of encryption itself. That side effect is load-bearing elsewhere in this project: ADR-0011's addendum relies on it directly to justify excluding `IntegrityVerificationPass` from any pipeline that also runs `EncryptionPass`, since GCM already authenticates the ciphertext.

Whether other ciphers were evaluated and rejected, or whether AES-256-GCM was chosen without formal comparison, is not recoverable from the evidence available. What can be stated with confidence is that this project has used one cipher, consistently, everywhere it encrypts, since before the earliest artifact examined for this reconstruction.

## Consequences

* A single cipher choice, used everywhere, means the security properties and limitations of AES-256-GCM (an attacker who obtains the key, which travels alongside the ciphertext per ADR-0002's threat model, can decrypt; a modified ciphertext fails authentication rather than silently decrypting to something wrong) apply uniformly across `StringProtectionPass` and `EncryptionPass`, not one property here and a different one there.
* Any future pass that needs to encrypt something should default to this same primitive unless a specific, stated reason requires otherwise; introducing a second cipher into this codebase would be a decision significant enough to need its own ADR.
