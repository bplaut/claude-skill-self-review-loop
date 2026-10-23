---
name: review-loop
description: Implement a spec file, then loop with fresh independent reviewers (a Claude subagent running /code-review, Codex, or both) until the change is clean, triaging and logging every finding and escalating scope decisions to the user. Use only when the user runs /review-loop <path-to-spec-file>.
argument-hint: <path-to-spec-file> [--research-spec <path>] [--rounds N] [--auto] [--reviewer claude|codex|both]
---

# review-loop

You are the **implementer**. You build the change described in the spec file,
then repeat rounds of independent review and triage until the change is clean.
The reviewer is always fresh, a newly spawned subagent or a new Codex
process, so it never sees your reasoning — only the spec (and research
spec, if any), the diff, and the decision log.

Arguments: `$ARGUMENTS` — the spec file path, optionally followed by any of
`--research-spec <path>`, `--rounds N`, `--auto` and
`--reviewer claude|codex|both`.

Constants (tweak here):

- `ROUND_CAP` = `N` if `--rounds N` was given, else `4`
- `AUTO` = true if `--auto` was given, else false. In auto mode the loop
  never pauses for a ruling mid-run: every escalation is logged with a
  provisional decision and the loop keeps going, then presents all open
  questions in one batch when it would otherwise exit or hits the cap. See
  "Auto mode" under Escalation.
- `REVIEWERS` = the `--reviewer` value, else `both`: `claude` is a Claude
  subagent each round, `codex` a Codex process, `both` one of each, run side
  by side. See Phase 2.
- `REVIEW_LEVEL = high` (Claude reviewer only; a Codex reviewer's depth is
  the user's Codex configuration)
- `RESEARCH_SPEC` = the `--research-spec` path if given, else none: a spec
  of what the code produces and why, which the spec builds on. It is
  context, not a second spec — the diff is held to the spec alone — and
  the run never edits it. Where it and the spec conflict, or the diff
  contradicts it on a point the spec leaves open, the user settles it (see
  Escalation).
- `RUN_ROOT = ~/review-loop`
- `RUN_DIR` = for a scratch spec, the directory containing it; for an
  in-repo spec, `RUN_ROOT/<repo>/<repo path without extension>/`, where
  `<repo>` is the main worktree's directory name — the last component of
  the first `worktree` line of `git worktree list --porcelain`, without a
  trailing `.git`, the same from any checkout of the repository — and the
  rest is the spec's repo path without its extension (Phase 0 says which
  kind a spec is, and what its repo path is). Everything this run produces
  — the decision log, the PR draft, anything else — goes here, so each
  invocation keeps its own records.
- `LOG = RUN_DIR/decisions.md`
- `FINDING_BAR` = `finding-bar.md` beside this file (absolute path
  `~/.claude/skills/review-loop/finding-bar.md`). The categories a
  finding must meet, shared verbatim by reviewer and implementer so the two
  cannot drift apart.

## What matters most

Everything below serves these four. When in doubt, weigh them above the
procedural detail.

1. **The reviewer is independent.** A new reviewer every round, subagent or
   Codex process, given only the spec (and research spec, if any), the
   diff, and the decision log — never your reasoning.
2. **Two channels only.** Anything the reviewer might need lives in the spec
   (with the research spec, if any) or the decision log. Conversation with
   the user that the spec omits goes in the log.
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
   file. `--research-spec` takes exactly one value, a path (step 4 checks
   it once step 3 has settled whether the run uses it). If `--rounds` is
   present, `N` must be a positive integer; `--auto` takes no value;
   `--reviewer` takes exactly one of `claude`, `codex`, `both`; anything
   else in `$ARGUMENTS` is an error.
   Every check in this phase applies in auto mode too: a failed
   check or an insufficient spec stops the run, since guessing there would
   poison everything after.
