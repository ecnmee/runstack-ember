# ADR-0012: Production Pipeline Entry Point

* Status: Proposed
* Date: 2026-08-22
* Deciders: RunStack Ember
* Scope: Pipeline, CLI
* Related: ADR-0006 (editions are configuration, not code), ADR-0007 (protection model), ADR-0011 (full-source encryption)

## Context

`Edition`, `PassRegistry`, and `Pipeline` are implemented, unit-tested, and exercised end-to-end for two editions (`FreeEditionIntegrationTest`, `EnterpriseEditionIntegrationTest`). Each of those tests, however, builds its own `PassRegistry` by hand, registering every pass factory inline. No file in this codebase does that once, for real use, outside a test.

`src/CLI/` exists as an empty, reserved directory (`.gitkeep`, pointing to ADR-0007). Nothing has been built there yet.

Meanwhile, `tools/diagnose/obfuscate-file.php` already does something that looks similar to a production entry point: it takes a file path, applies a sequence of passes, and writes output. It is the wrong template to copy. It constructs `SymbolRenamePass`, `StringProtectionPass`, `ControlFlowPass`, and so on directly, in an order hardcoded into the script itself, bypassing `Edition` and `PassRegistry` entirely. That is correct for a diagnostic tool whose entire purpose is to let a developer try arbitrary pass combinations. It is exactly the pattern a production entry point must not repeat, because it throws away the one guarantee `Edition`/`PassRegistry` exist to provide: that the set of passes protecting a customer's code is a named, declared, testable configuration, not an ad hoc list assembled at the call site.

This ADR defines the contract between edition selection and a protected PHP artifact on disk, and states which part of that contract is CLI-specific versus core library behavior, before any of it is implemented.

## Decision

### Decision summary

No new facade is created for the entry point at this stage. The entry point uses the existing abstractions, `Edition`, `PassRegistry`, and `Pipeline`, directly to turn an external request into an execution. The composition must not be duplicated across the CLI and any future API, nor re-encoded by hand anywhere outside an edition file.

`Pipeline` remains responsible for execution. `PassRegistry` remains responsible for resolving passes. `Edition` remains responsible for declaring the composition. The entry point only translates between the external interface and these abstractions.

A facade will be justified only when a real responsibility appears that does not naturally belong to the entry point, `Edition`, `PassRegistry`, or `Pipeline`. It will not be created merely to encapsulate calls that already compose cleanly.

### The core library gains one new piece: a populated `PassRegistry` factory

Every test that builds a working `PassRegistry` today registers the same set of factories by hand:

```php
$registry->register('symbol-rename', fn () => new SymbolRenamePass());
$registry->register('string-protection', fn () => new StringProtectionPass());
$registry->register('control-flow', fn () => new ControlFlowPass());
$registry->register('minification', fn () => new MinificationPass());
$registry->register('encryption', fn () => new EncryptionPass());
$registry->register('loader-minification', fn () => new MinificationPass('loader-minification'));
$registry->register('integrity-verification', fn () => new IntegrityVerificationPass());
```

This list has one canonical, correct form. It should exist once, as a named constructor on `PassRegistry` itself:

```php
PassRegistry::withDefaults(): self
```

`withDefaults()` is the single place that knows every pass name currently shipped and which class implements it. Adding a new pass to the product means adding one line here, not updating every consumer that builds a registry.

This stays inside the existing `Pipeline` namespace. It is not a new architectural concept; it is a second named constructor on a class that already has one (`Edition::fromFile`), following the same convention.

### Turning an edition name and PHP source into protected PHP source

The entry point is responsible for translating an external request into a pipeline execution. Pipeline construction uses `Edition` and `PassRegistry`; execution remains `Pipeline`'s responsibility. Concretely, today, that translation looks like this:

```php
$edition = Edition::fromFile($editionName, $editionsDir . "/{$editionName}.php");
$pipeline = PassRegistry::withDefaults()->buildPipeline($edition);
$result = $pipeline->run(new PipelineContext(new SourceCode($sourceText)));
```

