# A Claude skill for iterated code review

This repo implements /review-loop, a Claude Code skill that implements a spec
and then iterates with an independent reviewer until the change is clean. One
Claude session writes the code; a fresh subagent reviews it each round with no
access to the implementer's reasoning. After each review, the implementer
decides whether each finding should be accepted or rejected, logs each decision
with a reason, and stops to ask you whenever a decision is yours (scope, spec
gaps, major disagreements with the reviewer). The implementer and reviewer
iterate until they agree or the maximum number of rounds has been reached
(default 4). I've found the cost per round to average around $50, depending on
diff size.

## Setup

1. Clone this repo into your skills directory:

       git clone https://github.com/bplaut/claude-skill-self-review-loop ~/.claude/skills/review-loop

   Claude Code picks up `SKILL.md` from there; `finding-bar.md` is read by
   path at runtime. It's important to clone into that precise directory,
   because that's where Claude Code will look for the skill.
2. The reviewer runs the built-in `/code-review` skill, so that must be
   available in your Claude Code install.
3. In each repo you use it in, the spec's directory must be gitignored
   (the skill checks). A convention that works: `.review-loop/<change>/`
   with `.review-loop/` in `.gitignore`.
4. On exit the skill moves the spec to `completed_specs/` at the repo root
   and commits it, so the design record lands with the code. Create nothing
   in advance; the directory is made on first use.

## How a run goes

1. Write a spec. The skill expects to see the goal, the scope, the non-goals,
   anything fixed rather than free (interfaces, names, flags), and how to
   tell it works. Including non-goals can help a lot: scope disputes can be
   settled without escalation if the spec makes the scope clear enough. If the repo
   has a test suite, the spec should say which tests the change adds. You could
   modify the skill to allow thinner specs, but I've found detailed specs to
   perform the best.
2. Run `/review-loop path/to/spec.md`, optionally `--rounds N` to change the
   round cap from its default of 4.
3. Phase 0 checks the tree is clean and the spec is sufficient. If the spec
   has gaps, the session asks you before writing code and puts your answers
   into the spec file.
4. Phase 1 implements, committing as it goes.
5. Phase 2 spawns a reviewer with only the spec, the diff since the base
   commit, the shared finding bar, and the log's context and rulings. It
   forms its findings before reading earlier rounds' triage, so it is not
   anchored by them. Its message starts with `VERDICT: APPROVED` or
   `VERDICT: NOT APPROVED` and labels every finding `[Blocking]` or
   `[Non-blocking]`.
6. Phase 3 triages: each finding is accepted, rejected, escalated, or
   deferred, with a one-line reason, before any code changes. Escalations
   pause the loop and wait for your answer; your rulings are appended to
   the log and are final.
7. Phase 4 decides: another round if the verdict was NOT APPROVED or the
   accepted fixes restructured code, otherwise exit. At the cap it asks you
   whether to stop or continue. On exit it restructures the commits into one
   per logical change, files the spec, and hands you the review and
   push commands. It never pushes.

You can interrupt at any time and resume in the same session by continuing the
conversation. The log on disk is the loop's state, so you can also resume in a
new session by running the same `/review-loop <spec>` command.

## What counts as a finding

`finding-bar.md` is the contract both sides work to. I've set the finding bar
up based on my personal priorities / needs. In my setup, a finding must do one
of these things: name a concrete trigger that reaches a failure, make the
code simpler in a way the reviewer can name, or quote documentation that is
objectively false. I explicitly state that guarding against hypothetical future edits
is not a finding, because I found that this is a path to scope creep. The implementer
accepts a finding only after verifying it by the method the file gives for its category.

## Things you may want to change

- **The finding bar** (`finding-bar.md`). The most likely edit. You can make
  the standards stricter or looser, add or remove categories, or restructure
  it completely. This is the one place to change it; both the reviewer
  and the implementer read it.
- **What blocks approval** (`SKILL.md`, the verdict definitions in the reviewer
  prompt). In particular, making most or all simplifications Non-blocking would
  shorten loops at the cost of leaving extra duplication. (Trivial
  simplifications are already Non-blocking.)
- **When the implementer must ask you** (`SKILL.md`, "Escalation"). As
  written, every reversal of a logged decision and every rejection of a
  Blocking finding comes to you. Loosening either gives the implementer
  more autonomy and you fewer interruptions.
- **The round cap** (`--rounds N`, default 4). Hitting it asks you rather
  than exiting, so it bounds cost without ending a productive loop.
- **The review depth** (`REVIEW_LEVEL` in `SKILL.md`, default `high`).
  Lower levels are cheaper and shallower.
- **Where the spec is filed** (`completed_specs/` in the exit steps). Your
  repo may have a different convention, or you might not want it committed
  at all.

## Using the same loop to write a spec

The skill implements a spec; it does not write one. But the same
implementer-and-fresh-reviewer structure works for drafting the spec itself,
informally, by asking a session to run it by hand. A prompt that has worked for me:

> I want you to engage in a review loop for this spec, similar to the
> review-loop skill, but here the goal is just to write the spec, not to
> implement. You will need to create your own reviewer prompts for the subagents
> and your own finding bar, as the current version is only for
> implementation. Ask for reviewer feedback on both research components and
> engineering components.

The spec that comes out is the input to `/review-loop`. I hope to make a proper
spec-design version of this skill at some point.
