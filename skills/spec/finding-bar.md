# The finding bar for a spec

Shared by the reviewers (what to raise) and the author (what to accept).
The object under review is a spec: a design that is not built yet. A
finding is a defect in what the spec says, or fails to say, that would
make the implementation come out wrong, lose or double-count data or
spend, or make a number the project reports mislead.

A **[Blocking]** finding must meet one category below and say which.
**[Non-blocking]** findings need no category: wording, a typo, a clearer
example, a small observation.

## Any role

- **A1. Contradiction.** Two parts of the spec, or the spec and its
  research spec, cannot both hold. Quote both.
- **A2. Ambiguity.** Two readings a careful implementer could take that
  lead to materially different code, data on disk, results or output.
  Name both. Unclear wording that admits only one sensible reading is
  Non-blocking.
- **A3. A false or unsupported claim** about the existing code, the data,
  a tool or library, or prior results, including a false reason given for
  a decision. Quote the claim and show what is true.
- **A4. A hidden research decision.** A default or reading that changes
  what a reported number means, or which records it rests on, that the
  spec settles without listing it as a decision or an open question. Say
  what it changes.
- **A5. Unfaithful to its research spec.** When the spec implements a
  research spec, it produces something that document does not define,
  omits something it requires, or contradicts it. Quote both.

## Research

- **R1. The design cannot deliver its claim.** Name the case that breaks
  the spec's research claim: a confound, a selection effect, a measure
  that misses its construct, conditions that differ in more than the
  variable under study. A concrete, plausible scenario is enough; show it
  on existing data where that is cheap.
- **R2. A reported number would mislead.** Name the chart, table or
  statistic, the comparison a reader would draw, and why the data does
  not support it.
- **R3. An unstated threat changes how a number is read.** Name the
  number, the alternative reading the spec does not acknowledge, and the
  mechanism behind it with its support (code, data or published work).
- **R4. Prior work changes the plan.** Cite it (authors, title, year,
  link), having opened it, and say what in the spec it changes.
- **R5. A pilot cannot answer its own question.** A decision the spec
  says a pilot or first run will inform, for which the planned outputs
  give no usable evidence. Say what is missing.

## Engineering

- **E1. The mechanism does not do what it claims.** A step as specified
  would produce wrong results, lose or double-count data, or fail. Give
  the concrete input, file state or sequence of commands that breaks it.
  Inputs the system accepts but does not control (provider responses,
  external data) count.
- **E2. A gap.** Something the goal requires that no decision covers: a
  state of the files on disk (interrupted, partial, errored, stale) that
  no rule handles; a reader of the outputs the plan does not mention; a
  doc that would go stale; a test the project's rules require; an
  existing test broken without saying so; a commit that would not stand
  on its own.
- **E3. A project rule broken.** An invariant or rule in the project's
  `CLAUDE.md`, a module docstring, a check in the code, or a rubric for
  this kind of document. Name it and how the plan breaks it.
- **E4. A materially simpler plan.** Write it out. It must remove a
  command, a stage, a record type, a mechanism, an option, or a place
  that must change together, and still meet the goal; or show a part of
  the plan whose cost the goal does not justify. Fewer words alone is not
  enough.

## Something else (rare)

A major problem outside every category above. Label it [Blocking] and say
why it is major and why no category fits.

## Not findings

- New scope: extra conditions, controls, models, statistics, analyses,
  validation runs, options or features beyond the spec's goal. If one
  matters, give it a line under "For the owner".
- A different design you would prefer. The chosen design is reopened
  only through E4 or a concrete failure under another category.
- Checks, tests or indirection whose only trigger is a future edit by the
  team (another model, another benchmark, a later template change), or a
  state the team's actual workflow cannot produce. Cheap checks that
  detect a malfunction are not machinery; do not flag them as such.
- A wrong state that lasts only until the user's normal workflow clears
  it (rerunning the steps in order). Documenting it is enough
  (Non-blocking at most).
- A problem that predates the change and that the change does not make
  worse: one line under "For the owner".
- A risk with no support in the code, the data or published work, and
  standard practice or an obvious point the spec can take as given.
- Data analysis for its own sake, and drift in data that is still being
  written, unless it changes a conclusion the spec draws.
- Re-arguing a settled point without the argument the brief asks for.
- Exact values the spec marks provisional, unless the value breaks
  something.
- Detail below the spec's level: names, layout, formats or style that
  change no data and no behaviour and that an implementer can choose.
- The wording of text the owner wrote and keeps as their own, such as a
  prompt they drafted or a stub left for them to write: observations go
  under "For the owner". A defect in that text under a category above (a
  contradiction with a ruling or the rest of the spec, a class of inputs
  a prompt would label wrongly) is a finding, marked as needing the
  owner's decision.
- More tests than the change needs (one or two focused tests per
  behaviour), and tests that pin presentation: printed wording, titles,
  labels, colours.

## Verification, per category (the author, before accepting)

- A1, A5: read both texts.
- A2: write both readings out and say what each would produce.
- A3: check the claim against the code at the spec's base, the data, or
  the tool's source.
- A4: confirm the spec settles it and that the alternatives would change
  a number.
- R1, R2, R3: trace the case through the design; recompute any numbers.
- R4: open the cited work.
- R5: match the decision against the planned outputs.
- E1, E2: construct the input or file state and trace the spec's rules
  through it.
- E3: read the rule where it is stated.
- E4: write the alternative out and compare what an implementer and a
  maintainer must understand and keep in sync.

## The fix

The best fix is usually the smallest: correct, tighten or delete text
before adding any. A proposed fix is judged apart from its finding; a
real defect can come with a wrong fix.
