---
name: spec
description: Refine a design spec that is not yet implemented by looping with fresh independent reviewers (Claude subagents, Codex, or both) in research and engineering roles until the spec is clean, triaging and logging every finding and taking scope and research decisions to the user. Use when the user runs /review-loop:spec <path-to-spec-file>, or asks for a review loop to refine a spec before it is built.
argument-hint: <path-to-spec-file> [--research-spec <path>] [--rounds N] [--auto] [--reviewer claude|codex|both] [--skeptics on|off]
---

# review-loop:spec

You are the **author**. The spec file is your draft; you improve it through
rounds of independent review and triage until it is clean. Nothing is
implemented, run or committed, and nothing is edited but the spec and files
under `FEEDBACK`. The output is the spec itself. Implementing it with
`/review-loop:impl` is a separate step the user starts when they choose,
often in a new session or after teammates have read the spec; never start
it yourself.

Arguments: `$ARGUMENTS` — the spec path, optionally followed by any of
`--research-spec <path>`, `--rounds N`, `--auto`,
`--reviewer claude|codex|both` and `--skeptics on|off`.

Constants (tweak here):

- `ROUND_CAP` = `N` from `--rounds`, else `4`. A ceiling, not a quota.
- `AUTO` = true with `--auto`. See "Auto mode".
- `REVIEWERS` = the `--reviewer` value, else `both`: who fills each role.
  `both` runs a Claude subagent and a Codex process on every role.
- `SKEPTICS` = the `--skeptics` value, else `on`. See "Skeptic pass".
- `RESEARCH_SPEC` = the `--research-spec` path, else none: a spec of what
  the code produces and why, which this spec implements and must stay
  faithful to. It is the user's; the run never edits it.
- `RUN_ROOT = ~/review-loop`
- `RUN_DIR`, as in `/review-loop:impl`: for a scratch spec, the directory
  holding it; for an in-repo spec, `RUN_ROOT/<repo>/<repo path without
  extension>/`, where `<repo>` is the main worktree's directory name (the
  last component of the first `worktree` line of `git worktree list
  --porcelain`, without a trailing `.git`). A spec is **in-repo** when it
  belongs to this repository and is not gitignored there, and **scratch**
  otherwise.
- `FEEDBACK = RUN_DIR/feedback/`: everything this skill writes besides the
  spec. `/review-loop:impl` keeps its own log beside it.
- `LOG = FEEDBACK/decisions.md`
- `FINDING_BAR` = `${CLAUDE_SKILL_DIR}/finding-bar.md` and `BRIEF` =
  `${CLAUDE_SKILL_DIR}/reviewer-brief.md`, beside this file. Give reviewers
  expanded absolute paths.

## What matters most

Everything below serves these five. When in doubt, weigh them above the
procedural detail.

1. **The reviewer is independent.** A new reviewer every round, given the
   spec, the bar, the code and data, and a brief of settled points, never
   your reasoning or the decision log.
2. **You judge; the reviewer advises.** Reviewers have fresh eyes, not
   authority. Accept a finding only when you have checked it yourself and
   agree, and judge its proposed fix separately. Every accept and reject
   carries your own reason. Near-total acceptance means this step is not
   happening.
3. **Research decisions and scope are the user's.** Anything that changes
   what a reported number means, which records it rests on, or what the
   spec builds goes to the user when it arises. You never settle it, not
   even provisionally.
4. **The spec stays a lean implementation guide.** It describes the design
   as it is, at the level it is written. The smallest fix wins: correct,
   tighten or delete before adding. Discussion, analyses of existing data
   and the review history live in `LOG` or chat, never in the spec.
5. **At exit the spec stands alone.** Every ruling and settled point the
   implementer needs is in the spec, because the implementer never reads
   this loop's log.

## The log

`LOG` is the loop's state and its record. It is **append-only**: never
insert into the middle; a later line supersedes what it names. Nothing
parses it; a fresh session reads it to find its place, so keep to this
shape.