No facade class wraps this. `Edition`, `PassRegistry`, and `Pipeline` already compose cleanly; adding a fourth class whose only job is to call the other three in order would be structure added because a diagram suggested a box, not because the existing pieces fail to compose. This follows the same restraint already applied to the Runtime Layer in ADR-0011: build the abstraction when a second concrete need for it appears, not preemptively.

This composition is what "the core" means for the rest of this ADR. It is plain library code, usable from a CLI script, a future HTTP endpoint, a test, or a REPL, with no dependency on how it is invoked.

**When a facade would become justified.** Not to encapsulate the three calls above; that would be encapsulation for its own sake. A facade earns its place only when a real responsibility appears that does not naturally belong to the entry point, `Edition`, `PassRegistry`, or `Pipeline` individually. Nothing identified so far meets that bar.

### The anti-pattern this ADR exists to rule out

The separation already built into this codebase assigns each concern to exactly one place:

```text
Edition        declares WHAT runs (a named, ordered list of pass names)
PassRegistry   resolves WHICH IMPLEMENTATION a name maps to
Pipeline       executes HOW passes run (in order, recording what ran)
Pass           transforms WHATEVER REPRESENTATION it received
```

`tools/diagnose/obfuscate-file.php`, written earlier in this same project, is a working example of what happens when an entry point skips this separation: it constructs `new SymbolRenamePass()`, `new StringProtectionPass()`, `new ControlFlowPass()`, and so on directly, in an order hardcoded into the script, and it has its own opinion about which passes belong together (the `--lean` flag's logic lives entirely inside that script). That is legitimate for a diagnostic tool whose entire purpose is trying arbitrary combinations. It is exactly what the production entry point must not do: know pass names beyond the one string identifying which edition was requested, construct any `Pass` directly, or encode edition-specific rules (which passes go together, in what order) anywhere outside an edition file.

The production entry point's job is not "hide the composition behind a facade." It is: consume `Edition` and `PassRegistry` as they are, and never reconstruct, by hand, the decisions those two already encode.

### The entry point is not a new architectural layer

CLI is an adapter. A future HTTP API, if one is ever built, is another adapter. Both must converge on the same composition mechanism rather than each carrying its own copy of it:

```text
                 CLI              (future) HTTP API
                  \                    /
                   \                  /
                 Entry point translation
                          |
                   Edition selection
                          |
                   Pass resolution (PassRegistry)
                          |
                          v
                      Pipeline
                          |
                          v
                        Passes
```

If an HTTP API is ever added, the risk this ADR wants to foreclose in advance is `CLI -> its own logic` and `API -> its own logic` diverging over time into two slightly different, both slightly wrong, reimplementations of the same three lines. Both adapters call the same composition; only argument parsing and response formatting differ between them.

This does not require a facade class to be true. It requires only that the composition is written once and both adapters call it, wherever "once" ends up living.

### `Layer` is classification, not a scheduler

`EncryptionPass` is `Packaging`, `IntegrityVerificationPass` is `Runtime`, `SymbolRenamePass`/`StringProtectionPass`/`ControlFlowPass`/`MinificationPass` are `Source` (with `MinificationPass`'s second, `loader-minification` registration left as an open question by the classification fix, not resolved here). Every one of those classifications is now correct against ADR-0007's text.

None of that means the entry point runs passes in `Source -> Intermediate -> Runtime -> Packaging` order. It does not, and this ADR does not propose making it do so. `Pipeline` executes passes in the order an edition declares, by name; that order is what `EnterpriseEditionIntegrationTest` and `EncryptionPassTest::test_it_locks_in_the_documented_encrypted_pipeline_order` guard, and it does not match a naive pass-by-layer ordering (`minification` runs twice, once before `encryption` and once after, both instances classified `Source`; `encryption`, classified `Packaging`, runs before the second `minification`, not after every `Source`-layer pass in some global sense). `layer()` is architectural classification: it answers "what kind of thing is this pass," not "when does it run." Conflating the two would mean deriving execution order from `layer()` and getting it wrong, since the real order already encodes constraints (see ADR-0011's addendum) that a simple four-bucket sort cannot express.

### The CLI is I/O and error presentation around that composition, nothing else

`src/CLI/` (or a `bin/ember` script depending on how PHP packages typically expose executables; see open question below) is responsible for exactly:

1. Parsing arguments: input file path, edition name, output file path.
2. Reading the input file into a string.
3. Running the three-line composition above.
4. Writing the resulting `SourceCode->code` string to the output path.
5. Catching every exception the composition can throw and turning it into a clear message and a non-zero exit code, instead of a raw stack trace.

It does not construct a `Pass` directly. It does not know pass names beyond what it needs to pass an edition name string through. If a future change means the CLI needs to know more than that to do its job, that is a signal the contract in this ADR is wrong somewhere, not a reason to let the CLI reach past it.

### Error handling: catalog what can already go wrong, decide once

The composition above can currently throw, from what already exists in this codebase:

* `RuntimeException` from `Edition::fromFile` (missing file, malformed return value).
* `OutOfBoundsException` from `PassRegistry::make` (unknown pass name in an edition file).
* `LogicException` from `PassRegistry::make` (factory/name mismatch) or `PassRegistry::buildPipeline` (duplicate pass name).
* `RuntimeException` from `Pipeline::run` (empty pass list).
* `InvalidArgumentException` from any pass (wrong payload type reaching it).
* A parse error from `nikic/php-parser` (malformed input PHP) inside any `AbstractAstPass`-based pass.
* `RuntimeException` from `EncryptionPass` (encryption/compression failure) or a decrypt-time failure, though the latter only happens later, when the protected artifact runs, not when it is built.

Two options:

**Option A.** The CLI catches `\Throwable` broadly at the top level, prints `$e->getMessage()`, exits non-zero. Simple, but a future second consumer (an HTTP API, for instance) has to redo the same broad catch and hope the message text is presentable to whoever is asking.

**Option B.** The core composition wraps every exception above into a single `RunStack\Ember\Pipeline\ProtectionException` (or similar), preserving the original as `getPrevious()`. Every consumer, CLI or otherwise, only needs to catch one type to handle "this build failed, here is why" uniformly.

This ADR proposes Option B, but does not consider it fully decided; see open questions.

## Non-goals

This ADR does not:

* Decide whether `encryption`/`--lean` is exposed as a CLI flag versus purely an edition choice. `--lean` remains a `tools/diagnose/obfuscate-file.php` concept per ADR-0011's addendum; whether the production CLI ever needs an equivalent is a separate, later decision.
* Introduce a validator that checks an edition's pass list against the four-layer model, or design a layer-ordered execution scheme. See "`Layer` is classification, not a scheduler" above; both remain out of scope for the same reason.
* Decide on an HTTP API. "CLI/API" in this ADR's title and diagrams refers to CLI now, with the composition kept adapter-agnostic enough that an HTTP layer could reuse it later, not to a concrete HTTP design decided here.
* Touch `EncryptionPass`, `MinificationPass`, `PassRegistry`'s existing methods, or any edition's pass list. Those are closed per ADR-0011 and its addendum.

## Open questions

1. **Where does the CLI script physically live and how is it invoked?** `composer.json` has no `bin` entry yet. Options include a `bin/ember` executable script (the common Composer convention, installed to `vendor/bin/ember` for consumers who require this package) or something under `src/CLI/` invoked via `php src/CLI/whatever.php`. This affects `composer.json`, not just `src/`.
2. **Is `ProtectionException` (Option B above) worth the abstraction now**, or is broad `\Throwable` catching in the CLI (Option A) sufficient until a second consumer actually exists? This is the same "build it when a second need appears" question already applied twice in this ADR; it is not obviously resolved the same way both times, because exception handling is harder to retrofit across every call site later than a registry factory is.
3. **What does the CLI do with a `ParseError` from malformed input PHP?** Every other error case above has a message already written by this codebase. A parse error's message comes from `nikic/php-parser` directly, which was never designed to be shown to an end user as-is. Deciding to pass it through unmodified versus wrapping it with file/line context is a real, separate design question.
4. **Output destination when it is not provided.** Overwrite the input file, require an explicit output path always, or write to stdout by default with a flag to write to a file? Each has a different risk profile for a tool whose entire job is destructive by nature (replacing readable PHP with something else).

## Status

Proposed. Not implemented. Written for review before any code changes, per the same process ADR-0011 followed.
