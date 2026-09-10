# Architecture Overview

* Status: Living document
* Last updated: 2026-09-09
* Audience: Developers working on or with the RunStack Ember engine

This document gives a top-down view of how RunStack Ember is put together. It does not replace the ADRs; where a decision has its own rationale, this document points to the ADR instead of repeating it.

## The core idea: a pipeline of passes

RunStack Ember protects a PHP file by running it through an ordered sequence of **passes**. Each pass implements the `Pass` interface (`name()`, `layer()`, `process()`) and transforms a `Representation` (today, always `SourceCode`, a PHP source string) into another `Representation` of the same kind. A `Pipeline` runs the passes in order and records what ran (see the glossary for exact terms).

Three collaborators make this work, each with exactly one responsibility (ADR-0012):

```text
Edition        declares WHAT runs (a named, ordered list of pass names)
PassRegistry   resolves WHICH IMPLEMENTATION a name maps to
Pipeline       executes HOW passes run (in order, recording what ran)
```

An `Edition` is a plain PHP file returning a list of pass names (`src/Editions/free.php`, `basic.php`, `premium.php`, `enterprise.php`), see ADR-0006. `PassRegistry::withDefaults()` is the single place that knows every pass name this product ships and which class implements it, see ADR-0012.

## The four layers

Every pass declares which of four layers it belongs to, based on when and where it acts on the protected program (ADR-0007):

- **Source**: acts on the PHP source before it runs, via the AST or raw tokens. `SymbolRenamePass`, `StringProtectionPass`, `ControlFlowPass`, `MinificationPass`.
- **Intermediate**: acts on a compiled or serialized representation, after Source and before Packaging. Not yet implemented.
- **Runtime**: checks and behavior that run while the protected application executes. `IntegrityVerificationPass`.
- **Packaging**: how the protected artifact is assembled and distributed. `EncryptionPass`, and `MinificationPass`'s second registration (`loader-minification`, ADR-0007's addendum).

`Layer` is classification, not a scheduler: it answers "what kind of thing is this pass," not "when does it run." Execution order is whatever an edition declares, by name (ADR-0012). The current encrypted pipeline, for example, runs `minification` twice, once as `Source` and once (as `loader-minification`) as `Packaging`, with `encryption` in between:

```text
symbol-rename -> string-protection -> control-flow -> minification -> encryption -> loader-minification
```

The non-encrypted pipeline ends at `integrity-verification` instead, since `IntegrityVerificationPass` and `EncryptionPass` never run together (ADR-0011's addendum).

## Licensing

License signing uses an Ed25519 key pair, not a shared secret (ADR-0008). The private key never leaves the license-issuing service; the public key is embedded in every distributed artifact and verifies signatures fully offline, so a protected artifact never needs network access to check its own license. No externally-managed key of any kind, environment variable, license server, or key store, is part of the licensing model (ADR-0002).

## Threat model

RunStack Ember raises the cost of casual and semi-automated attacks: casual source inspection, generic static analysis and decompilers, unauthorized redistribution, casual license tampering. It does not claim to stop a dedicated, technically skilled attacker with unlimited time, an attacker with root or physical access at the moment the artifact runs, or side-channel attacks (ADR-0009). Every feature and every marketing claim is written against this boundary, not as an unqualified strength claim.

## Production entry point

`Edition`, `PassRegistry`, and `Pipeline` compose directly; no facade wraps them, because none is yet justified (ADR-0012). The CLI (`bin/ember`) is a thin adapter: it parses arguments, reads the input file, runs the three-line composition, writes the result, and turns exceptions into clear messages across four categories (usage error, malformed PHP via `ParseException`, protection-composition failure via `ProtectionException`, and anything else left uncaught as a bug signal). A future HTTP API, if built, would be a second adapter converging on the same composition, not a second implementation of it.

## Where to go next

- For exact terminology, see the glossary.
- For why each decision was made, see the ADR index.
- For what is planned but not yet built, see the roadmap.