~~~
# review-loop:spec log
- spec: <absolute path> (in-repo <repo path> | scratch)   - research spec: <path> | none   - started: <time>
- mode: auto | interactive   - cap: <N> (<who set it>)   - reviewers: <claude | codex | claude+codex>   - skeptics: on | off
## Context
- request (<date>): "<the user's words, verbatim>"
- <each ruling that bears on this spec, whether or not the spec states it, in the user's words, dated: from your conversation with them, or from an earlier log of this spec (a start-over's renamed feedback, or one the user names; on a fresh run of a spec that attributes choices to the user or calls them settled, ask where its earlier log is: a gap until answered, "none" included)>
- <a standing preference of the user's that bears on this spec>
- reading: <your reading of a terse instruction> — open to findings
- decision: base <repo> at <sha> on <ref> | none
- decision: roles <plan, and why>
- [escalate] 0.q<n> <a question the spec carries, or setup raised>
## Round N
review started <time>; roles <...>; spec copy round<N>-spec.md (<lines> lines)
<role>/<reviewer>: VERDICT <...> — round<N>-<role>-<reviewer>.md
- [accept|reject|escalate|defer|settled] <id> <spec section> — <summary> (<severity> <category>)
  reason: <your own, and how you verified it>
  fix: <your fix, when it differs from the reviewer's>
  skeptic: <its verdict, in a line>
  meanwhile: <auto mode, [escalate] only: what the spec says until the user rules>
- decision: <a choice of yours and why; if the user may reverse it, its alternative and trade-off>
- correction: <a reversal of your own earlier triage, and why>
- ruling (<date>)[ on <id>]: "<the user's words>" — <the option it takes; what it supersedes>
- reading: <your reading of a terse ruling>, asked of the user
- for the user: <an observation, with its evidence>
- deviation: <a reviewer's reported breach of its constraints>
revised <time>: <before> -> <after> lines; doc checks: <result or none>
verification: VERDICT <...> — round<N>-verify.md
outcome: repeat | pause | exit — <reason>
## Result
## After exit (<date>)
~~~

`- ruling` is only ever the user's own words or their announced edit to the
spec; your choices are `- decision:`. `[settled]` marks a finding a ruling
already decides, or one that repeats a rejected finding without the
argument the brief asks for; name what it repeats or what decides it. A
finding's id is `<round>.<role letter><reviewer letter><n>`, with `c` for
Claude and `x` for Codex (`2.ex3`), and `<round>.v<n>` for a verification
pass, so it never reads as a finding-bar category.

## Phase 0 — setup (fail fast)

If a check fails, stop and tell the user which one and why.

1. **Arguments.** The first token is a path to an existing, non-empty
   file; `--research-spec` names another one; `N` is a positive integer;
   `--auto` takes no value; `--reviewer` takes `claude`, `codex` or
   `both`; `--skeptics` takes `on` or `off`; anything else is an error.
   Sort the spec, which fixes `RUN_DIR`. An in-repo spec must be in this
   session's own checkout, since you edit it and checks run against it.
2. **Resume?** If `LOG` exists, ask the user whether to resume it or start
   over; never overwrite it silently. To start over, rename `FEEDBACK` to
   `feedback-<YYYYMMDD-HHMM>/`. On a fresh run, once step 3 passes,
   create `LOG` with its header and the request, and copy the spec as given
   to `FEEDBACK/round0-spec.md`; the steps below append to `## Context`.
3. **The session can run it.** With a Codex reviewer, `codex login status`
   must exit 0 and the working directory must be inside a git repository,
   since Codex refuses to start outside one. If either fails under the
   default, say so and suggest `--reviewer claude`; do not drop Codex
   yourself.
4. **The spec is reviewable.** It must state its goal, its scope and its
   **non-goals**, and, for an experiment or a reported number, its research
   question and claim. A missing goal or non-goals is the user's to supply.
   Questions the spec already puts to the user (not ones it addresses to
   someone else, which are spec content), and any part of it that reverses
   one of their rulings, are escalations from the start
   (`[escalate] 0.q<n>`). Weigh the spec's depth against the change: an
   extension of an existing mechanism follows that mechanism's precedent
   and carries only what is new. Move analyses of existing data out
   yourself; a threat, caveat or expected result you would cut is the
   user's call. Structure the project requires of a design document (its
   `CLAUDE.md`, a rubric it names) is yours to add.
