# ADR-0007: Protection model, layered by where protection happens

## Status

Accepted

## Date

2026-07-10

## Authors

RunStack Team

## Supersedes

—

## Superseded by

—

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
