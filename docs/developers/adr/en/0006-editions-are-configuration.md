# ADR-0006: Editions are configuration, not code

* Status: Accepted
* Date: 2026-08-24 (reconstructed; see note below)
* Authors: RunStack Team
* Supersedes: none
* Superseded by: none

**Note on this document's date.** This ADR is cited by name throughout the codebase (`src/Editions/free.php` and every other edition file: "Editions are configuration, not code (ADR-0006)") and by `PassRegistry`'s own docblocks, but the original document was never found in either repository. This is a reconstruction of the decision those citations already describe consistently, written on the date above, not a claim that this exact text existed earlier. Where the original wording is unknown, this document states the decision as the code already enforces it, not as invented history.

## Context

Before this decision (as later ADRs, including ADR-0007, describe it), commercial tiers and technical implementation were tangled together. Adding a protection technique to a tier meant touching code that also decided which customers could use it, and the two concerns were hard to separate or reason about independently.

## Decision

An edition is a plain, ordered list of pass names, nothing else:

```php
return [
    'symbol-rename',
    'string-protection',
    'control-flow',
    'minification',
    'encryption',
    'loader-minification',
];
```

Every edition file in `src/Editions/` (`free.php`, `basic.php`, `premium.php`, `enterprise.php`) follows exactly this shape: a PHP file returning a `list<string>`. It contains no class instantiation, no conditional logic, no reference to which class implements any pass it names. `PassRegistry` (see `PassRegistry::withDefaults()`, ADR-0012) is solely responsible for resolving each name to a concrete `Pass` instance; `Edition::fromFile()` is solely responsible for loading and validating the list itself (existing file, returns an array, every element a string).

This split means an edition can be read, diffed, and reasoned about by anyone, without needing to know a single line of PHP beyond what a list of strings means, and a pass's implementation can change entirely (as `EncryptionPass` and `MinificationPass` already have) without touching a single edition file.

## Consequences

* A commercial tier is a selection of pass names, never a place where protection logic itself lives. Adding "does this tier get encryption" is a one-line edit to a list, not a code change to any pass.
* `PassRegistry` rejects a name it does not recognize (`OutOfBoundsException`) and a duplicated name within one edition (`LogicException`, `assertNoDuplicates`), so a malformed edition fails loudly at build time rather than silently doing less than intended.
* This is why `MinificationPass` needed a second registered name (`loader-minification`) rather than a special case inside any edition file when it started running twice in the encrypted pipeline (ADR-0011's addendum, ADR-0007's addendum): the edition format has no room for anything other than a flat list of names, by design, so the second use had to be a second name, not a parameter or a flag.