5. **The base.** If the spec builds on code, log the commit it builds on as
   `BASE` (the spec may name it; else `HEAD`; if it builds on several
   branches, one per branch, as the spec names them) and the branch it builds
   on, as its remote-tracking ref where it has one (`origin/<branch>`; a
   local branch can lag what was fetched): never a session worktree's own
   branch or one that only carries the spec.
   Reviewers read code at `BASE` with `git show`, never the working tree,
   which other sessions move. If code for this change already exists
   elsewhere (a prototype), the spec tells the implementer to build from
   scratch from the branch it builds on, named in the spec, and to stop if
   `HEAD` holds changes beyond that branch other than design documents (the
   spec and its research spec); it carries no pointers to the prototype. If uncommitted edits touch files
   the spec names, ask whether the spec builds on them rather than
   labelling them out of scope; they may be another session's, and
   committing them is the user's job.
6. **Roles.** **Research** if the spec makes a research claim, defines a
   metric, chart or judge, or plans an experiment; **engineering** if it
   plans code; both if both. When the research design is new or still
   open, run research-only rounds until research passes (Phase 4), then
   add engineering: engineering rounds on a design that then changes are
   wasted. When it is narrow or already ruled on, run both from round 1.
   Log the plan with its reason; the user may override it.
7. **Ask.** Put the gaps, your readings of terse instructions (in concrete
   terms), the `0.q` questions, step 5's questions, the role plan and, if
   none was given, the cap to the user in one message ("How to ask"). In
   interactive mode, end your turn. In auto mode, end it only while a gap
   or a step 5 question is open; the `0.q` questions are parked (Auto
   mode).

**Resuming.** The spec and the log are your only state. Take the mode,
cap, reviewers, skeptics, research spec, base and roles from
the log (its header and Context, as changed by any later ruling or
decision line), not from this invocation's flags; where a flag differs,
tell the user. If the
spec argument is not the log's spec, stop and say from which checkout to
resume. A spec that changed since the log's last entry may hold your own
interrupted revision or someone's edit: compare it with the round's copy
and the logged accepts and rulings before building on it, and never
revert it on your own. Step 3 still applies. Then read the log and continue from the first
step it does not record as done.

## Phase 1 — the round's inputs

Before spawning reviewers each round:

1. Re-read the spec, and `RESEARCH_SPEC`, from disk.
   - A spec the user says they edited is authoritative: log their edit as
     a ruling, reapply any of your edits their save lost from a stale copy
     (text they rewrote stands), and say so in the round report. A change
     nobody announced may be another session's: build on it without
     undoing anything, log it as a decision, and ask the user whose it is.
   - A changed research spec is the user's revision: log what changed,
     re-sync the spec as "Escalation" says, and drop amendments it now
     carries.
   - If the ref logged with `BASE` has moved past it, check the new
     commits against what the spec relies on; if any touches it,
     log `- decision: base moves to <sha> — <what changed>`, update the
     spec, and list the move under What changed. The check covers only
     what the user has fetched; say so.
   - Do not write to the spec or start a round while the user says they
     are editing either file.
2. Copy the spec to `FEEDBACK/round<N>-spec.md`, and the research spec to
   `FEEDBACK/round<N>-research-spec.md`: the copies are what the round
   reviews, so an edit mid-round cannot split the reviewers.
3. Write `FEEDBACK/round<N>-brief.md` from `BRIEF`. It carries, as short
   facts:
   - **Settled and pending**, every round: the user's request and any focus
     they gave, verbatim; their rulings and closed points; standing
     preferences that bear on the spec and are not already in the project's
     `CLAUDE.md`; your readings of their instructions, marked "open to
     findings"; rulings the research spec does not yet reflect, marked as
     amendments; findings you rejected, provisionally included, one line
     each with your reason; questions pending with the user.
   - **What changed since the last round**, from round 2: each section
     edited, in a phrase, and every design removed, so reviewers check the
     fixes landed and hunt leftovers.
   It never argues for the spec or predicts findings.
4. Append `## Round N` and its `review started` line to `LOG`.

## Phase 2 — review

Spawn every reviewer of the round at once. A Claude reviewer is the Agent
tool with `subagent_type: "general-purpose"` and this prompt:

    You are the <ROLE> reviewer for round <N>. Your scratch directory is <FEEDBACK>/round<N>-<role>-claude/. Read <FEEDBACK>/round<N>-brief.md in full and follow it exactly.

**Never `fork`**, which inherits your context, and **never reuse** a
previous round's reviewer. A Codex reviewer is a new process per role and
round, run through Bash with `run_in_background: true`, from inside the
repository (`cd` back first, as a Bash call of its own, if the working
directory has moved), with absolute paths:

    codex exec --sandbox read-only -o "<FEEDBACK>/round<N>-<role>-codex.md" "You are the <ROLE> reviewer for round <N>. You have no scratch directory, so write no files. Read <FEEDBACK>/round<N>-brief.md in full and follow it exactly."

