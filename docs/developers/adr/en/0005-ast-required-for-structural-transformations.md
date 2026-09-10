# ADR-0005: An AST is required for structural transformations

* Status: Accepted
* Date: 2026-08-24 (reconstructed; see note below)
* Authors: RunStack Team
* Supersedes: none
* Superseded by: none

**Historical note.** The original ADR document was not preserved. The pairing of this number with this decision is corroborated independently, not inferred once: `SymbolRenamePass`, `ControlFlowPass`, `StringProtectionPass`, and `AbstractAstPass` each cite "(ADR-0005)" directly, in their own docblocks, all describing the same rule. `MinificationPass` cites it too, to explain why it is the one pass that does not follow it. What is not preserved is the original historical wording, only the decision, corroborated from five independent citation sites.

## Decision supported by evidence

A pass that structurally transforms PHP code, meaning it changes what the code means or how symbols relate to each other, must operate on an AST produced by `nikic/php-parser`, not on tokens, not on regular expressions, and not on any other text-level representation. `SymbolRenamePass`, `ControlFlowPass`, `StringProtectionPass`, and `IntegrityVerificationPass` all extend `AbstractAstPass`, which owns the shared parse-transform-print sequence, specifically so this rule is enforced structurally rather than by convention alone.

`MinificationPass` is the documented, deliberate exception: per its own docblock, "ADR-0005 requires an AST for structural transformations; minification is not structural, it never changes what the code means, only how much whitespace surrounds it." It uses PHP's own tokenizer instead, because the rule this ADR states is scoped to structural change, and minification does not structurally change anything.

## Reconstructed rationale

Regular expressions and naive text manipulation get PHP's own syntax wrong in cases a hand-written pattern will not anticipate: string literals that contain code-like text, heredocs, nested braces, and namespace-qualified names are the concrete cases already handled correctly elsewhere in this codebase specifically because an AST was used instead. Why this project settled on `nikic/php-parser` specifically, rather than some other parser, is not recoverable from the evidence available; only that every AST-based pass in this codebase uses it, consistently.

## Consequences

* Any future pass whose job is to change what code means, not merely how it looks, needs a stated reason if it proposes not extending `AbstractAstPass` and not working from a real AST. Minification-style exceptions exist, and are legitimate, but they are exceptions to be argued for individually, the way `MinificationPass`'s own docblock already does, not a default.
* This is also why `EncryptionPass`, which replaces the entire file with an opaque loader rather than transforming the meaning of code within it, was classified as not needing to extend `AbstractAstPass` either (ADR-0011): it is not doing AST-level structural transformation of the protected program, it is replacing the whole representation, a different kind of operation this ADR's scope was never meant to cover.
