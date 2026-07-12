# ADR-0009: Threat model

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

ADR-0003 requires that every marketing claim be backed by a specific, testable guarantee. That requirement is incomplete without a stated threat model, because "protection" is meaningless without saying who it protects against. Without this, it is easy to slide back into the kind of unqualified claim ("99% protection") that ADR-0003 exists to prevent, simply by describing a real feature in an unbounded way.

## Decision

RunStack Ember is designed to raise the cost of casual and semi-automated attacks. It explicitly does not claim to stop a sufficiently resourced, targeted attacker.

In scope, the product is designed to meaningfully slow down or block:

- Casual inspection of source code by someone with basic PHP knowledge and no specialized tooling.
- Automated static analysis and generic decompilers that are not built specifically for Ember's output.
- Unauthorized redistribution or reuse of licensed code by end users of the protected application.
- Casual license tampering (editing a license file, bypassing an obvious check).

Out of scope, the product does not claim to stop:

- A dedicated, technically skilled attacker with unlimited time, willing to manually reverse engineer the runtime and any protection passes applied to a specific build.
- An attacker with root or physical access to the machine running the protected application at the moment it decrypts and executes.
- Side-channel attacks against the PHP runtime or the underlying operating system.

Any feature or marketing claim states which side of this boundary it falls on. Claims are written in terms of the attacker they are designed to stop, not in terms of an abstract percentage.

## Consequences

- Documentation and marketing copy describe capabilities against a stated adversary ("stops automated static analysis tools that are not Ember-aware") instead of unqualified strength claims ("99% protection").
- Sales conversations can set correct customer expectations up front, which reduces the credibility risk that motivated ADR-0003 in the first place.
- Future protection passes are evaluated against this threat model: a proposed feature that only helps against an attacker already out of scope is deprioritized in favor of ones that raise the cost for attackers in scope.
