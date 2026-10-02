---
name: review-loop
description: Implement a spec file, then loop with a fresh independent reviewer subagent (running /code-review) until the change is clean, triaging and logging every finding and escalating scope decisions to the user. Use only when the user runs /review-loop <path-to-spec-file>.
argument-hint: <path-to-spec-file> [--rounds N] [--auto]
---

# review-loop

You are the **implementer**. You build the change described in the spec file,
then repeat rounds of independent review and triage until the change is clean.
The reviewer is always a freshly spawned subagent so it never sees your
reasoning — only the spec, the diff, and the decision log.

Arguments: `$ARGUMENTS` — the spec file path, optionally followed by
`--rounds N` and/or `--auto`.

Constants (tweak here):

- `ROUND_CAP` = `N` if `--rounds N` was given, else `4`
- `AUTO` = true if `--auto` was given, else false. In auto mode the loop
  never pauses for a ruling mid-run: every escalation is logged with a
  provisional decision and the loop keeps going, then presents all open
  questions in one batch when it would otherwise exit or hits the cap. See
  "Auto mode" under Escalation.
- `REVIEW_LEVEL = high`
- `RUN_DIR` = the directory containing the spec file. Everything this run
  produces — the decision log, the PR draft, anything else — goes here, so
  each invocation keeps its own records.
- `LOG = RUN_DIR/decisions.md`
- `FINDING_BAR` = `finding-bar.md` beside this file (absolute path
  `~/.claude/skills/review-loop/finding-bar.md`). The categories a
  finding must meet, shared verbatim by reviewer and implementer so the two
  cannot drift apart.

## What matters most

Everything below serves these four. When in doubt, weigh them above the
procedural detail.

1. **The reviewer is independent.** A new subagent every round, given only
   the spec, the diff, and the decision log — never your reasoning.
2. **Two channels only.** Anything the reviewer might need lives in the spec
   or the decision log. Conversation with the user that the spec omits goes in
   the log.
3. **You judge; the reviewer advises.** Accept a finding only if it names a
   concrete trigger, makes the code simpler, or quotes a doc that is
   objectively false; a plausible reason is not enough. Every accept and
   every reject carries a reason.
4. **Scope is the user's.** Neither you nor the reviewer expands or cuts the spec.
   When a decision is theirs, stop and ask; ending your turn is what pauses the
   loop, and the log on disk is its state. Research decisions — anything that
   changes what a result number means — are the most important of these.

## Phase 0 — setup (fail fast)

Run these checks before writing any code. If any fails, stop and tell the user
exactly which one and why; do not try to fix the repo state yourself.

1. The first token of `$ARGUMENTS` is a path to an existing, non-empty
   file. Read it in full. If `--rounds` is present, `N` must be a positive
   integer; `--auto` takes no value; anything else in `$ARGUMENTS` is an
   error. Every check in this phase applies in auto mode too: a failed
   check or an insufficient spec stops the run, since guessing there would
   poison everything after.
2. **Is this a resume?** If `LOG` already exists, tell the user and ask whether
   to resume from it or start over; never silently overwrite it. On start
   over, rename the old log to `decisions-<YYYYMMDD-HHMM>.md` in `RUN_DIR`
   so the earlier run's record survives, then continue as a fresh run. On
   resume, see "Resuming" below; the remaining checks change meaning.
3. No git operation is in progress: for each of `rebase-merge`,
   `rebase-apply`, `MERGE_HEAD`, `CHERRY_PICK_HEAD`,
   `test -e "$(git rev-parse --git-path <name>)"` fails. Ask git for the
   path: in a linked worktree `.git` is a file, so a literal
   `.git/<name>` never exists.
4. Tree state. **Fresh run:** no tracked file is modified or staged —
   `git status --porcelain` shows only `??` lines. Untracked files never
   enter `git diff <BASE>`; an uncommitted edit would. A dirty tree is
   likely another session's work — surface it, never revert it.
   **Resume:** a dirty tree is expected — it is this run's interrupted
   work. Show the user `git status` and `git diff --stat`, and ask whether to
   keep or discard it; do not stop.
