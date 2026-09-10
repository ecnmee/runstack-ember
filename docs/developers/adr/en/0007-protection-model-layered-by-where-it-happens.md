# ADR-0007: Protection model, layered by where protection happens

* Status: Accepted
* Date: 2026-07-10
* Authors: RunStack Team
* Supersedes: none
* Superseded by: none

## Context

ADR-0006 defines protection techniques as passes in a pipeline, which answers *how* a technique is implemented and ordered. It does not answer a question that comes up every time a new technique is proposed: *where does this belong?* Without an explicit model, that question gets answered ad hoc, and the boundary between techniques drifts, the same way commercial tiers and technical implementation were tangled together in the previous implementation.

## Decision

Every protection technique belongs to exactly one of four layers, defined by when and where it acts on the application:

- **Source layer**: transformations applied to PHP source code before it ever runs, operating on the AST. Examples: minification, symbol renaming, string protection, control flow changes.
- **Intermediate layer**: transformations applied to a compiled or serialized representation of the code, after the source layer, before packaging. Examples: bytecode generation, virtualization.
- **Runtime layer**: checks and behavior that execute while the protected application runs. Examples: integrity verification, anti-debugging, license enforcement, environment binding.
- **Packaging layer**: how the protected artifact is assembled and distributed. Examples: encryption of the final payload, loader generation, PHAR building.

A pass declares which layer it belongs to. A technique that seems to span two layers is a signal that it should be split into two passes, one per layer, connected through the pipeline rather than merged into one.

## Consequences

- Anyone proposing a new protection technique states its layer before writing code, which surfaces design questions early (for example: "should this check run at packaging time or at runtime?").
- The four layers become the organizing structure for both the codebase (`src/Source`, `src/Intermediate`, `src/Runtime`, `src/Packaging`) and the documentation, replacing "Level N" as the primary way of describing the product.
- Commercial tiers remain a configuration on top of this model, per ADR-0006: a tier is a selection of passes across these four layers, not a layer or a level itself.

## Addendum (2026-08-24): what `Layer` describes, and the `loader-minification` case

This ADR's original text says every pass belongs to a layer "defined by when and where it acts on the application." That phrase was never made more precise than the four bullet definitions themselves, and `MinificationPass`'s two registry entries, `minification` and `loader-minification` (ADR-0012), are the first case where the imprecision matters: both registrations resolve to the same class, with identical `process()` behavior, transforming a representation that is syntactically PHP source either way. Nothing about the class itself changes between the two. Something else has to account for why they should carry different `Layer` values, if they should at all.

### The rule: position in the build sequence AND representation touched, not artifact type alone

"When" and "where" are two separate questions, and a rule based on only one of them is narrower than what this ADR's original text already committed to:

* **When**: has the protected program already been frozen into its distributed form (post-`EncryptionPass`), or not yet?
* **Where**: is the pass touching the protected program itself, or a wrapper/loader generated around it?

For every pass in this codebase today, both questions happen to agree, so a rule based on either alone would give the same answer as the combined rule would. That will not necessarily stay true. A hypothetical future pass could touch the same kind of representation (say, generated loader text) at two different points in the sequence for reasons that have nothing to do with packaging, and a rule based on "which artifact" alone would have nothing to say about that case. The combined rule does.

### Resolution: `loader-minification` is `Layer::Packaging`

Applying both questions to every pass currently registered, not only the one in question, shows the same rule holding consistently rather than being invented to fit one case:

| Registration | When | Where | Layer |
|---|---|---|---|
| `minification` | before the protected program is frozen | the protected program itself | `Source` |
| `encryption` | while the artifact is assembled | payload plus loader | `Packaging` |
| `loader-minification` | after the protected program is frozen | the loader `EncryptionPass` produced | `Packaging` |
| `integrity-verification` | while the protected application executes | the running application | `Runtime` |

`loader-minification` reporting `Layer::Source` was incorrect. It should report `Layer::Packaging`: it runs after the protected program is already frozen into an opaque payload, and it touches only the loader `EncryptionPass` generated, per this ADR's own Packaging examples ("how the protected artifact is assembled"). Shrinking the loader's boilerplate is part of assembling the final distributed artifact, in the same sense `EncryptionPass`'s loader generation already is.

To be explicit about the direction of that reasoning, since it is easy to state backwards: the classification follows from what the instance operates on and when, not from what it happens to be named. `loader-minification` is `Layer::Packaging` because that registration's instance transforms a Packaging-layer artifact; the name does not cause that classification, it exists so the configuration can say out loud what was already true. A registration named, say, `second-minification-pass` touching the exact same loader would carry the exact same `Layer::Packaging` value, for the exact same reason. Nothing about `layer()` should ever be justified by pointing at `name()`.

### Why this is not "a technique spanning two layers," and does not call for splitting the pass

This ADR's original text says a technique that seems to span two layers should be split into two passes, one per layer, rather than merged into one. `MinificationPass` used under two names looks, at first glance, like exactly that case. It is not, for a specific reason worth stating rather than leaving implicit: that guidance protects against a pass whose `process()` behavior itself branches on which layer it is currently participating in, typically by inspecting pipeline context to decide "am I running before or after X" and acting differently as a result. That is a real design smell, because it hides a layer-dependent decision inside a single class's control flow instead of surfacing it as two declared, separately reasoned-about passes.

`MinificationPass` does not do this. Its `process()` method has no branch on which name it was registered under and no branch on anything resembling "which layer am I in"; token-level whitespace and comment stripping is identical regardless of what representation happens to be handed to it. Splitting it into two classes now would produce two classes with byte-identical implementations, differing only in the metadata (`name()`, and now `layer()`) attached to them, which is duplication with no corresponding reduction in hidden branching, since there was never any branching to remove. One class, registered twice with distinct metadata per registration, is the better shape for this specific case. The splitting guidance stands for the case it was written for; this is not that case.

### Implementation is not decided here

This addendum settles the semantic question and the classification it resolves to. It does not decide how `layer()` should report a value that now needs to vary per registration rather than being fixed per class, the way `name()` already does via `MinificationPass`'s constructor override. A `layer()` implementation that switches on `$this->name` internally would make `name()` implicitly control architectural classification, an indirect coupling worth avoiding. The alternative (an explicit second constructor parameter, `new MinificationPass('loader-minification', Layer::Packaging)`, or an equivalent explicit mechanism) is a small design change, not merely a one-line fix, and is left for its own implementation step after this addendum is accepted, not decided as a side effect of it.
