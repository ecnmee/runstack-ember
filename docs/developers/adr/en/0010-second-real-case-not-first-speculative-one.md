# ADR-0010: Build for the second real case, not the first speculative one

* Status: Accepted
* Date: 2026-08-24 (reconstructed; see note below)
* Authors: RunStack Team
* Supersedes: none
* Superseded by: none

**Historical note.** The original ADR document was not preserved. The pairing of this number with this decision is corroborated independently across four separate files: `Representation.php`, `Edition.php`, `SourceCode.php`, and `AbstractAstPass.php` each cite "ADR-0010" directly, describing the same principle in each case. What is not preserved is the original historical wording, only the decision, corroborated from four independent citation sites, and consistently applied in every architectural decision made during this project's later work as well.

## Decision supported by evidence

An abstraction, an interface member, a configuration field, or a structural layer is introduced when a second real, concrete need for it is known, not when it is first imagined as plausibly useful. Waiting for a second case, not building for a first speculative one, is the rule; four independent sites in the codebase apply it explicitly:

* `Representation`, the marker interface every pipeline payload implements, is deliberately empty: "Concrete representations... are introduced once each layer has its first real pass and their actual shape is known, instead of being designed speculatively today."
* `Edition` carries only a name and a pass list today: "Additional fields (version, feature flags, limits) are added here once a real edition needs them, rather than speculatively now."
* `SourceCode`, the first concrete `Representation`, was "introduced now because MinificationPass, the first real Source layer pass, needs it, not before."
* `AbstractAstPass` itself was "introduced after four concrete passes had already implemented this exact parse/transform/print sequence independently, not before... four was well past that bar."

## Reconstructed rationale

Speculative structure, built for a need that has not yet appeared, tends to guess wrong about the shape that need eventually takes, and the guess has to be unwound or worked around later. Waiting for a second concrete instance before generalizing means the abstraction is built from two real examples instead of one imagined one, which this codebase's own history bears out directly: `AbstractAstPass` did not exist until four separate passes had already, independently, implemented the same parse-transform-print sequence, at which point the shared shape was no longer a guess.

The exact threshold ("second case," specifically, rather than third or fifth) is not separately justified anywhere recoverable; it is simply the number used consistently across every citation found.

## Consequences

* This principle was applied repeatedly in work on this project after this reconstruction effort began, though not always with this ADR number cited explicitly at the time: the decision not to introduce a facade class for the entry point in ADR-0012 ("build the abstraction when a second concrete need for it appears, not preemptively"), and the decision not to open a Runtime Layer implementation in ADR-0011's addendum until a second concrete need for shared runtime behavior appears, both restate this same rule in different words.
* A reviewer proposing a new abstraction, interface, or configuration field should be able to name the second concrete case that motivates it, not only the first. "This might be useful later" is not, by itself, sufficient justification under this decision.
* This is a general project default, not an absolute rule without exceptions: a case for building ahead of the second concrete need should be argued explicitly, on its own terms, rather than assumed to be forbidden outright.