Nothing else on the line: no `cd`, pipe, redirect or `$(...)`, and no
other flag. Never `codex exec resume`.

Save each Claude reviewer's final message **verbatim** to
`FEEDBACK/round<N>-<role>-claude.md` as it returns; Codex's is its `-o`
file. Read each verdict from its `VERDICT:` line. A reviewer that returns
no verdict (it ended its turn waiting on helpers, returned nothing, or
Codex exited non-zero or left the file empty) has failed: stop and tell
the user. Never proceed on part of a round, and never substitute another
reviewer.

## Phase 3 — triage and revise

Once every reviewer has returned, append a line per reviewer (verdict and
file) to `LOG`, and a `- deviation:` line for any breach one reports.

**Skeptic pass** (`SKEPTICS`, on by default). Before you decide anything,
give each Blocking finding from a reviewer a fresh subagent under the
reviewer constraints, all at once, with the finding verbatim, the round's
brief and the bar, and nothing of your own view. It tries to refute the
finding against the spec, code and data, and says whether the finding is
real, whether Blocking is warranted, and whether a smaller fix exists.
Append its output verbatim to `FEEDBACK/round<N>-skeptics.md` and add a
`skeptic:` line to the finding's entry. Its verdict is input, not a
ruling: where you disagree with it, say why.

Then decide every finding, Non-blocking ones and those under an APPROVED
verdict included, and append the decisions **before** editing the spec. A
finding several reviewers raised gets one entry naming each; two roles
finding the same defect is strong evidence it is real.

- **The acceptance test.** Accept only when all three hold:
  1. You verified it by its category's method (the bar's "Verification,
     per category"). A claim you cannot check is checked another way (the
     code or source instead of an unreachable page), not rejected for
     that alone; a request that was denied is not unreachable: tell the
     user.
  2. It is not on the bar's "Not findings" list and breaks no rule of the
     project.
  3. You agree it makes the spec better at its own goal, and your reason
     says so in your own words.
- **The fix is judged on its own.** "Accept the finding, write your own
  fix" is normal: a smaller fix, an existing mechanism, a caveat instead of
  machinery. Never accept a fix that adds a check, guard or option whose
  only trigger is hypothetical, and hold your own additions to the same
  test.
- **Research decisions, scope and rulings go to the user** ("Escalation"),
  whoever raised them and at whatever severity.
- **Your own earlier triage** you may reverse, with a `- correction:`
  line. A finding against one of your own decisions in the spec is an
  ordinary finding.
- **"For the owner" and "Considered but not raised" lines** that bear on a
  claim the spec makes are triaged like findings; good items a reviewer
  held back you may raise yourself. The rest become `- for the user:`
  lines. A scope idea that comes back in a later round is escalated, not
  relayed again.
- **Numbers, citations and commands** enter the spec only after you
  recompute, open or run them yourself: read-only against `BASE`, or a
  command that changes state in a throwaway copy of the repository under
  `FEEDBACK`.
- **Defer** is for good ideas outside this spec's scope: they go to its
  non-goals or future-work list, or to `## Result`, never into the design.
- **Before revising, re-read your accepts** as if you had to defend the
  spec as written, and log each call that changes as a `- correction:`.

Then revise the spec:

- Re-read it from disk first (Phase 1 step 1 applies to any change).
- Apply each accepted fix everywhere it bears, and search the spec for
  every name, number or claim it changed, so nothing contradicts it.
- Write the design as it now is: no mention of reviews, rounds, findings
  or rulings, and no log labels; a ruling's substance goes where the spec
  makes that choice. A rejected alternative and its trade-off, where the
  project's rubric asks for them, are design, not discussion. Quote the
  research spec rather than paraphrasing it.
- Questions for the user live in one numbered section of the spec, each
  with options and your recommended default; a ruling replaces its
  question with the decision, written where it applies.
- Do not over-specify: leave the implementer what it can decide, state
  rules rather than closed lists, and name functions and tests rather than
  line numbers. Data facts that may change are described qualitatively or
  dated with a command in the repository; counts from a reviewer's script
  stay in `LOG`.