5. Nothing the run writes to `RUN_DIR` can enter the reviewed diff. That
   needs checking only when `RUN_DIR` is inside the current checkout:
   `git -C RUN_DIR rev-parse --show-toplevel` prints the same path as
   `git rev-parse --show-toplevel` (run both without changing directory).
   Then `RUN_DIR` must not be the top level — `git -C RUN_DIR rev-parse
   --show-prefix` is non-empty (a spec there would put the log in the root
   and make the next check meaningless) — and must be gitignored —
   `git -C RUN_DIR check-ignore -q .` succeeds; if not, hand the user the
   one-line `.gitignore` edit. Anywhere else (outside git, another
   repository, another checkout) needs no check.
6. **The spec is sufficient** (fresh run only; on resume it was checked
   before). Scope disputes are settled by pointing at the spec, so it must
   carry enough to point at. Check that it states:
   - the goal — what should be true when the change is done;
   - the scope — what is being built, and which files or components it
     touches;
   - **non-goals** — what it deliberately does not do, so "out of scope"
     means something;
   - anything fixed rather than free — interfaces, names, formats, flags
     that must come out a particular way;
   - how to tell it works: if the repo has a test suite, which tests the
     change adds — the spec decides what gets a test, and the reviewer holds
     the diff to that list.
   Also read for ambiguities where the readings lead to materially
   different code. Missing non-goals is the most common gap ("add retry
   logic" doesn't say whether backoff tuning or a metrics counter is in
   bounds). If anything is missing or ambiguous, **stop and discuss it with
   The user** before writing code: list each gap with what you would assume.
   Write the answers **into the spec file itself** and re-run this check.
   This is a judgement, not a form — don't pad a spec that is already
   sufficient.

Then, on a fresh run:

- **Make a new branch** from the current `HEAD`: `git switch -c <name>`,
  with a name for the change that does not already exist, in the style of
  the repo's existing branches. Always a new one, whatever was checked out,
  so the run's commits sit on a branch of their own; if you started on a
  branch other than the default, the new one stacks on it.
- Record the base commit: `BASE=$(git rev-parse HEAD)`. Every review and
  the exit restructure are relative to this commit; commits before it are
  never touched.
- **Tests, once.** If the repo has a test suite (its `CLAUDE.md` or
  `README.md` says how to run it), run it now and note the result; if it is
  already red, every later "run the tests" means "no new failures", and the
  reviewer needs to know which failures it inherited. If there is no suite,
  record "no suite": "run the tests" then means the hand checks the spec
  names, and nothing in this skill asks you to create a suite.
- Create `LOG` with a header recording the spec path, the branch and the
  one it was made from (`detached` if none), `BASE`,
  the start time, the mode (`auto` or `interactive`), the test status at
  `BASE`, and the untracked paths present
  now (`git status --porcelain | grep '^??'`, so your own new files can be
  told from pre-existing ones later). Then a `## Context` section listing
  every decision from your conversation with the user that the spec does not
  state — scope cuts, non-goals, deliberate simplifications, files that are
  out of bounds — or "none".

**Resuming.** The spec and the log are your only state (a fresh session has
no memory of the last one). `BASE` and the branch are the values in the log
header, **not** the current `HEAD`; if you are not on that branch, stop and
tell the user. Find the last `## Round N` heading and re-enter at its first
incomplete step:

- no round heading → Phase 1 unfinished; compare `git diff <BASE>` to the spec
- heading but no pasted reviewer message → spawn round N's reviewer
- message but no decisions → triage
- an `[escalate]` entry with no matching `- ruling` line → in interactive
  mode the loop paused on a question; take the user's answers from the
  current conversation, or ask again, record them, then continue that
  round. In auto mode an open escalation with a `provisional:` line is not
  a pause; continue with the round's next incomplete step
- decisions but no `applied, committed` → apply and commit
- `applied, committed` but no `outcome:` line → decide Phase 4 and write it
- `outcome: repeat` → start round N+1; `outcome: exit` → the exit steps
- `outcome: pause` (auto mode) → the batch is awaiting rulings. If the
  rulings are in the current conversation, record and apply them and
  continue per "Auto mode"; otherwise present the batch again

