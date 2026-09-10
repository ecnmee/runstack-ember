# ADR-0013: Intermediate layer: bytecode representation and virtual machine

* Status: Proposed
* Date: 2026-09-10
* Authors: RunStack Team
* Supersedes: none
* Superseded by: none
* Related: ADR-0005 (AST required for structural transformations), ADR-0007 (protection model, four layers), ADR-0011 (full-source encryption / runtime execution strategy), ADR-0012 (production pipeline entry point)

## Context

`Layer::Intermediate` has existed as a name since ADR-0007 and as an empty directory (`src/Intermediate/.gitkeep`) since the project's early structure, but nothing has been built there. Every implemented pass today is `Source` (`SymbolRenamePass`, `StringProtectionPass`, `ControlFlowPass`, `MinificationPass`), `Runtime` (`IntegrityVerificationPass`), or `Packaging` (`EncryptionPass`, and `MinificationPass`'s `loader-minification` registration). The roadmap's Phase 2 names this gap directly: "Bytecode representation and a real dispatch-based virtual machine, replacing the previous implementation's non-functional VM," alongside published benchmarks.

This ADR exists to settle the representation and execution design before any code is written, the same way ADR-0011 settled full-source encryption's design before `EncryptionPass` existed. It does not decide performance targets or a ship date; the roadmap already says nothing here is a product feature until it has an implementation and a test (ADR-0003).

## What "Intermediate" means today, concretely

ADR-0007 defines this layer as acting on "a compiled or serialized representation of the code, after the source layer, before packaging." Two very different designs both satisfy that sentence, and this ADR has to pick one before anything else is decided:

**Option A: stay `SourceCode -> SourceCode`, self-contained.** The pass compiles the protected PHP into a custom instruction stream, embeds that stream (as a PHP array or string literal) alongside a small dispatch-loop interpreter, and emits the result as a `.php` file that runs the interpreter against the embedded instructions when included. This is the same shape `EncryptionPass` already uses: a payload plus a self-contained loader, no shared runtime, no Composer dependency added to the protected application (ADR-0011). `Layer::Intermediate` would classify it the same way `Layer::Packaging` classifies `EncryptionPass`, by what it operates on, not by requiring a new `Representation` type.

