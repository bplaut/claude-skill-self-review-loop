# The finding bar

Shared by the reviewer (what to raise) and the implementer (what to accept).
Every finding must do one of three things, and say which.

## 1. Name its trigger

The concrete input, file, state, command, or external response that reaches
the failure. Data and provider responses the code accepts but does not control
count even if nobody has observed them yet — the code cannot prevent them,
which is what fail-fast is for; a missing fail-fast check is a finding when
you can name the data that would slip through. The team's own future edits do
not count: "a future edit could reorder this" or "a fifth type might be added"
is not a trigger, because guarding against your own edits is the job of tests
and review, not of runtime checks or extra indirection. Do not raise those.

## 2. Simplify the code

Name what becomes easier to maintain, and how. The measure is complexity and
maintainability, not line count: fewer places that must change together,
fewer distinct mechanisms doing one job, fewer branches or states a reader has
to hold in mind, less indirection between a call and what it does.

- Adding helpers is an expected part of deduplication, not a cost to pay: a
  constant or helper that replaces duplication already in the diff (two
  identical loops, the same tuple spelled out three times) is the correct
  outcome. It is not fine when it exists to prevent duplication or drift that
  might happen later.
- The same rule applies in reverse: an existing check or branch whose only
  trigger is a future edit (an exhaustiveness `else` over a closed set, a
  guard against a misspelled literal) is dead weight, and deleting it is a
  simplification.
- Replacing one structure with another of equivalent complexity is not a
  simplification — judge that by writing the alternative out and comparing
  what a maintainer would have to understand and keep in sync, not by
  inspection.
- Replacing a hand-rolled copy of a behavior with a call to the shared helper
  that now provides it removes a second implementation and is a
  simplification whatever the length.
- Very short helpers that extract code that is easily understandable inline
  should not be created. An example of a bad helper is
  `def _is_priced(u): return u.cost_usd is not None`.

## 3. Quote documentation or a comment that is objectively false about the code

Explain why it is objectively false, rather than just ambiguous.

## Not findings

New features or generalisations beyond the spec's purpose; refactors of code
the diff did not touch; pre-existing problems outside the diff (one line,
marked out of scope, at most).

## Verification, per category (the implementer does this before accepting)

- Trigger: reproduce it — with real data, or with a constructed input the code
  would accept, a stubbed provider response if need be. A trigger you cannot
  demonstrate either way is not a trigger.
- Simplification: write the alternative out and compare what a maintainer must
  understand and keep in sync. Line count alone decides nothing.
- False documentation: check the quoted statement against the code. Ambiguous
  is not false.
