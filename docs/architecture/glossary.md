# Glossary

Terms used across the Ember codebase and documentation. Add to this file whenever a new term is introduced in an ADR or in code, so readers do not need to reconstruct meaning from context.

**Pass**
A single, independently testable transformation applied to the code or its intermediate representation. Owns exactly one responsibility (for example: renaming identifiers, injecting dead code, encrypting a payload). See ADR-0006.

**Pipeline**
The ordered sequence of passes that runs when protecting an application. Configured per build, per commercial edition.

**Layer**
One of four stages a pass belongs to, based on when and where it acts: Source, Intermediate, Runtime, Packaging. See ADR-0007.

**Edition**
A commercial tier (Free, Basic, Premium, Enterprise). Defined as a configuration of which passes are enabled, not as separate code.

**Level (deprecated)**
The previous implementation's term for a numbered protection stage tied directly to a commercial tier. Superseded by Pass and Layer. Kept in this glossary only so historical references make sense.

**Threat model**
The explicit statement of which attackers a protection technique is designed to stop, and which it is not. See ADR-0009.

**License signature**
An Ed25519 signature over a deterministic, serialized license structure, verified offline using a public key embedded in the distributed code. See ADR-0008.
