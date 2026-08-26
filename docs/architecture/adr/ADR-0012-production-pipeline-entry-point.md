# ADR-0012: Production Pipeline Entry Point

* Status: Accepted
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

### Error handling: four categories, not one wrapper

The composition can fail for reasons that are not all the same kind of failure, and collapsing them into one exception type would hide that difference from every consumer:

| Category | Example | Type |
|---|---|---|
| CLI usage / validation error | missing argument, input file does not exist, output path equals input path | Handled entirely inside the CLI, before the composition ever runs. Not a library exception at all. |
| Malformed input PHP | `nikic/php-parser` cannot parse the file | `RunStack\Ember\Source\ParseException`, thrown by `AbstractAstPass::process()` when it catches `\PhpParser\Error` internally. See "Dependency boundary" below: the CLI never sees `\PhpParser\Error` directly, and does not need to. |
| Protection composition failure | unknown pass name, duplicate pass in an edition, empty pipeline, malformed edition file, encryption/compression failure | Wrapped as `RunStack\Ember\Pipeline\ProtectionException`, with the original exception preserved as `getPrevious()`. |
| Unexpected failure | anything not in the three categories above | Left uncaught by the CLI's specific handlers. It propagates as whatever it actually is, with its real class name and stack trace intact, because that is a bug signal, not a normal failure mode of the tool, and hiding it behind `ProtectionException` would make real defects harder to find, not easier. |

**Dependency boundary: `AbstractAstPass` translates the external parser's exception, so the CLI never depends on it.** `\PhpParser\Error extends \RuntimeException`. If `AbstractAstPass` did not translate it, that inheritance would matter for catch order at every consuming call site: a `catch (\RuntimeException)` block placed before a `catch (\PhpParser\Error)` block would silently absorb parse errors as if they were protection-composition failures. Rather than push that ordering requirement onto every consumer, `AbstractAstPass::process()` catches `\PhpParser\Error` once, where it already touches the parser directly, and throws `RunStack\Ember\Source\ParseException` (extending `\InvalidArgumentException`, an unrelated hierarchy from `\RuntimeException`, so no catch-order hazard exists downstream). The CLI catches `ParseException`, not `\PhpParser\Error`; it does not need to know which parsing library this package uses internally.

This also means `AbstractAstPass::process()`'s other two `\InvalidArgumentException` sites, wrong payload type reaching a pass and a parse that succeeds but yields an empty AST, are deliberately unchanged. The first indicates a bug in how the pipeline was composed, not a property of the input PHP; the second is a real, separate open question this decision does not resolve (is an empty-but-syntactically-valid file a parse failure or something else) and is left as `\InvalidArgumentException`, not promoted to `ParseException`, until that question is decided on its own.

`ProtectionException` wraps only the exception types the composition is already known to throw for domain reasons: `OutOfBoundsException` and `LogicException` from `PassRegistry`, `RuntimeException` from `Edition::fromFile`, `Pipeline::run`, and `EncryptionPass`. It does not wrap `ParseException` (a different category, above, with its own catch block) or the bug-indicating `\InvalidArgumentException` case described in the previous paragraph, which belongs in the "unexpected failure" category, uncaught, visible.

### Output: a predictable derived name by default, never silently in place

```text
ember protect input.php --edition=enterprise
```

writes to `input.ember.php` in the same directory, derived by inserting `.ember` before the original extension. This never requires `--output` for the common case, and it never overwrites `input.php`.

```text
ember protect input.php --edition=enterprise --output=protected.php
```

writes to the given path instead. Whichever path is used, resolved (not literal-string) input and output paths are compared before anything runs; if they resolve to the same file, the CLI refuses and exits with a usage error rather than silently destroying the source. There is no `--in-place` flag in this version. Overwriting the only copy of a file this tool's entire purpose is to transform is a dangerous default to make convenient before there is a concrete reason to need it.

`--edition` has no default and is always required. Silently choosing one on the caller's behalf risks either under-protecting (defaulting to Free when the caller assumed more) or doing more than asked (defaulting to Enterprise, running encryption unexpectedly).

## Non-goals

This ADR does not:

* Decide whether `encryption`/`--lean` is exposed as a CLI flag versus purely an edition choice. `--lean` remains a `tools/diagnose/obfuscate-file.php` concept per ADR-0011's addendum; whether the production CLI ever needs an equivalent is a separate, later decision.
* Introduce a validator that checks an edition's pass list against the four-layer model, or design a layer-ordered execution scheme. See "`Layer` is classification, not a scheduler" above; both remain out of scope for the same reason.
* Decide on an HTTP API. "CLI/API" in this ADR's title and diagrams refers to CLI now, with the composition kept adapter-agnostic enough that an HTTP layer could reuse it later, not to a concrete HTTP design decided here.
* Touch `EncryptionPass`, `MinificationPass`, `PassRegistry`'s existing methods, or any edition's pass list. Those are closed per ADR-0011 and its addendum.

## Decided

1. **The CLI lives at `bin/ember`**, the standard Composer executable convention (`"bin": ["bin/ember"]`), not under `src/CLI/`. `src/CLI/` remains reserved for classes, per its own `.gitkeep`, should CLI-specific logic ever grow past what a thin script can hold; nothing in this version's five responsibilities needs one.
2. **`ProtectionException` is created**, scoped to exactly the domain-failure category in the table above, not as a catch-all for every `\Throwable` the composition could produce.
3. **`AbstractAstPass` translates `\PhpParser\Error` into `RunStack\Ember\Source\ParseException`**, preserving the original as `getPrevious()`. The CLI catches `ParseException`, not `\PhpParser\Error`, and re-presents it with the input file path added, since the parser itself never sees a file path, only a string. This was corrected after `bin/ember`'s first implementation attempt assumed `\PhpParser\Error` would reach it directly; it does not, because `AbstractAstPass` already catches and re-throws it as a plain `\InvalidArgumentException`, indistinguishable from the unrelated "wrong payload type" case without this change. See "Dependency boundary" above for the reasoning in full.
4. **Output defaults to a derived filename (`input.ember.php`)**, `--output` overrides it, and an input/output path collision is a refused usage error, never a silent overwrite.

## Status

Accepted. All four questions this ADR opened with are decided above. `PassRegistry::withDefaults()` is implemented and tested (91/91, monorepo). `ProtectionException` and `bin/ember` are the remaining implementation work this ADR authorizes.