- Run the checks the project names for documents of this kind, and fix
  what they report.
- Append the `revised` line. If the spec grew by more than about a quarter,
  or most accepted findings sat below its level, say so in the round
  report.

**Round report.** After each round, tell the user in a few lines, so they
can follow without the log: each role's verdict; the counts accepted,
rejected, escalated and deferred; what changed in the spec and its length;
the round's `[defer]`, `- for the user:`, `- deviation:` and `- reading:`
lines; and, in auto mode, each research or scope question parked this
round, in full as "How to ask" describes, and each provisional rejection
with its reason, so a user who is present can step in.

## Escalation — when a decision is the user's

Escalate (do not decide) when:

- **the decision would change what a reported number means or which
  records it rests on**: a metric's definition or denominator, which
  records are excluded (failed, errored or unscored ones), a scoring rule
  or the instructions a grader follows, a sampling parameter, which
  conditions are reported, whether to add a caveat on how a number is
  read. These are research decisions and the most important escalations:
  an engineering mistake is caught by the next review, a research mistake
  silently moves the numbers the project exists to produce. An ambiguity
  whose readings differ in such a way is one of these, not a sentence for
  you to add. Correcting a threat the spec already states is yours. Making
  the spec say what the research spec plainly says is yours; anything in
  the research spec that is unclear or conflicts with a ruling or the
  code, and any choice it leaves open that changes a number, is the
  user's;
- a fix would change the spec's goal, scope or non-goals, or cut or
  narrow something the user asked for;
- a finding would reopen one of the user's rulings, or edit text the user
  wrote;
- a fix alters how a ruling is carried out in substance or cost;
- you disagree with a Blocking finding, however sure you are: give your
  argument and what you checked;
- the round cap is reached and the loop would otherwise continue.

Engineering details no ruling covers are yours.

**How to ask.** First check the consequence you will state against the
code or data, as you would a finding. Batch the round's questions into one
numbered message the user can answer by number. For each: the trigger as a
numbered sequence of the user's own actions (the commands they would run
and what is on disk at each step), or, for a construct or judge rule,
example responses with the label each option gives; where the options
diverge and what does not change; the options, including descoping,
deferring, measuring first in a pilot, or stating a caveat; your
recommendation and its reason; and any reviewer's or skeptic's view that
differs. Quote the spec rather than citing its labels, define every term,
and say when your recommendation is your own unreviewed view on a research
question. Log each question as `[escalate]` **before** asking, then ask and
end your turn; ending the turn is what pauses the loop.

When the user rules, log the ruling, write its substance into the spec,
and continue; the next brief carries it. A reply still discussing a point
is not a ruling: make no edits that touch it. If a ruling is terse, log
your concrete reading and ask the user to correct it in your next message;
a reading that would change a reported number is escalated instead. A
round is not finished while an `[escalate]` lacks a ruling (auto mode
aside).

The user may interject at any time, in auto mode too; answer at once. A
changed cap or an answer to an escalation is a ruling: log it and act on
it. An added requirement is theirs to decide, but give
your view before building it, including, if it widens scope, its cost to
this loop and whether it belongs in its own spec. If the spec or research
spec changes materially while reviewers run, triage against the current
text and make sure the change itself gets reviewed.

### Auto mode (`--auto`)

The triggers are the same; you stop for few of them:

- **Research and scope questions are parked, never decided.** The spec
  lists the question with its options and your recommendation; text it
  already has, including a part that departs from a ruling (the question
  quotes that ruling), stays as it is until the ruling, and nothing new is
  built on any answer (`meanwhile: unchanged`). If every remaining finding
  depends on the answer, pause now rather than review a design the answer
  could rewrite.
- **Never act against a ruling or edit text the user wrote**
  (`meanwhile: unchanged`).
- **A Blocking finding you disagree with**, other than a research or
  scope call (parked, above), **is rejected provisionally**: the spec stays
  as it is (`meanwhile: rejected provisionally`) and the next brief lists
  it among the rejected findings, open to contest. If no reviewer in that round contests it
  with an argument against your reason, close it with a decision line; it
  needs no ruling and is reported for information. If one does, it is
  parked like a research question (`meanwhile: unchanged`, pending in later
  briefs) rather than rejected provisionally again.
- **Other escalations** (engineering choices within the agreed scope): act
  on your recommendation, preferring what is cheapest to undo.