2. **Where the spec lives.** Here and in step 4, `<file>` is a spec's path
   as given, made absolute, `<its dir>` the directory holding it and
   `<file name>` its last component. A spec file is either in-repo or
   scratch:
   - **In-repo**: the file belongs to this repository — `git -C <its dir>
     rev-parse --path-format=absolute --git-common-dir` prints the same
     path as `git rev-parse --path-format=absolute --git-common-dir` run
     here — and is not gitignored there (`git -C <its dir> check-ignore -q
     <file name>` fails; `-C` resolves the path from `<its dir>`, so give
     it the bare file name). Its directory may be another checkout of the
     repository, such as the main checkout while you run in a linked
     worktree; what identifies the file is its repo path,
     `$(git -C <its dir> rev-parse --show-prefix)<file name>`.
   - **Scratch**: anything else — outside the repository, or gitignored
     inside it.

   Sort the spec now. That fixes `RUN_DIR`; create it if it does not
   exist. On a fresh run, `RESEARCH_SPEC` is sorted the same way; on
   resume, its path, kind and repo path are the log header's (step 3).
3. **Is this a resume?** If `LOG` already exists, its header must name this
   spec — the same absolute path for a scratch spec, the same repo path
   for an in-repo one — and this repository: its recorded git common dir
   is what `git rev-parse --path-format=absolute --git-common-dir` prints
   here, the same from any checkout of the repository and different for
   another clone; if not, the log is another run's, so stop and tell the
   user. Otherwise tell the user and ask
   whether to resume from it or start over; never silently overwrite it.
   On start over, continue as a fresh run with this invocation's flags,
   telling the user where they differ from the old header's mode,
   reviewers or research spec. The old log stays in place until the new
   one is created (below), so a check that stops the run first leaves the
   next invocation facing the same question. On resume, the mode, the
   reviewers and `RESEARCH_SPEC` are the log header's, not this
   invocation's flags — if the flags say otherwise, tell the user which
   the run is using — and see "Resuming" below; the remaining checks
   change meaning.
