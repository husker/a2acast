<!--
Delete any section that does not apply. Do not delete the security sections
on a PR that touches crypto, identity, pins, certs, revocation, verdicts,
the wire format, or anything under a #62/#74/#76/#93 line.
-->

## What this changes

<!-- One or two sentences. What is different after this lands? -->

## Security property claimed

<!--
REQUIRED for security-relevant PRs. One sentence, in the form:
"After this, an attacker who <capability> can no longer <effect>."

Write only what the code does. If the guard is partial, say which case it
does NOT cover. A claim here is a hypothesis for the reviewer to falsify,
not a summary of intent -- if you have not tested it, do not assert it.
-->

## How a reviewer verifies it

<!--
Name the function and the predicate to read. "See the description" is not
an answer; the description is the thing under test.
-->

## Test that FAILS without this fix

<!--
Name it. If there is no such test, this PR is not ready.
For a guard, name BOTH tests -- see the checklist below.
-->

## Rollout / compatibility

<!--
Does this add or change a SIGNED field, or change how a frame is
reconstructed? If yes: what does a peer running an older version do when it
receives one? Mixed versions are the normal state of this fleet, not an
edge case. Off-by-default is usually the right answer.
-->

---

### Author checklist

- [ ] The claim above describes what the code does, not what I meant it to do
- [ ] Stdlib-only; no new third-party imports in `mesh.py`
- [ ] Python 3.8+ compatible (no `match`, no `X | Y` at runtime)
- [ ] `python3 -m unittest discover -s tests` passes (`python` on Windows)
- [ ] If this touches the signed frame body, it went to design review **before** implementation

### Reviewer checklist

- [ ] I read the code, not the description
- [ ] I reverted the fix and watched the named test **fail**
- [ ] For a guard: I disabled the mechanism and watched the *legitimate-use* test fail
      (a guard must be load-bearing in **both** directions -- blocking the attack
      AND still permitting the honest case)
- [ ] I am not an author of this change, and I am not on the same machine as one
