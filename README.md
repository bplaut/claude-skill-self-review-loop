# Claude skills for iterated review of specs and code

This repo is a Claude Code plugin, `review-loop`, with two skills.
`/review-loop:spec` refines a design spec before anything is built; it is
described under [Reviewing a spec](#reviewing-a-spec).
`/review-loop:impl` implements a spec and then iterates with an independent
reviewer until the change is clean. One
Claude session writes the code; a fresh reviewer (a Claude subagent, Codex, or
one of each) reviews it each round with no access to the implementer's
reasoning. After each review, the implementer
decides whether each finding should be accepted or rejected, logs each decision
with a reason, and stops to ask you whenever a decision is yours (scope, spec
gaps, major disagreements with the reviewer). The implementer and reviewer
iterate until they agree or the maximum number of rounds has been reached
(default 4). I've found the cost per round to average around $50 for Fable 5.1
on high effort or $10 for Opus 5.5 on xhigh effort, depending on diff size.

## Setup

1. Clone this repo into your skills directory:

       git clone https://github.com/bplaut/claude-skill-self-review-loop ~/.claude/skills/review-loop

   The repo is a plugin (`.claude-plugin/plugin.json` names it
   `review-loop`), and Claude Code loads a plugin in your skills directory
   at the start of every session, with no install step. Its skills live in
   `skills/spec/` and `skills/impl/`, so the commands are
   `/review-loop:spec` and `/review-loop:impl`; each skill reads the files
   beside its `SKILL.md`, such as its `finding-bar.md`, at runtime. Clone into a folder
   directly under `~/.claude/skills/`, since that's where Claude Code looks.
2. The implementation's Claude reviewer runs the built-in `/code-review`
   skill, so that must be available in your Claude Code install. A Codex
   reviewer, part of both skills' default, needs the [Codex CLI](https://github.com/openai/codex)
   installed and logged in (or pass `--reviewer claude` to go without); it
   runs as `codex exec --sandbox read-only`, so allow that command in your
   permissions and, if you use Claude Code's sandbox, add
   `codex exec --sandbox read-only *` to `sandbox.excludedCommands`, since
   Codex fails inside that sandbox.
3. A spec is either in the repo (a design doc, say; `/review-loop:impl`
   needs it committed) or a scratch file kept outside it. Each run keeps its records (the decision log, the
   PR draft) in a run directory: for a scratch spec, the directory holding
   it, so give each scratch spec a directory of its own; for a spec in
   the repo, `~/review-loop/<repo>/<its path in the repo>/`, without the
   extension (`RUN_ROOT` in each skill's `SKILL.md`),
   made on first use. Keep scratch specs outside the repo too, e.g. under
   `~/review-loop/<repo>/<change>/`: Claude Code blocks a worktree
   session's file edits in the main checkout, so a run directory there
   blocks worktree runs; outside the repo, every session reaches the same
   single copy, and it outlives the worktree. Add `~/review-loop` to
   `permissions.additionalDirectories` in `~/.claude/settings.json` so
   sessions treat it like the project directory. A scratch spec in a
   gitignored directory inside the repo also works (the skill checks).

## How a run goes

1. Write a spec. The skill expects to see the goal, the scope, the non-goals,
   anything fixed rather than free (interfaces, names, flags), and how to
   tell it works. Including non-goals can help a lot: scope disputes can be
   settled without escalation if the spec makes the scope clear enough. If the repo
   has a test suite, the spec should say which tests the change adds. You could
   modify the skill to allow thinner specs, but I've found detailed specs to
   perform the best. A spec committed in the repo is read as of the commit
   the run starts from, so the change may rewrite it (to describe what was
   built, if the repo's rules say so) and that rewrite is reviewed like any
   other doc. If the change also has a research spec, saying what the code
   should produce and why, pass it with `--research-spec <path>`: the
   implementer and the reviewer read it as context and the run never edits
   it, the diff is held to the spec alone, and a conflict between the two,
   or a place where the diff contradicts it and the spec is silent, comes
   to you.
2. Run `/review-loop:impl path/to/spec.md`, optionally with `--research-spec
   <path>` (step 1), `--rounds N` to change
   the round cap from its default of 4, `--auto` to run without pausing for
   your decisions (see step 6), and `--reviewer claude|codex|both` to choose
   who reviews (default `both`). `both` runs two reviewers per round, and a
   round is approved only when both approve. A Codex review sends the spec,
   the log, the diff and whatever else Codex chooses to read to OpenAI, and
   its depth is whatever your Codex config sets (model, reasoning effort).
3. Phase 0 checks the tree is clean and the spec is sufficient. If the spec
   has gaps, the session asks you before writing code and puts your answers
   into the spec file (for a committed spec, into the log, where they bind
   like the spec's own text). It then makes a new branch from wherever you are, so
   start from the commit the change should build on.
4. Phase 1 implements, committing as it goes.
5. Phase 2 runs the reviewer with only the spec (and research spec, if
   any), the diff since the base commit, the shared finding bar, and the log's context and rulings. It
   forms its findings before reading earlier rounds' triage, so it is not
   anchored by them. Its message starts with `VERDICT: APPROVED` or
   `VERDICT: NOT APPROVED` and labels every finding `[Blocking]` or
   `[Non-blocking]`.
6. Phase 3 triages: each finding is accepted, rejected, escalated, or deferred,
   with a one-line reason, before any code changes. Escalations pause the loop
   and wait for your answer; your rulings are appended to the log and are
   final. With `--auto`, escalations do not pause the loop: each is logged with
   a provisional decision (usually the implementer's own recommendation,
   preferring whatever is cheapest to undo; for a research decision, always the
   spec's text, a committed spec's as of the base commit with your rulings, or
   the research spec's where the spec is silent) and the loop continues, then presents every open
   question in one batch when it would otherwise exit or hits the cap. Anything
   a reviewer notices about the research itself (a confound, a data pattern,
   what a number means) is logged as a note for you and included in every batch
   and in the exit report. A ruling that restructures code triggers further
   review rounds, so auto mode does not mean one pass.
7. Phase 4 decides: another round if the verdict was NOT APPROVED or the
   accepted fixes restructured code, otherwise exit. At the cap it asks you
   whether to stop or continue. On exit it restructures the commits into one
   per logical change and hands you the review and push commands; with a
   committed spec, the PR draft lists where the build departs from it. It
   never pushes. In a linked worktree it then detaches `HEAD`, since git
   won't let you check out a branch another worktree holds.

You can interrupt at any time and resume in the same session by continuing the
conversation. The log on disk is the loop's state, so you can also resume in a
new session by running the same `/review-loop:impl <spec>` command.

## What counts as a finding

`skills/impl/finding-bar.md` is the contract both sides work to. I've set the finding bar
up based on my personal priorities / needs. In my setup, a Blocking finding must
do one of these things: name a concrete trigger that reaches a failure, make the
code simpler in a way the reviewer can name, or quote documentation that is
objectively false. Non-blocking findings (naming, wording, and the like) need no
category. I explicitly state that guarding against hypothetical future edits
is not a finding, because I found that this is a path to scope creep. The implementer
accepts a finding only after verifying it by the method the file gives for its category.
A spec review has its own bar, `skills/spec/finding-bar.md`, with research and
engineering categories and the same rule against guarding hypothetical future edits.

## Things you may want to change

- **The finding bar** (`skills/impl/finding-bar.md`, and
  `skills/spec/finding-bar.md` for spec reviews, whose categories the role
  sections of `skills/spec/reviewer-brief.md` name). The most likely edit. You can make
  the standards stricter or looser, add or remove categories, or restructure
  it completely. This is the one place to change it; both the reviewer
  and the implementer read it.
- **What blocks approval** (`skills/impl/SKILL.md`, the verdict definitions in the reviewer
  prompt). In particular, making most or all simplifications Non-blocking would
  shorten loops at the cost of leaving extra duplication. (Trivial
  simplifications are already Non-blocking.)
- **When the implementer must ask you** (`skills/impl/SKILL.md`, "Escalation"). As
  written, every reversal of a logged decision, every rejection of a
  Blocking finding, and every decision that would change what a result
  number means comes to you. Loosening either gives the implementer
  more autonomy and you fewer interruptions. `--auto` is the per-run
  version of this trade: the same questions reach you, but batched at the
  end, at the cost of later rounds possibly reviewing code built on a
  provisional call you then reverse.
- **The round cap** (`--rounds N`, default 4). Hitting it asks you rather
  than exiting, so it bounds cost without ending a productive loop.
- **The review depth** (`REVIEW_LEVEL` in `skills/impl/SKILL.md`, default `high`).
  Lower levels are cheaper and shallower. A Codex reviewer's depth is set
  in your Codex config instead.
- **Where run records go** (`RUN_ROOT` in each skill's `SKILL.md`, default
  `~/review-loop`; keep the two equal), for specs in the repo. A scratch spec's
  records go beside it. A spec review writes under `feedback/` in the same
  run directory the implementation later uses.
- **The skeptic pass on specs** (`--skeptics`, default `on`). On, a fresh
  subagent tries to refute each Blocking finding before it is decided;
  off, rounds are cheaper and faster.

## Reviewing a spec

`/review-loop:spec path/to/spec.md` runs the same kind of loop on a spec,
before anything is built. One session is the author and improves the spec.
Fresh reviewers in a research role, an engineering role or both, chosen
from what the spec contains (research first while the research design is
still open), review it each round against `skills/spec/finding-bar.md`. Each
reviewer reads the spec, the bar and a short brief of what you have
settled, never the author's reasoning or the log. With `--skeptics on`, the
default, each Blocking finding also gets a fresh subagent that tries to
refute it before the author decides, as a check on accepting too readily.

The author triages every finding into a log under `feedback/` in the run
directory, so a spec's review and its later implementation keep their
records together. Research decisions, scope, and any Blocking finding the
author disagrees with come to you. With `--auto`, research and scope
questions are parked in the spec with nothing built on them, a Blocking
finding the author disagrees with is rejected provisionally and closed
only if the next round leaves it uncontested, and the open questions come
to you in one batch. If the spec implements a research spec, pass it with
`--research-spec <path>`: the run never edits it and holds the spec to it.
The other options are `--rounds N` (default 4), `--auto` and
`--reviewer claude|codex|both` (default `both`).

The loop ends when every role passes, after a final check of the edits no
reviewer saw, and leaves a spec that stands alone: everything the
implementer needs and no trace of the review. It does not start the
implementation: when you are ready, often in a new session or after
teammates have read the spec, run `/review-loop:impl` on it (with
`--research-spec` if it has one), committing in-repo specs first as your
project lands design docs.
