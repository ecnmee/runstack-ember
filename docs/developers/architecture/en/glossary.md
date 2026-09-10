# Glossary

Terms as they are actually used in this codebase. Where a term names a real class or file, that is stated explicitly; where a term is a description rather than a literal identifier, this glossary does not claim otherwise.

| Term | Meaning in RunStack Ember |
|---|---|
| Pass | A single transformation implementing the `Pass` interface (`name()`, `layer()`, `process()`). |
| Pipeline | An ordered sequence of `Pass` instances, executed in order by `Pipeline::run()`. |
| PassRegistry | Resolves a stable pass name to a `Pass` instance, and builds a `Pipeline` from an `Edition`. `PassRegistry::withDefaults()` registers every pass this product ships, under its canonical name. |
| Edition | A named, ordered list of pass names, loaded from a PHP file in `src/Editions/` via `Edition::fromFile()`. Declares composition; implies nothing about implementation. |
| Layer | One of `Source`, `Intermediate`, `Runtime`, `Packaging` (the `Layer` enum). Describes what kind of representation a pass acts on and when, per ADR-0007. Does not determine execution order; that is whatever an edition declares. |
| SourceCode | The `Representation` implementation carrying a PHP source string through the pipeline. |
| Loader | The self-contained PHP code `EncryptionPass` generates to decrypt and execute the protected program at runtime. |
| Protected program | The customer's original PHP source, before or after Source-layer transformation, as distinct from the loader wrapped around it once encrypted. |
| Minification | Token-level whitespace and comment removal, implemented by `MinificationPass`. Registered under two names (`minification`, `loader-minification`) for two different targets; see that class's docblock. |
| String protection | Per-literal AES-256-GCM encryption of string values, implemented by `StringProtectionPass`. |
| Control-flow (obfuscation) | Wrapping statements in opaque, always-true guards (`crc32(__FILE__) === crc32(__FILE__)`), implemented by `ControlFlowPass`. Static obfuscation of the protected program's structure; not a runtime integrity mechanism. |
| Symbol rename | Renaming local variables and parameters, implemented by `SymbolRenamePass`. Does not rename properties, classes, or functions. |
| Encryption | Whole-file compression (`gzdeflate`) plus AES-256-GCM encryption plus loader generation, implemented by `EncryptionPass`. Layer: `Packaging`. |
| Integrity verification | A self-checking SHA-256 hash embedded in the distributed file, implemented by `IntegrityVerificationPass`. Layer: `Runtime`. Excluded from any pipeline that also runs `EncryptionPass` (ADR-0011's addendum): GCM already authenticates the ciphertext. |
| CLI | `bin/ember`, the production entry point (ADR-0012). Thin: argument parsing, file I/O, error presentation. Does not construct a `Pass` directly. |
| ParseException | Thrown by `AbstractAstPass` when `nikic/php-parser` cannot parse the input. Extends `InvalidArgumentException`. Lets consumers (the CLI) classify "input was not valid PHP" without depending on the parsing library directly. |
| ProtectionException | Thrown by `bin/ember`, wrapping `PassRegistry`/`Edition`/`Pipeline` domain failures (unknown pass name, empty pipeline, and so on). Not used for parse failures or for bugs in pipeline composition. |
| Enterprise | The edition currently activating `encryption`. `Premium` does not (a deliberate product decision, not a gap). |

## Terms not to confuse

* **Pass vs. Layer**: a pass is a class; a layer is a classification that class declares about itself. Two registrations of the same class (`minification`, `loader-minification`) can declare different layers.
* **Edition vs. Pipeline**: an edition is a declared list of names; a pipeline is the runnable object `PassRegistry` builds from that list.
* **Pipeline vs. PassRegistry**: the registry resolves names to instances; the pipeline runs the resulting instances in order. Neither does the other's job.
* **Runtime vs. Packaging**: Runtime is behavior that executes while the protected application runs (`IntegrityVerificationPass`); Packaging is how the distributed artifact is assembled (`EncryptionPass`). Confusing these was the exact drift ADR-0007's addendum corrected.
* **Layer vs. file location**: `EncryptionPass` and `IntegrityVerificationPass` both live in `src/Source/`, and both report a `Layer` other than `Source`. Where a file sits is not evidence of what layer it belongs to; see ADR-0007's addendum for why this drifted once already.