4. **The run's inputs are usable.** With a Codex reviewer,
   `codex login status` must exit 0 — it fails when the CLI is missing or
   logged out — so that stops the run now rather than after
   implementation; it runs on every invocation, resume included. On a
   fresh run, `--research-spec`'s path, if given, is an existing,
   non-empty file other than the spec itself (a different repo path if
   both are in-repo, else a different real path, as `realpath` prints
   it), and each in-repo spec must be committed at `HEAD` exactly as it
   is on disk: `git rev-parse HEAD:<repo path>` and
   `git hash-object --path=<repo path> <file>`, run at the top level of
   this checkout, print the same hash. If not, stop and say which case it
   is. A spec not yet committed at `HEAD` must be committed, and the run
   started from a commit that has it — or, if it was meant as a scratch
   spec, moved out of the repository or gitignored. A committed spec whose
   file differs from `HEAD` holds either an uncommitted change, likely an
   earlier attempt's or another session's — surface it, never revert it —
   or, when the path given is in another checkout, that checkout's
   different version: give this checkout's path, or start from a commit
   whose spec matches. On a start over, the spec at `HEAD` must also match
   the one at the old log's `BASE` (`git rev-parse <old BASE>:<repo
   path>`); if not, `HEAD` holds a change to the spec, perhaps an earlier
   attempt's rewrite — stop and ask the user which commit to start from.
   If they confirm `HEAD`, continue; if they name another, tell them to
   start from it and run the command again. On resume these commit checks
   were made before.

   Then read each spec in full: a scratch spec at its path, an in-repo
   spec as of `BASE` — `git show <BASE>:<repo path>` — never from its
   path, here or in any later phase, since the change may rewrite the
   file (Phase 1). On a fresh run `BASE` is the current `HEAD` (recorded
   below); on resume it is the log's.
5. No git operation is in progress: for each of `rebase-merge`,
   `rebase-apply`, `MERGE_HEAD`, `CHERRY_PICK_HEAD`, `REVERT_HEAD`,
   `sequencer` (a multi-commit cherry-pick or revert between commits),
   `test -e "$(git rev-parse --git-path <name>)"` fails. Ask git for the
   path: in a linked worktree `.git` is a file, so a literal
   `.git/<name>` never exists.
6. Tree state. **Fresh run:** no tracked file is modified or staged —
   `git status --porcelain` shows only `??` lines. Untracked files never
   enter `git diff <BASE>`; an uncommitted edit would. A dirty tree is
   likely another session's work — surface it, never revert it.
   **Resume:** a dirty tree is expected — it is this run's interrupted
   work. Show the user `git status` and `git diff --stat`, and ask whether to
   keep or discard it; do not stop.
7. Nothing the run writes to `RUN_DIR` can enter the reviewed diff. That
   needs checking only when `RUN_DIR` is inside the current checkout:
   `git -C RUN_DIR rev-parse --show-toplevel` prints the same path as
   `git rev-parse --show-toplevel` (run both without changing directory).
   Then `RUN_DIR` must not be the top level — `git -C RUN_DIR rev-parse
   --show-prefix` is non-empty (a spec there would put the log in the root
   and make the next check meaningless) — and must be gitignored —
   `git -C RUN_DIR check-ignore -q .` succeeds; if not, hand the user the
   one-line `.gitignore` edit. Anywhere else (outside git, another
   repository, another checkout) needs no check.
8. **The spec is sufficient** (fresh run only; on resume it was checked
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
   bounds). `RESEARCH_SPEC` is not held to this list, but a conflict
   between it and the spec is a gap too: a spec that narrows or fills in
   the research spec does not conflict with it; one that says something
   else about what the code produces does. If anything is missing or
   ambiguous, **stop and discuss it with the user** before writing code:
   list each gap with what you would assume. Write the answers **into the
   spec file itself** if it is a scratch spec, and re-run this check. An
   in-repo spec stays as committed — it is the reviewed design, and the
   answers depart from it — so hold its answers until `LOG` is created
   (below), where each becomes a `- ruling spec: <answer>` line under
   `## Context` and binds like the spec's own text; re-run this check on
   the spec together with those answers. This is a judgement, not a form —
   don't pad a spec that is already sufficient.

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
- On a start over, rename the old log to `decisions-<YYYYMMDD-HHMM>.md` in
  `RUN_DIR` now, so the earlier run's record survives. Then create `LOG`
  with a header recording the spec's absolute path and kind (`in-repo`,
  with its repo path, or `scratch`), the same for `RESEARCH_SPEC` (or
  `none`), the repository's git common dir (`git rev-parse
  --path-format=absolute --git-common-dir`), the branch and the
  one it was made from (`detached` if none), `BASE`,
  the start time, the mode (`auto` or `interactive`), the reviewers
  (`claude`, `codex` or `claude+codex`), the test status at
  `BASE`, and the untracked paths present
  now (`git status --porcelain | grep '^??'`, so your own new files can be
  told from pre-existing ones later). Then a `## Context` section listing
  every decision from your conversation with the user that the spec does not
  state — scope cuts, non-goals, deliberate simplifications, files that are
  out of bounds — or "none". For an in-repo spec, the Phase 0 answers are
  among them, each as a `- ruling spec:` line (step 8).
- With a Codex reviewer, write its prompt file now (see "Codex reviewer").
  Every placeholder in it is fixed from here on, so it is written once per
  run and every round reuses it.

**Resuming.** The spec and the log are your only state (a fresh session has
no memory of the last one). `BASE` and the branch are the values in the log
header, **not** the current `HEAD`; if you are not on that branch, stop and
tell the user. Find the last `## Round N` heading and re-enter at its first
incomplete step:

- no round heading → Phase 1 unfinished; compare `git diff <BASE>` to the spec
- heading but no pasted reviews → run round N's reviewers, all of them
- reviews pasted but no decisions → triage
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
- spec: <path> (in-repo <repo path> | scratch)   - research spec: <path> (in-repo <repo path> | scratch) | none
- repo: <git common dir>   - branch: <name> (from <branch>)   - BASE: <sha>   - started: <time>
- mode: auto | interactive
- reviewers: claude | codex | claude+codex
- tests at BASE: <pass/fail summary, or "no suite">
- untracked at start: <paths>
## Context
- ruling spec: <the user's answer> <- in-repo spec only: a Phase 0 gap answer
## Implementation decisions        <- Phase 1 only; complete before round 1
- <decision> — <reason> [— superseded, see round N]   <- the suffix is the one edit allowed later
## Round N
review started <time>
<reviewer>:                         <- claude or codex; one block per reviewer
```<its final message, verbatim>```
- [accept|reject|escalate|defer] <file:line> — <summary> (<reviewer(s)>)
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

**An in-repo spec is also a doc of the repo.** Whatever the repo's own
rules have the implementing change do to its design doc — rewrite it to
describe what was built, say — is part of the change: do it here, to the
file at its repo path in this checkout (never at the path you were given,
if that is another checkout), keep it current as later rounds' fixes
land, and the reviewer reviews it like any other doc in the diff. It is
still read as of `BASE` (Phase 0), never from the rewritten file. A move,
rename or deletion of the spec is the exception: the run is found again
by the spec's path, so it leaves that to the user and the exit report
names it. `RESEARCH_SPEC` is never edited by the run, whatever the repo's
rules say: it is the user's design input and may cover more than this
change. If those rules call for changing it, the exit report says so.

**The specs and the log stay out of the code.** Docstrings, comments,
tests, repo docs and commit messages never cite the spec, `RESEARCH_SPEC`
or `LOG` by their own labels — "Decision 3's variance", "per finding 2",
"the round 1 fix" — here or when applying fixes in later rounds. A reader
of the code has no log, and a spec's labels need not survive (a scratch
spec is never committed; an in-repo one may be rewritten), so the
reference explains nothing. An in-repo spec's own text is the exception:
references within it to its own sections, or to an in-repo research
spec's, are ordinary cross-references between committed docs. When a
decision shapes the code, write its substance (the formula, the rule, the
reason) where the code is.

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

Before running the reviewers:

1. Commit everything: no tracked file modified or staged, and no untracked
   paths beyond those recorded in the log header — an uncommitted new file
   is invisible to `git diff <BASE>`. The commit structure need not be
   final; the reviewer reads the diff, not the commits.
2. Append `## Round N` to `LOG` with a line `review started <time>`, so an
   interrupted review leaves a trace to resume from.

A Claude reviewer is spawned with the `Agent` tool:
`subagent_type: "general-purpose"`.
**Never `fork`** — fork inherits your context, which defeats the purpose.
**Never reuse a previous round's reviewer** via SendMessage — a new agent
each round; the decision log carries forward what needs remembering.
A Codex reviewer is run as in "Codex reviewer" below. With `both`, start
the Codex reviewer first (it runs in the background), then spawn the Claude
reviewer, then wait for Codex. Neither may see the other's message for the
round, so nothing from the round enters the log until both have returned
(Phase 3); each reads the other's earlier rounds there like any earlier
round.

The reviewer prompt must contain exactly this information and nothing about
your own reasoning, plan, or self-review. Fill `<spec location>` with the
spec's path for a scratch spec, and for an in-repo one with
`` `git show <BASE>:<repo path>` ``; fill `<research spec location>` the
same way. Each `[If ...]` passage goes in, without its brackets and
condition, only when the condition holds.

