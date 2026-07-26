# Contributing to a2acast

Thanks for looking at this. a2acast is a cross-machine agent-to-agent mesh: one
stdlib-only Python file (`mesh.py`) plus harness integrations for Claude Code, Codex CLI,
and Copilot CLI.

> **If you are an AI agent**, the rules that matter are duplicated into `AGENTS.md`,
> `CLAUDE.md`, and `.github/copilot-instructions.md`, because those load into context
> automatically and this file does not. Read the one your harness loads. This document is
> the human-facing version with the reasoning spelled out.

## Getting set up

```bash
git clone https://github.com/husker/a2acast && cd a2acast
python3 -m unittest discover -s tests    # `python` on Windows
```

No build step and no dependencies. `mesh.py` runs as-is.

## Hard constraints

**`mesh.py` is stdlib-only.** The single-file, dependency-free property is the point of
the project — you can `curl` it, read it end to end, and audit it. A third-party import
destroys that. CI enforces this (`stdlib_only` job, #140) and the local suite enforces it too (`StdlibOnlyTests`); a third-party import fails before push. Stay vigilant — the test catches imports, not subtle misuse of a stdlib facility, so please don't be the one who breaks it.

**Python 3.8+.** `requires-python = ">=3.8"`. No `match` statements, no `X | Y` unions
evaluated at runtime, no 3.9-and-later stdlib additions.

**Mixed versions are the normal state.** Nodes in a real mesh run whatever version their
operator last installed. Any change to the wire format, and especially any new *signed*
field, must be safe against peers that predate it.

## Security-relevant changes

This means anything touching crypto, identity, pins, certificates, revocation, frame
verdicts, or the wire format — plus anything on the #62 / #74 / #76 / #93 lines.

The bar is higher here than "the tests pass," for a specific reason. On 2026-07-25 three
PRs landed in review together, each with a confident, well-written description asserting a
security property its code did not have:

- a rename guard that cited a corrected occupancy predicate but implemented the
  uncorrected one, leaving both known seams open;
- a rollout-safety gate that was send-side against a hazard that is receive-side;
- a mechanism built to make revocation non-suppressible that inverted into a suppression
  primitive for exactly the adversary it named.

Every one was caught by a reviewer reading the code, and none by reading the description.
The tests were green throughout. That is the failure mode these rules exist to prevent.

### Rules

1. **Never claim a property you have not tested.** A PR description is a hypothesis for a
   reviewer to falsify, not a summary of what you intended. Write what the code does. If a
   guard covers three of four cases, name the fourth.

2. **Ship a test that fails without the fix.** Not one that passes with it. If reverting
   your change leaves the suite green, the test does not test your change.

3. **A guard must be load-bearing in both directions.** Revert the fix and the attack test
   must fail. Disable the mechanism entirely and the *legitimate-use* test must fail. A
   guard that only ever refuses is one you could replace with `return False`.

4. **Gate on the property, not a proxy for it.** "The sender has a revocation list" is not
   "the fleet can parse this field." "The name has no pin" is not "the name is unclaimed."
   Substituting the cheap observable for the real condition is where these bugs live.

5. **Default new wire behaviour to OFF.** Gate it on an explicit operator assertion that
   the whole fleet is capable — never on something a single sender can observe about
   itself, which tells you nothing about receivers.

6. **The receive path does not durably mutate trust state.** Inbound frames may be
   observed, logged, and verified. Whether a received frame may ever write to the pin
   store or any identity record is an open design question (#137). Don't settle it inside
   a PR.

7. **Design-first for the signed frame body.** Bring the design to review before
   implementing. One of the three failures above was built around the wrong gate precisely
   because this step was skipped.

## Review

**Authors do not review their own work.** Every security-relevant change needs a reviewer
who did not write it.

"Independent" means a **different machine**, not a different session. Several agents can
run on one host under one node key, so co-located sessions share an identity and are not
independent reviewers (#136).

If you are reviewing:

- Read the code. The description is the thing under test.
- Reproduce the claim yourself. Don't take a summary on trust — including a confident,
  well-argued one from another agent.
- Prefer mutation-testing a guard both ways over reading the diff. It is the only thing
  that shows a guard is actually doing work.
- Report what you ran, with numbers. "Suite green" is not evidence that a specific hazard
  is covered.

Being wrong in review is fine and expected. Overclaiming is the problem — in either
direction. Asserting a bug is worse than it is costs as much credibility as asserting a
fix is more complete than it is, and it is easier to do when the stakes feel high.

## Pull requests

The template will prompt you for the claimed property, how to verify it, and the test that
fails without it. Fill those in — they are the parts a reviewer actually uses.

Keep changes to `mesh.py` reviewable. It is a large single file, and a diff that touches
one concern is far likelier to get a careful read than one that touches four.
