# ADR-0001: Product is a code protection platform, not an obfuscator

* Status: Accepted
* Date: 2026-08-24 (reconstructed; see note below)
* Authors: RunStack Team
* Origin: Reconstructed from implementation and project documentation
* Historical wording: Not preserved
* Supersedes: none
* Superseded by: none

**Note on this document's date and origin.** No original ADR-0001 document, dated at or near this project's earliest history, was found in either repository. What is recoverable is the decision itself, visible consistently in the codebase and its public-facing documentation, not the original deliberation or its exact wording. This document is written on the date above and states the decision as the project already enforces it. It does not claim to reproduce a historical document that has not been preserved, and where the evidence runs out, this document says so rather than filling the gap with invented history.

## Context

The product's own `composer.json` describes it as a "PHP code protection platform (source, intermediate, runtime and packaging passes)," not as an obfuscator, and the public README's tagline is "Beyond obfuscation." The package namespace is `RunStack\Ember`, with no trace of an `Obfuscator`-rooted name anywhere in the current codebase. ADR-0007 (protection model) organizes the codebase into four layers, Source, Intermediate, Runtime, Packaging, a structure that only makes sense for a platform with more than one kind of protection technique. ADR-0006 (editions are configuration) and ADR-0008 (asymmetric license verification) both presuppose licensing as a first-class product concern, not an afterthought bolted onto an obfuscator.

Beyond this, the specific circumstances that led to scoping the product this way, what it was called before, what was considered and rejected, are not recoverable from the evidence available.

## Decision supported by evidence

The product is **RunStack Ember**, described consistently across its own package metadata and public documentation as a code protection platform for PHP applications, not as an obfuscator. Every component that ships under this name maps to one of the concerns already visible in the codebase's own structure and terminology: obfuscation (the Source layer), encryption, licensing, runtime protection, integrity verification, build pipeline. The namespace, `RunStack\Ember`, carries no reference to obfuscation as the product's defining term.

## Rationale reconstructed from available evidence

A previous implementation, referenced by other ADRs (for example ADR-0002's account of a shared HMAC secret shipped inside distributed code) but not present in either repository examined for this reconstruction, is described elsewhere as having grown feature by feature without an explicit product scope. Scoping the current rewrite as a platform, rather than as a single tool, is consistent with that account and with the breadth of concerns the codebase and its ADRs already treat as core (ADR-0006 through ADR-0012 collectively cover editions, licensing, threat modeling, and a multi-layer protection model, not obfuscation alone).

Whether the product was ever formally scoped around a specific enumerated set of pillars, and if so what that set originally was, is not recoverable from the evidence available. What can be stated with confidence is that the current codebase and its public documentation consistently treat the product as more than an obfuscator, across every artifact examined.

## Consequences

- The README, package metadata, and public documentation describe a platform, not a single tool, consistently with the evidence above.
- New features are expected to map to a concern already visible in the codebase's structure (Source, Intermediate, Runtime, Packaging layers per ADR-0007; licensing per ADR-0008) rather than being justified against an enumerated list this document cannot confirm ever existed in a specific form.
- The namespace and package name (`RunStack\Ember`) carry no reference to obfuscation as the product's defining term, and any future naming decision should preserve that.