```
You are an independent code reviewer. You must not edit any file or run
any git command that changes state, and every helper agent you spawn must
be told the same — pass this constraint into each helper's prompt.

You are a subagent: your final message is the only thing the parent
session sees, and your turn ending is final — nothing re-invokes you.

1. Read the spec at <spec location>. The diff is meant to implement it.
   [If RESEARCH_SPEC: Also read the research spec at <research spec
   location>: it says what the code produces and why, and the spec builds
   on it. It is context, not a second spec: hold the diff to the spec
   alone. Report as a spec gap (step 5) a conflict between the two specs,
   or a place where the diff contradicts the research spec on a point the
   spec leaves open. A spec that narrows or fills in the research spec
   does not conflict with it. The run never changes the research spec: if
   the repository's rules call for updating it, that is left to the user
   and is not a finding; a diff that changes it is one.]
   [If the spec is in-repo: The spec read with `git show` is the version
   as of <BASE>. If the diff changes that file, the change is the
   repository's own update of its design doc: review it like any other
   doc in the diff, against the code. A move, rename or deletion of that
   file that the repository's rules call for is left to the user: its
   absence is not a finding, and a diff that moves, renames or deletes
   the file is one.]
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
5. Also check the diff against the spec — with the log's `- ruling spec:`
   lines, which bind like its text — and the implementation decisions:
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
fresh reviewer and do not proceed on an empty review. With `both`, the
round's verdict is APPROVED only if both reviewers returned APPROVED, and
either one failing stops the run: never proceed on half a review.

### Codex reviewer

A Codex reviewer is a new `codex exec` process each round, given the prompt
above. `codex review` would not do: it imposes its own output format, which
has no `VERDICT:` line and no `[Blocking]` labels.

1. The prompt file, `RUN_DIR/codex-reviewer-prompt.md`, is written once,
   when `LOG` is created on a fresh run: the prompt above with its
   placeholders filled in as for a subagent, and four changes that adapt
   what was written for a Claude Code subagent:
   - drop the clause about helper agents from the first paragraph;
   - at the end of the step that reads the spec, add: "Also read the
     repository's `CLAUDE.md`, if it has one, for its conventions." (a
     subagent loads it on its own; Codex reads `AGENTS.md` instead);
   - replace the step that runs /code-review with: "Review the diff since
     <BASE>: `git diff <BASE>`, for behavior defects, robustness issues
     with a trigger, and simplifications, per the finding bar. Do not review
     commits before <BASE>. The commit structure since <BASE> is
     work-in-progress and is restructured before merge; review the diff,
     not the commits or their messages.";
   - drop the sentence about the ReportFindings tool.

   The prompt goes in a file because it contains backticks and
   apostrophes, which the shell would interpret if it were passed inline.
2. Run this one command with `run_in_background: true`, `RUN_DIR` as an
   absolute path and `N` the round:

       codex exec --sandbox read-only -o "<RUN_DIR>/codex-round-<N>.md" "Read <RUN_DIR>/codex-reviewer-prompt.md and follow its instructions exactly."

   Nothing else on the line — no `cd`, pipe, redirect or `$(...)` — so a
   permission rule for `codex exec --sandbox read-only` matches it; and no
   `-c`, `--config`, `--enable` or `--disable`, so the user's Codex
   configuration stays in force. `--sandbox read-only` keeps Codex from
   writing, whatever that configuration says. Never `codex exec resume`: a
   new process each round, for the same reason as a new subagent. Wait for
   the completion notification without touching the working tree.
3. The reviewer's final message is the contents of
   `RUN_DIR/codex-round-<N>.md`; the command's own output is Codex's
   transcript and is not pasted. If the command exits non-zero, or the file
   is missing or empty, stop and tell the user, as for a missing verdict.
   Never fall back to a Claude reviewer in its place.

## Phase 3 — triage

Once every reviewer for the round has returned, paste each final message
**verbatim** into `LOG` under the round's heading, in one append, each in a
fenced block labelled with its reviewer (`claude:` or `codex:`). Not before:
a reviewer still running reads `LOG`, and must not find the other's review
of the same round there. The user and later reviewers must be able to see
what was actually said, not your paraphrase of it.

Then, and only then, for each finding — including the non-blocking items
under an APPROVED verdict — decide **accept**, **reject**, **escalate**, or
**defer**, and append the decisions below the pasted messages **before**
changing any code:

```
- [accept|reject|escalate|defer] <file:line> — <finding summary> (<reviewer(s)>)
  reason: <one sentence>