The log is **append-only** and has a fixed shape. Nothing parses it; a
fresh session reads it to find its place, and a predictable shape makes
that fast. Never insert into the middle of the file. One exception: when a
later round's accepted fix or a ruling supersedes an entry under
`## Implementation decisions` (or an earlier `- decision:` line), append
` — superseded, see round N` to the end of that entry. The original text
stays; the suffix stops the next reviewer, who reads the header before the
rounds, from re-reporting a decision the code no longer follows.

~~~
# review-loop decision log
- spec: <path>   - branch: <name> (from <branch>)   - BASE: <sha>   - started: <time>
- mode: auto | interactive
- tests at BASE: <pass/fail summary, or "no suite">
- untracked at start: <paths>
## Context
## Implementation decisions        <- Phase 1 only; complete before round 1
- <decision> — <reason> [— superseded, see round N]   <- the suffix is the one edit allowed later
## Round N
review started <time>
```<reviewer's final message, verbatim>```
- [accept|reject|escalate|defer] <file:line> — <summary>
  reason: <one sentence>
  provisional: <what you did meanwhile> <- auto mode, [escalate] entries only
- decision: <what and why>          <- a design choice made while fixing
- ruling <file:line>: <the user's answer> <- appended when they answer; never edit the entry above
- for the user: <research observation or question the reviewer raised> <- information, not a decision; collected at exit
tests: <summary>                    <- no new failures vs the header, or "no suite"
applied, committed <time>
outcome: repeat | exit | pause — <reason>   <- written before the round ends; pause is auto mode only
## Result                           <- exit; collects every [defer] line
~~~

## Phase 1 — implement

Build the change as you normally would, following every rule in the
project's `CLAUDE.md`, and run its tests.

**The spec and the log stay out of the code.** Docstrings, comments, tests,
repo docs and commit messages never cite the spec or `LOG` by their own
labels — "Decision 3's variance", "per finding 2", "the round 1 fix" — here
or when applying fixes in later rounds. A reader of the code has neither
file, so the reference explains nothing. When a decision shapes the code,
write its substance (the formula, the rule, the reason) where the code is.

**Commit as you go.** Commit whenever a coherent piece lands so progress is
tracked. The history is restructured once, at exit, so intermediate commits
need only an honest message.

Any design decision you make during implementation that the spec does not
fix goes in `LOG` under `## Implementation decisions`, one line each with the
reason, so the reviewer can judge the diff against it. This is the one
deliberate exception to "the reviewer never sees your reasoning": it sees
the decisions and their stated reasons, not the deliberation behind them.
That section is complete before round 1; a decision made later, while
applying fixes, goes under its round as a `- decision:` line.

If you hit an escalation trigger (see below), escalate before continuing on
that point; finish everything independent of it first — unless the question
is load-bearing for what follows, in which case ask early rather than build
on an assumption the user may reverse. In auto mode you cannot ask early:
log the escalation with a `provisional:` line naming the reading you built,
and prefer the reading that is cheapest to undo.

## Phase 2 — review

Before spawning:

1. Commit everything: no tracked file modified or staged, and no untracked
   paths beyond those recorded in the log header — an uncommitted new file
   is invisible to `git diff <BASE>`. The commit structure need not be
   final; the reviewer reads the diff, not the commits.
2. Append `## Round N` to `LOG` with a line `review started <time>`, so an
   interrupted review leaves a trace to resume from.

Spawn a reviewer with the `Agent` tool: `subagent_type: "general-purpose"`.
**Never `fork`** — fork inherits your context, which defeats the purpose.
**Never reuse a previous round's reviewer** via SendMessage — a new agent
each round; the decision log carries forward what needs remembering.

The reviewer prompt must contain exactly this information and nothing about
your own reasoning, plan, or self-review:

```
You are an independent code reviewer. You must not edit any file or run
any git command that changes state, and every helper agent you spawn must
be told the same — pass this constraint into each helper's prompt.

You are a subagent: your final message is the only thing the parent
session sees, and your turn ending is final — nothing re-invokes you.

1. Read the spec at <spec path>. The diff is meant to implement it.
2. Read the finding bar at <FINDING_BAR>. Every [Blocking] finding you
   raise must meet one of its categories and say which; [Non-blocking]
   findings need no category.
3. From the decision log at <LOG>, read **only** the header, `## Context`,
   `## Implementation decisions`, and every `- ruling` line. Do not read
   the earlier rounds' findings or triage yet — you form your own view of
   the diff first, and read theirs afterwards (step 6). Context and
   implementation decisions tell you what the implementer chose and why;
   a finding that would change an implementation decision is an ordinary
   finding — name the decision it touches and go on. An entry ending
   "superseded, see round N" is history, not the current design; do not
   report the code for disagreeing with it. Rulings are the user's
   (the human) and are final unless you can say why the ruling's reason is
   wrong.
4. Run the /code-review skill at level <REVIEW_LEVEL> (invoke the Skill
   tool with skill "code-review" and args "<REVIEW_LEVEL>"). Restrict the
   review to the diff since <BASE>: `git diff <BASE>`. Do not review
   commits before <BASE>. The commit structure since <BASE> is
   work-in-progress and is restructured before merge; review the diff, not
   the commits or their messages.
   The code-review skill will have you spawn helper agents (finders, then
   verifiers). You have no later turn in which to receive background
   results, so **spawn every helper with `run_in_background: false`** and
   read each result before moving on. Never end your turn while a helper is
   still running — "waiting for the finders" is a failed review.
5. Also check the diff against the spec and the implementation decisions:
   anything the spec requires that the diff does not do, or does
   differently without a logged decision, is a finding. If you think the
   spec itself is missing something, raise it as a **spec gap** marked
   "needs a human decision", not as a fix for the implementer to just make.
   Before anything else in the review, sweep the diff for duplication it
   introduces: added line sequences that appear at more than one site, the
   same literal or key list spelled out in several places, a hand-rolled
   copy of a behavior a helper in the diff already provides. Duplication a
   diff adds is visible in the diff itself, so list every instance now
   rather than leaving some for a later round to find.
6. **Now** read the earlier rounds in <LOG>: their findings, the
   implementer's accept/reject/escalate decisions and reasons. For each of
   your findings, check whether it was already decided. A rejection you
   agree with: drop the finding. A decision you think was wrong — a
   rejection, an acceptance, or a ruling — keep the finding, mark it
   "reverses round N decision on X", and say why the logged reason is
   wrong; such findings go to the user, so make the case fully. A re-raise
   without that argument is noise; do not include it. An `[escalate]`
   entry with no `- ruling` line is pending the user, not the implementer:
   do not re-argue it, and mention it only if you have new evidence the
   entry does not already state.
7. Your final message must list every finding as plain text, most severe
   first. The parent session cannot see the ReportFindings tool output, so the
   code-review skill's instruction not to repeat findings as text does not apply
   here — repeat them. For each finding give: file:line, a one-sentence summary,
   its trigger / what it makes easier to maintain and how / why the
   documentation is false, whether it was CONFIRMED or only PLAUSIBLE (this
   records whether the code does what you say, not how much it matters), whether
   it reverses a logged decision, and whether it needs a human decision. Also
   include a "Considered but not raised" section, and a "For the user"
   section for anything you noticed about the research itself rather than
   the code: a confound, a definition that changes what a reported number
   means, a pattern in the data, a question the spec does not answer. These
   are not findings against the diff and need no category; state each with
   its evidence. Leave the section out only if it is empty. The first line
   of your message must be exactly `VERDICT: NOT APPROVED` or
   `VERDICT: APPROVED` — nothing before it.
   - NOT APPROVED — at least one [Blocking] finding: a behavior defect or
     robustness issue with a trigger, a diff-vs-spec gap, or a nontrivial
     simplification (one that introduces, merges, or restructures
     code). One-site edits such as deleting dead code do not block approval
     because you can trust that the implementer will address it without
     re-review. Similarly, documentation and naming issues should be reported
     but generally do not block approval. You may also block approval for a
     really important issue outside these categories, sparingly — if you do,
     label that finding [Blocking] and say why it blocks. Label every finding
     [Blocking] or [Non-blocking].
   - APPROVED — every finding is [Non-blocking]: naming, wording,
     documentation fixes, etc. Non-blocking findings don't need a category.

