# ADR-0003: Marketing follows implementation

* Status: Accepted
* Date: 2026-08-24 (reconstructed; see note below)
* Authors: RunStack Team
* Supersedes: none
* Superseded by: none

**Note on this document's origin.** This ADR is quoted almost verbatim in the project's earlier public README: "Per ADR-0003, a capability is documented here only once it has a working implementation and a test that exercises the specific claim, so this README will grow slowly and deliberately rather than all at once." The number and this specific sentence appear together, which is stronger evidence than most of the reconstructed decisions in this set. The surrounding structure (Context, Consequences) below is written to match this project's ADR format; the decision itself is this project's own words, recovered, not invented.

## Context

It is easy for a project's public-facing documentation, especially a README meant to represent the product, to describe capabilities as though they already exist because they are planned, in progress, or merely plausible. Once that happens once, every future claim in the same document becomes suspect: a reader has no way to tell which sentences describe working software and which describe intent.

## Decision

A capability is documented in this project's public-facing material only once it has a working implementation and a test that exercises the specific claim being made. Not "the feature is planned," not "the architecture supports it," not "it should work": a passing, specific test, first.

This is why the current public README does not claim a CLI exists until `bin/ember` and `EmberCliTest` (ADR-0012) both do, does not claim `encryption` is available to every edition until each edition's own file says so and an integration test exercises it, and does not claim licensing is enforced (ADR-0008) while `src/Licensing/` remains an empty, reserved directory.

## Consequences

* Public documentation grows slowly and deliberately, in step with what is actually built, rather than all at once ahead of it.
* A stale or aspirational claim in a README or other public document is treated as a documentation bug, the same category of problem as a stale ADR classification (see ADR-0007's addendum), not a harmless simplification.
* This does not forbid discussing plans, roadmaps, or direction. It specifically constrains claims of present capability: a roadmap section that says "planned" is not violating this decision; a features list that omits the word and lets a planned capability read as a shipped one is.