```

The parenthesis names the reviewer that raised the finding. With `both`, a
finding both raised gets one line naming both, so it is decided once.

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
- the spec and `RESEARCH_SPEC` conflict on something the diff does, or the
  diff contradicts `RESEARCH_SPEC` on a point the spec leaves open. Check
  the conflict is real first: a spec that narrows or fills in the research
  spec does not conflict with it;
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
  the spec's current text (for an in-repo spec, its text as of `BASE`
  together with the log's rulings) and, where the spec is silent,
  `RESEARCH_SPEC`'s, whatever you would recommend: past runs show
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
  round, do **not** run the exit steps (no restructure, no PR draft, no
  freeing the branch), and present every open question in one message: context,
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
- the round's verdict was APPROVED **and** you did not make any significant code changes
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
   commits. If Phase 1 changed an in-repo spec and the repo's rules don't
   say which commit that change goes in, it lands in the last commit or in
   one of its own after the code. Afterwards
   assert `git rev-parse HEAD^{tree}` is unchanged from
   before the restructure; a differing tree means the rebuild lost or
   altered a change. That check proves content survived, not that each
   commit stands on its own — read each commit's diff for that. Never
   script conflict resolution: if a rebase hits a conflict, abort it and
   rebuild the commits explicitly (soft-reset to `BASE`, restore each
   commit's files, commit) — a scripted resolver has silently dropped
   content before.
2. Append `## Result` to `LOG`: rounds run, findings
   accepted/rejected/escalated/deferred per round, every `[defer]` line
   collected into one list, every `- for the user:` line collected into
   another, and why the loop stopped. `LOG` and a scratch spec stay in
   `RUN_DIR`, never committed.
3. If the project's git workflow calls for a PR, draft it as that workflow
   says (title and body to a file in `RUN_DIR`, hand the user the push-and-create
   command). With an in-repo spec, the body also states where the build
   departs from the spec as of `BASE`, since a doc rewritten to describe
   what was built no longer shows what changed: every entry in
   `## Context` that no later ruling reverses (the `- ruling spec:`
   answers among them), and every implementation decision not
   superseded, round `- decision:` line and ruling that contradicts or
   narrows what the spec says — not the routine choices it left open.
4. **Free the branch** if this checkout is a linked worktree, which it is
   when `git rev-parse --path-format=absolute --git-dir` and
   `git rev-parse --path-format=absolute --git-common-dir` print different
   paths: run `git switch --detach`. Git refuses to check out a branch that
   another worktree has checked out, so until then the user cannot open the
   branch in their own checkout. Detaching stays on the same commit, so no
   file changes; switching to the default branch instead would fail
   whenever the main checkout has it. In the main checkout, skip this step:
   the branch is already the user's.
5. Report to the user: what was built, the round count, the log path, any
   rulings they made, every `[defer]` line collected into one list, every
   `- for the user:` line collected into another (the research observations
   the reviewers made along the way, which otherwise survive only in the
   log), whether the last round's fixes went unreviewed, whether the work
   is stacked (if `git log --oneline origin/HEAD..<BASE>` is non-empty,
   name the branch it was made from: a PR against the default branch
   would carry those commits too; compare against the remote's default
   branch, which a PR targets and local `main` can lag, and against the
   local default branch only when there is no remote), whether the branch
   was freed, whether the repo's rules call for a change to
   `RESEARCH_SPEC`, or a move, rename or deletion of the spec, that the
   run left to them, and
   `git log -p --reverse <BASE>..<branch>` to review the commits — the
   branch, not `HEAD`, since the user runs it from their own checkout.

If the user asks for changes after exit, `git switch <branch>` first. If
git refuses because the branch is checked out elsewhere, tell them rather
than committing on a detached `HEAD`.

The user can interrupt at any time; a later `/review-loop <same spec>` resumes
from `LOG`, with the mode, the reviewers and the research spec the log
recorded (Phase 0, step 3).