**Option B: introduce a genuinely new `Representation` type** (for example `BytecodeProgram`, distinct from `SourceCode`), with `PassRegistry`/`Pipeline` gaining the ability to chain a pass whose output type differs from its input type. This is a real architectural change: today every pass is `SourceCode -> SourceCode` (`Pass::process()`'s contract has never varied), and `Pipeline::run()` has never had to check that one pass's output type matches the next pass's expected input type.

This ADR proposes **Option A**, for the same reason ADR-0011 and ADR-0012 both declined to introduce new abstractions speculatively: the self-contained pattern is already proven (it is what ships in Enterprise today), it requires no change to `Pipeline`, `PassRegistry`, or the `Pass` contract, and Option B's generality is not yet justified by a concrete need. If a second Intermediate-layer pass is ever proposed that genuinely cannot work this way, Option B becomes a decision for that pass's own ADR, not a speculative one made here.

## Proposed decision

- A new pass, tentatively named `BytecodeCompilationPass`, registered as `bytecode-compilation`, classified `Layer::Intermediate`.
- Input: the AST already available from the Source-layer passes (this pass extends `AbstractAstPass`, the same base every AST-consuming pass already uses, per ADR-0005).
- Output: PHP source containing (1) an embedded instruction stream and constant pool, serialized in a form `var_export()` or an equivalent can reconstruct without `eval()` on the data itself, and (2) a small, self-contained dispatch-loop interpreter function that executes that instruction stream. No shared Runtime API, per the same reasoning ADR-0011 used to keep the Runtime Layer closed until a second concrete need appears.
- Ordering: this pass runs after Source-layer passes. It is not chained with `EncryptionPass` on the same artifact; see "Decided: alternative terminal strategies" below.
- Scope for a first version: a minimal instruction set sufficient to express ordinary PHP control flow and function calls, explicitly not attempting full PHP language coverage in one pass. Which subset is the first question to answer once this ADR is accepted, not before.

## Non-goals

- This ADR does not design the instruction set itself (opcodes, operand encoding, constant pool format). That is implementation work following from this ADR's acceptance, the same way ADR-0011 preceded `EncryptionPass`'s actual implementation.
- This ADR does not commit to a performance target. The roadmap already calls for benchmarks published alongside methodology; those benchmarks are what will show whether this design is viable, not a number asserted here.
- This ADR does not decide whether `bytecode-compilation` and `encryption` can run in the same edition. See open questions.
- This ADR does not decide which edition(s) activate this pass, the same way ADR-0011 separated the design decision from the later, separate activation decision for `EncryptionPass`.

## Decided (2026-09-10): `BytecodeCompilationPass` and `EncryptionPass` are alternative terminal strategies, not a sequential pipeline

The question was not simply "do they run in the same edition" or "do they run in sequence." It is a distinct architectural question from either: whether two protection techniques must compose into a pipeline merely because both happen to be passes.

`BytecodeCompilationPass` and `EncryptionPass` are alternative terminal protection strategies for a given artifact:

- **Encryption**: loader plus encrypted PHP payload, decrypted and executed at runtime (ADR-0011).
- **Bytecode**: a compiled artifact with its own execution mechanism, not layered on top of, or underneath, encryption.

An edition may expose both strategies as options, but a single protection operation on a given file selects one execution strategy, never both applied to the same artifact. This is deliberately weaker than either "always compose them" or "never allow both in one edition": it does not assume the two mechanisms are technically composable, and it does not collapse the product decision into a plain either/or, since an edition can still offer both, just not chained.

> `BytecodeCompilationPass` and `EncryptionPass` are alternative terminal protection strategies for an artifact. The architecture must not require them to execute together. An edition may expose both strategies, but a protection operation selects one execution strategy, unless a later technical decision demonstrates a valid, useful composition.

This leaves it to a future implementation to prove whether a safe, useful composition exists, rather than having this ADR pay that cost upfront by assuming one.

## Decided (2026-09-10): fail closed on unsupported constructs, scoped to passes that need total semantic coverage

For PHP constructs the instruction set cannot represent (reflection, `__call`/`__get`/`__set`, `eval`, dynamic callbacks, dynamically constructed class or method names, dynamic property access, and similar), `BytecodeCompilationPass` refuses to process the file rather than silently falling back to embedding the original PHP for the parts it cannot cover.

The rule stated generally:

> A protection pass whose guarantee depends on total semantic coverage must produce a result completely covered by its supported semantics. If it encounters a construct it cannot correctly transform or represent, it must fail explicitly rather than silently preserving the original code.

A silent fallback would produce a hybrid artifact, part transformed, part original, that may well still execute correctly but gives no honest answer to "how much of this artifact is actually protected," and a small, seemingly unimportant fragment can determine the behavior of the whole program. Fail-closed is not free: it means some real customer code will simply be rejected by this pass at least in an early version, and that rejection is deliberate rather than a defect to silently work around by degrading protection instead.

This rule belongs specifically to passes whose guarantee requires total coverage, `BytecodeCompilationPass` being the first, not to every pass in the pipeline as a blanket policy. A pass that can safely preserve a construct it does not deeply understand (a hypothetical future minification refinement, for instance) has no comparable reason to refuse; the obligation to fail closed follows from what a specific pass's guarantee actually claims, not from pass-hood in general.

Coverage may grow over time (supported constructs get transformed, unsupported ones fail, and the boundary between the two moves as the instruction set grows), but the fail-closed principle itself does not change as that boundary moves.

## Open questions

1. **Instruction set scope.** A minimal set covering arithmetic, control flow, and function/method calls is the obvious starting point. Given the fail-closed decision above, scope here is really a product/rollout question (how much real customer code can this pass accept in a first version) rather than a design-safety question; it does not need to be resolved before implementation begins, the way the previous two questions did.
2. **Benchmark methodology.** The roadmap asks for benchmarks "published alongside methodology," which implies the methodology itself needs review before numbers are published, not just the numbers. This ADR does not propose one.
3. **Threat model contribution.** ADR-0009 already states RunStack Ember's boundary (raises cost against casual/semi-automated attackers, not a dedicated reverse engineer). Whether bytecode interpretation materially raises that cost over the existing Source-layer passes, or mainly adds runtime overhead without a proportionate protection gain, should be stated once a first implementation exists to evaluate, not assumed here.

## Status

Proposed. The two architectural questions this ADR most needed settled before any code exists, execution-strategy composition and the coverage-failure policy, are decided above. Instruction set scope, benchmark methodology, and the threat-model contribution remain open, and per this project's own process (ADR-0003), none of this is announced as a product feature until an implementation and a test exist for it.
