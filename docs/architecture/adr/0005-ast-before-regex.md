# ADR-0005: AST before regex

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

Structural transformations of PHP source code (renaming, control flow changes, dead code injection) are fragile when implemented with regular expressions, because PHP's grammar is not regular and edge cases (heredocs, nested closures, attributes, match expressions) break naive text substitution silently. The previous implementation already used `nikic/php-parser` for its earlier passes (minification, identifier renaming, string obfuscation), which is the correct approach and should be the standard for everything structural, not just those passes.

## Decision

Every structural transformation of PHP source code goes through an AST produced by `nikic/php-parser`. Regular expressions are reserved for simple textual tasks that do not require understanding PHP's grammar, such as stripping comments from already-tokenized output or working with non-PHP text like license key formatting.

Any new pass that touches code structure is implemented as an AST visitor, not a string transformation.

## Consequences

- Slightly more upfront code per pass compared to a regex-based shortcut.
- Consistent handling of edge cases across all passes, since they share the same parser and node visitor infrastructure.
- New PHP syntax (enums, readonly properties, first-class callable syntax) is supported by upgrading the parser dependency, rather than patching regex patterns across the codebase.