- Rounds finish despite open escalations. When the loop would exit, or
  hits the cap wanting to continue, and escalations are open, append
  `outcome: pause` and present them all in one message, research decisions
  first, each with what was done meanwhile, plus the cap question if it
  applies and the `- for the user:` lines since the last batch. After the
  rulings, apply them and decide Phase 4 again.

## Phase 4 — repeat or exit

A revision is **substantial** if, by its effect, it adds, removes or
changes a decision, a mechanism, a record or file the design writes, or a
rule a number depends on, or if it moves the base. A wording fix, a
corrected reference or doc list, a deleted sentence, or a ruling written
in that changes none of these is not.

A role **passes** a round when its reviewer returned APPROVED, or when
every Blocking finding it raised was accepted, `[settled]` or decided by
a ruling, and the revisions they caused were not substantial; with `both`, it passes
only if it passes for each reviewer. A provisionally rejected finding is
none of these, so its role runs again. A role that has passed sits out until a
substantial revision touches what it covers.

The loop **would exit** when every role in the plan has passed and nothing
substantial has changed since. Otherwise it repeats, within the cap, with
every role that has not passed or that a substantial revision brought back
(in a research-first plan, engineering joins once research passes). Two
cases go to the user:

- When every role has passed and only a change made since (a fix after an
  approval, or a ruling) calls for another round, and the user is in the
  conversation, offer the round with those changes listed; if they
  decline, the loop would exit. In auto mode with no pause, run it.
- At the cap with the loop wanting to repeat, escalate the cap question
  with the open findings, the substantial changes no round has reviewed,
  and any role that has not passed or never ran. Never exit or extend on
  your own.

**Before any exit** (the loop would exit, the user declined a round, or
the user ended the loop), and once no escalation is open, run Phase 1 step
1 (a user edit, a research-spec revision or a base move found there may
mean another round), then the **verification pass**: one fresh subagent
under the reviewer constraints, scratch `FEEDBACK/round<N>-verify/`, given
the spec, the research spec as it now is, the current base, the bar, the
last brief, every ruling logged
since that brief, each role's reviewer files and spec copy from the round
it last reviewed, and the Blocking findings of those rounds that were
accepted, settled or decided by a ruling, with the fix applied or the
ruling. It checks that
each fix resolves its finding as applied; that the edits since those
copies leave no contradiction, stale reference or leftover of a removed
design; that the spec stands alone (no reference to the review or its
labels, or to prototype code, and every ruling reflected); and, with a
research spec, every item of it against the spec. Save its output to
`FEEDBACK/round<N>-verify.md` and triage it like a round. Its fixes are not
verified again; they go on the unreviewed list. A Blocking result that
needs more than a small correction means another round or an escalation,
and one you disagree with goes to the user even in auto mode, since no
role runs again to contest it. Run the pass again only if a round, a
ruling, a user edit or a base move comes after it; a repeat never reuses
an earlier pass's file, directory or finding ids.

Then append `outcome: repeat | pause | exit — <reason>`. On exit, append
`## Result` to `LOG` and report the same to the user in plain language:
the absolute paths of the spec and `LOG`; the rounds run against the cap
and why the loop stopped; per round, the verdicts and the counts accepted,
rejected, escalated and deferred; each role's last verdict, and any role
that never ran; every ruling; the open questions left in the spec; your
decisions they may reverse, each with its alternative and trade-off; every
accepted change that alters what a reported number means; for
information, the provisional rejections closed uncontested; the `[defer]`
and `- for the user:` lines; every base move, and the last base check with
what it covers; the edits no reviewer saw; the spec's length as given
and now; `diff <FEEDBACK>/round0-spec.md <spec>` to read the whole change;
and how to implement it when they are ready: `/review-loop:impl <spec>`,
with `--research-spec <path>` when this run had one, after committing each
in-repo one as the project lands design documents, run from a commit that
holds both the base and the committed specs (`/review-loop:impl` reads
them as committed and branches from what is checked out). With a research
spec, name the rulings it does not yet carry that say something else than
it: `/review-loop:impl` stops on those as conflicts until the user writes
them in.

**After exit.** Later rulings, re-checks and spec edits go under
`## After exit (<date>)`: the user's words as rulings, yours as decisions.
Tell the user which edits no reviewer saw. Run another round only if the
user asks; it ends with its own `## Result`.