Your work is very important to the user's project and is greatly appreciated.
```

Read the verdict from the `VERDICT:` line; accept it anywhere in the first
five lines, since a reviewer sometimes adds a preamble. If no such line
exists — it only called ReportFindings, or it ended its turn "waiting" on
agents it had spawned — stop and tell the user what came back. Do not spawn a
fresh reviewer and do not proceed on an empty review.

## Phase 3 — triage

First paste the reviewer's final message **verbatim** into `LOG` under the
round's heading, in a fenced block. The user and later reviewers must be
able to see what was actually said, not your paraphrase of it.

Then, for each finding — including the non-blocking items under an APPROVED
verdict — decide **accept**, **reject**, **escalate**, or **defer**, and
append the decisions below the pasted message **before** changing any code:

```
- [accept|reject|escalate|defer] <file:line> — <finding summary>
  reason: <one sentence>
```

Rules:

- **The acceptance test.** Accept a finding only if all of these hold:
  1. For the three main categories, you verified it by that category's method
     (the file's last section): reproduced the trigger, wrote the simplification
     out and compared, or checked the quoted doc against the code. If the
     finding does not belong to these categories, use your best judgment.
  2. Its justification is not a future edit someone might make — that fails
     however plausible the reason attached; reject it, saying so.
  3. You agree it is an improvement. The reviewer has fresh eyes, not
     authority: form your own opinion. Every suggestion arrives with a
     reason, and accepting on "has a reason" is how a small change
     accumulates guards against hypotheticals nobody asked for.
- **A Blocking finding is accepted or escalated, never rejected on your
  own authority.** If you would argue against one, write the argument as
  your recommendation and escalate it. You can reject Non-blocking findings without
  escalation, although escalation is preferred when the decision is genuinely
  ambiguous.
- Every decision has a stated reason. "The reviewer said so" is not a
  reason to accept; "I wrote it that way" is not a reason to reject.
  "Unlikely in practice" is a valid reason to reject a robustness finding,
  and "nitpick" a valid reason to reject a naming or wording one. Neither
  is a valid reason to reject a simplification that meets the bar:
  deduplicating or deleting dead code already in the diff is not optional.
  Reject a simplification only by showing, against `FINDING_BAR`, that it
  does not meet the bar.
- **Scope changes are the user's, not yours or the reviewer's.** A finding that
  the spec should have covered something is an **escalation** if you agree
  it matters, and a **reject** with "out of scope" if you don't; never
  something you implement on your own initiative. A good idea that isn't
  worth stopping for is a `- [defer]` line in the round's triage.
- **Reversals always escalate.** A finding that would reverse any logged
  decision — your own earlier accept or reject, or a ruling by the user — is an
  escalation, not your call, even when you find the reviewer's argument
  convincing. Agreeing with the reviewer is not a way around this.
- A PLAUSIBLE finding gets checked (read the code, run it) before it is
  accepted or rejected. Never dismiss it unverified.
- **Research observations always reach the user.** Everything in the
  reviewer's "For the user" section, and any finding you reject or defer
  because it is about the research rather than the code (a confound, a
  metric's meaning, a data pattern), goes under the round heading as a
  `- for the user: <observation, with its evidence>` line. Never drop one
  as out of scope: out of scope means it is not the diff's to fix, not that
  the user should not hear it. These lines are collected at exit and in
  every auto-mode batch.

Then apply every accepted finding, run the tests, commit them, and append
`tests: <summary, no new failures vs BASE>` and `applied, committed <time>`
under the round heading. If the repo has a suite, a fix for a Blocking defect
must land with a test that reproduces its trigger where the suite can express
it: the reproduction you did for the acceptance test then survives as a
regression check, and the next reviewer has something to run rather than
re-derive. If you skip that test, say why on the accept line.

## Escalation — when a decision is the user's

Escalate (stop and ask, do not guess) when:

- fixing a finding, or finishing the implementation, would change the spec
  (add scope, drop scope, change an interface the spec fixes);
- the spec is ambiguous and the readings lead to materially different code
  (routine judgement calls you make yourself, as a careful colleague would);
- a finding would reverse any logged decision (yours or the user's);
- **the decision would change what a result number means or which
  records it is computed over** — a metric's definition or denominator, a
  judge's template, labels or notes, which sentinel or unjudged records
  are excluded, a sampling parameter, which models or cells are charted.
  These are research decisions and the most important escalations of all:
  an engineering call that goes wrong is caught by the next review, but a
  research call that goes wrong silently moves the numbers the project
  exists to produce. Escalate them even when the spec seems to settle the
  point and you are confident in your reading;
- you would reject a Blocking finding;
- tests fail and the fix needs a scope change;
- the round cap is reached but the loop would otherwise continue (the last
  verdict was NOT APPROVED, or its accepted fixes were structural). Ask
  whether to exit with the unreviewed-fixes caveat or run N more rounds;
  do not decide that yourself.

Also use common sense. If you think a situation arises that should be escalated,
do so even if it diverges from the flow described by the skill.

Mechanics (interactive mode; auto mode changes steps 1–4, see below):

1. Finish all work that does not depend on the answer first and commit it,
   so the tree is clean while you wait. Do not restructure history here; it
   is work-in-progress until exit, and the user can ask to see it.
2. Batch every open question from the round into one message. For each:
   the context, the options, and your recommendation.
3. Write the same questions to `LOG` as `- [escalate] ...` entries
   **before** asking, so nothing is lost if the session ends.
4. Ask the user and end your turn. Ending the turn is what pauses the loop.
5. When the user answers, append each ruling to `LOG` as its own line,
   `- ruling <file:line>: <their answer>`, at the end of the round — never
   edit the `[escalate]` entry it answers. Apply it and resume the loop at
   the same round.

A round is not finished while any `[escalate]` entry lacks a matching
`- ruling` line.
Phase 4 does not run until every escalation is resolved: ask, end the turn,
and only after the rulings are applied decide whether the loop continues.
Never list an open escalation in the exit report as something for the user to
look at later — that is a question, not a note.

### Auto mode (`--auto`)

The triggers above are unchanged: the same decisions are still the user's,
and every one still becomes an `[escalate]` entry with your recommendation.
What changes is that you do not stop for them:

- **Decide provisionally and keep going.** Under each `[escalate]` entry add
  `provisional: <what you did meanwhile>`. Act on your own recommendation:
  for a Blocking finding you would reject, that means leaving the code as
  it is; for a spec ambiguity, building the reading you recommend. When
  options are close, choose the one that is cheapest to undo, and say so.
  For a research decision (the trigger above), the provisional is always
  the spec's current text, whatever you would recommend: past runs show
  the implementer's recommendation is reversed most often on exactly
  these, and building a research preference into the code makes later
  rounds review numbers the user has not agreed to. List these first in
  the batch.
- **Never act against a ruling.** A finding that would reverse a ruling the
  user already made is logged as an escalation and the code stays as ruled.
- **Rounds finish despite open escalations.** The "round is not finished"
  rule above is suspended: write the round's `outcome:` line and, if the
  loop continues, start the next round. Where Phase 4 says to ask the user
  whether to run another round, run one, within the cap.
- **The stopping point is a pause, not an exit.** When the loop would exit
  (Phase 4's exit conditions) or the cap is reached with the loop wanting to
  continue, check for open escalations. If there are none, exit as usual.
  If there are any, append `outcome: pause — <n> open escalations` to the
  round, do **not** run the exit steps (no restructure, no filing the spec,
  no PR draft), and present every open question in one message: context,
  options, your recommendation, and what was done provisionally. Add the
  cap question if the cap applies, and every `- for the user:` line logged
  since the last batch, in their own section after the questions. Then end
  your turn.
- **After the rulings**, append each `- ruling` line as usual. If any ruling
  makes significant code changes (see Phase 4's test below) or the cap ruling
  asks for more rounds, apply the rulings, commit, and continue in auto
  mode. New escalations batch toward the next stopping point. If no ruling makes
  significant code changes and no more rounds were asked for, apply and commit
  any ruling that changes code, then run the exit steps; the report says that
  that code went unreviewed.

The cost of auto mode is that later rounds review code built on a
provisional decision, so a reversed one can undo several rounds of work.
That is why provisional choices prefer the cheapest-to-undo option and the
batch says what each provisional action was.

## Phase 4 — terminate or repeat

After triage, exit the loop if
- the reviewer returned APPROVED **and** you did not make any significant code changes
afterwards, or
- all [Blocking] findings were escalated to the user, whose rulings resulted in
  no significant code changes, or
- `ROUND_CAP` rounds have run and the user, asked per the escalation trigger, said to exit.

If an accepted fix after an APPROVED round requires significant code changes —
for example, adds a helper or changes a function's behavior — treat the round as
NOT APPROVED and run one more, within the cap. If the only code changes are
obviously correct, exit the loop. If you're unsure, ask the user whether to run
another round (in auto mode, run one instead).

A Blocking finding that the user rejects is settled — there is nothing for a
reviewer to contest — so if every Blocking finding ended that way, exit as if
APPROVED. A ruling that *does* make significant code changes needs to be
reviewed like any other accepted fix and triggers another round (if under the
cap).

Before leaving the round, append `outcome: repeat — <reason>` or
`outcome: exit — <reason>` to `LOG`, so a resumed session can read the
decision instead of re-deriving it. In auto mode, an exit with open
escalations is `outcome: pause` instead (see "Auto mode").

Whenever fixes land after the last review (accepted findings from the last
round, or the user's rulings), those changes are unreviewed, and the report must
say so.

On exit:

1. **Make the history merge-ready.** Restructure the commits since `BASE`
   into one commit per logical change, following the project's git workflow
   (`git reset --soft BASE` and recommit, or a `GIT_SEQUENCE_EDITOR`-scripted
   `rebase -i`; never `--force`, never touch commits before `BASE`). Each
   commit must stand on its own — its code runs and its docs describe the
   state at that commit, not a later one — with no "address review"
   commits. Afterwards assert `git rev-parse HEAD^{tree}` is unchanged from
   before the restructure; a differing tree means the rebuild lost or
   altered a change. That check proves content survived, not that each
   commit stands on its own — read each commit's diff for that. Never
   script conflict resolution: if a rebase hits a conflict, abort it and
   rebuild the commits explicitly (soft-reset to `BASE`, restore each
   commit's files, commit) — a scripted resolver has silently dropped
   content before.
2. **File the spec.** Move the spec file to `completed_specs/` at the top
   level of the current checkout (`git rev-parse --show-toplevel`), where
   the branch is, even when `RUN_DIR` is outside it (create the
   directory if needed; stop if a file of that name is
   already there) and commit it on its own, after the code commits. The
   subject names the change; the body lists the subjects of the commits the
   spec covers, one per line, from `git log --format=%s --reverse
   <BASE>..HEAD` — subjects only, no hashes, since a later restructure would
   change the hashes and the covered commits are the ones directly
   preceding this one anyway. This is the only commit that touches the
   spec, so the logical commits stay clean and a reader of history can find
   the design intent behind them. `LOG` stays in `RUN_DIR`, untracked.
3. Append `## Result` to `LOG`: rounds run, findings
   accepted/rejected/escalated/deferred per round, every `[defer]` line
   collected into one list, every `- for the user:` line collected into
   another, and why the loop stopped.
4. If the project's git workflow calls for a PR, draft it as that workflow
   says (title and body to a file in `RUN_DIR`, hand the user the push-and-create
   command).
5. Report to the user: what was built, the round count, the log path, any
   rulings they made, every `[defer]` line collected into one list, every
   `- for the user:` line collected into another (the research observations
   the reviewers made along the way, which otherwise survive only in the
   log), whether the last round's fixes went unreviewed, whether the work
   is stacked (if `git log --oneline origin/HEAD..<BASE>` is non-empty,
   name the branch it was made from: a PR against the default branch
   would carry those commits too; compare against the remote's default
   branch, which a PR targets and local `main` can lag, and against the
   local default branch only when there is no remote), and
   `git log -p --reverse <BASE>..HEAD` to review the commits.

The user can interrupt at any time; a later `/review-loop <same spec>` resumes
from `LOG` (Phase 0). The mode comes from the log header on resume, not
from the new invocation's flags.
