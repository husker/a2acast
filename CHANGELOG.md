# Changelog

## 0.20.0

Fleet-wide liveness truth: every stall mode of a receiver is now detectable,
most are auto-recovered, and each node's health is visible from every other
node — plus two receive-path crash-class fixes. All new presence fields are
additive, advisory, and display-only; older peers ignore them and read as
"unknown" (mixed-version-safe).

- **#158/#164 — `watch --supervised` stall-watchdog.** A supervised receiver
  whose #161 heartbeat goes stale recovers itself: close-response first, then
  `os._exit` under a KeepAlive restarter, so a wedged subscription is a
  restart instead of a silent multi-day outage.
- **#166/#167 — presence carries receiver-health.** Pong/ack control frames
  stamp the sender's own `recv: healthy|stalled`, making remote receiver
  liveness readable fleet-wide (data layer).
- **#169 — `mesh fleet`.** A read-only fleet health table: last-seen,
  posture/status, and per-node RECEIVER / AGENT columns from the propagated
  health fields.
- **#170/#171 — agent-liveness heartbeat.** A per-node agent heartbeat
  distinguishes a live agent from a bare watcher (increment 1), propagated
  over presence as a raw `agent: active|idle` column (increment 2).
- **#173/#175 — type-gate untrusted set-membership in the receive path.** A
  crafted authenticated frame carrying a non-string where a state/status
  string is expected (`[] in <set>` raises) can no longer crash the receive
  loop; both the recv/agent/status ingest and the wire-reachable
  message/task-ingest sites are gated.
- **#158/#176 — wake-path watchdog.** The #161 heartbeat proves frames
  arrive; #170 proves an agent woke; #176 closes the field-confirmed blind
  spot between them — deliveries buffered while no agent drains them. A
  watchdog on the presence holder detects undrained deliveries, SIGTERMs a
  positively-identified wedged defer-mode hook (relay-mode and unknown-mode
  holders are never touched; pid-reuse guarded) so the wake path re-arms at
  the next turn boundary, and surfaces the unarmed case via a loud
  classification log, a `mesh status` WAKE-STALLED line, and a `wake:
  ok|stalled` presence key. It cannot wake an agent no harness is waiting
  on — that state is bounded and visible, not abolished.
- **#168 — no session URLs in the public record.** Repo-level
  `attribution.sessionUrl: false` plus a CLAUDE.md hard constraint: no Claude
  session links or trailers in commits, PRs, or issues.

## 0.19.0

Receiver-stall visibility, a Windows signing fix, and the first real-exec slice
of supervised adoption (dormant, default-off).

- **#158/#161 -- witness silent receiver stalls.** A durable per-node receiver
  heartbeat (refreshed on every relay event) plus a `mesh status`
  ok / STALLED / no-heartbeat indicator, so a watcher that holds its
  subscription but stops delivering is visible instead of silently dark.
- **#159 -- Windows ssh-keygen override.** `A2ACAST_SSH_KEYGEN` selects the
  ssh-keygen binary (fail-closed on a bad override), fixing the stock Windows
  OpenSSH 9.5p2 `-Y sign` ~60s timeout race that degraded frames to unsigned.
- **#162 -- adopt-supervise `--exec` (#62 B2b-2a), dormant.** An operator
  running `mesh adopt-supervise --exec` can apply an owner-signed
  git-working-tree upgrade (fetch-verify the signed SHA, checkout, import
  smoke-test, local rollback on won't-start). Default dry-run; a received
  frame never auto-execs.

## 0.18.0

Owner-signed approvals over the mesh (#62): the "authorize once → every node
observes" pipeline lands, still **log-only / dry-run and gated default-off**.
An owner-signed grant now distributes over the mesh and each node verifies it
against its pinned owner key — but no node yet acts on one. This is the
wire-format adoption release ahead of any real fleet adoption; every step is
off by default and inert on 0.17.0 nodes.

- Owner-signed approval frames (#62 Phase A): `mesh approve <descriptor>
  --broadcast` distributes a single-use, expiring, owner-signed grant; a
  receiving node verifies it against the pinned owner pubkey and logs
  `MESH_APPROVAL_OK` / `MESH_APPROVAL_BAD` — no action, no trust mutation, no
  nonce consumed (a pure-crypto verify). Old peers ignore the frame
  (forward-compatible).
- Gated auto-act scaffold (#62 Phase B1): an `act_on_approvals` opt-in —
  default off, settable only by the direct operator and never by any frame
  (a structural bootstrap denylist) — that, when on, logs
  `MESH_APPROVAL_WOULD_ACT` and still takes no action.
- Pending-adoption record (#62 Phase B2a): a verified `adopt-release` grant
  (opt-in on) writes an authenticated pending-adoption record — storing the
  owner-signed token + descriptor, not a bare hash — and logs
  `MESH_ADOPT_PENDING`. The receive path still runs no code.
- `mesh adopt-supervise` dry-run (#62 Phase B2b-1): an operator-run, opt-in
  supervisor reads a pending record, re-verifies the owner signature, detects
  the install type (uv-tool git/PyPI via the uv receipt, pipx, pip,
  source-clone), maps the pinned artifact to the upgrade operation it *would*
  run, and logs `MESH_ADOPT_WOULDEXEC` / `REFUSED` — refusing anything it
  cannot map to a known operation. It installs nothing and cycles nothing;
  the real package-manager exec is a separate, operator-gated slice.

## 0.17.0

Signed-approvals flagship groundwork (#62): the trust program's machinery
lands **observe-only and default-off**, so the fleet can adopt it before any
enforcement flips. Nothing here changes delivery or trust behavior by
default; every ratchet is gated on an explicit operator opt-in.

Signing, identity & enrollment (#62 / #74 / #76 / #93):
- Signature-verdict enforcement (#74) is present but **dormant** — the
  receiver can refuse a frame that lies about its signature, gated behind
  `enforce_verdicts` (default off). Downgrade-ratchet observation logs
  `MESH_DOWNGRADE` when a pinned peer drops to unsigned (observe-only, #132).
- mcp-serve now signs with the node's pinned per-harness key (#144): the
  server threads its `--harness` into the sign path instead of falling back
  to the generic key, and `claude-setup`/`copilot-setup` emit `--harness` so
  the watcher's key matches the CLI's. Restart mcp-serve to adopt.
- Best-effort signing that degrades to unsigned now warns loudly on stderr
  (#145) instead of failing silently.
- Owner-signed enrollment (#76): membership certs (mint / verify / cache,
  log-only) make a pin non-self-service; the attended-mint property is
  verified at use, not only at creation (#110). The owner ceremony is
  hardened (#87 F1–F3) — the signing env is scrubbed, rotation refuses
  unattended postures, and the passphrase never touches a2acast.
- Owner-signed revocation (#76): full-set revocation lists, sender-asserted
  freshness (observe-only `REVGAP`, #131), and a non-suppressible pull
  transport (#133) — all observe-only, off by default. A suppression vector
  (a spoofed unsigned freshness hint weaponizing an honest peer's reply) was
  found and closed with load-bearing tests.
- Rename pin-migration + tombstones (#93): a key-verified rename can carry
  its pin to the new name, gated behind `rename_migration` (default off);
  rename-observation frames and revocation records are otherwise log-only
  (#102). The receiver derives the old name from the signed sender, never a
  spoofable field.
- Fleet-scaled cap on TOFU pin-store growth (#76): a malicious member can no
  longer grow the pin store unbounded — the cap raises a loud refusal at the
  ceiling, and `note_peer`'s roster and `peers.json` are bounded the same way.
- `mesh pins-audit` — a pairwise key-collision scan over the pin store
  (#116); the node-signature base is reconstructed from the received wrapper
  so new signed fields carry forward to older peers (#120); repeatable
  KEY_MISMATCH rehearsal tooling (#62).

Reliability & delivery:
- Oversize payloads chunk under the inline limit and reassemble (#66); a
  chunk reporting a mismatched count no longer discards a legitimate
  in-flight message (#128). Attachment TTL is warned on both ends of the
  cliff.
- mcp-serve piggybacks pending deliveries on every tool result (#121); a
  dying receive loop emits a last-gasp crash frame (#123); presence carries
  process posture — what is listening, not just that it was seen (#122).
- Phase-A observability is durably logged so #62 soak evidence survives a
  hook wake (#129).

Harness & platform:
- Setup preflights the Windows registry PATH so plugin hooks actually arm
  (#105); strict std streams degrade at CLI entry so a delivered send cannot
  exit non-zero (#98). Windows PATH requirements documented (#90).

Tests, CI & docs:
- CI enforces mesh.py stdlib-only (#138), mirrored by a local test so a
  third-party import fails before push. CONTRIBUTING.md carries
  agent-readable contribution rules; the README leads with the PyPI install.

Known limb (#90): on Windows a Stop-hook-*spawned async* claude-hook still
exits before it can independently receive. The in-session Stop-hook wake
path itself works — it carries live fleet coordination — so delivery
degrades to delayed redelivery, never loss.

## 0.16.1
- Fixed the two delivery-loss mechanisms behind #86, validated live on all
  three platforms:
  - Every delivery path now writes the per-node activity file (shared
    `_activity_line` format), so a lifecycle hook deferring behind a plain
    `mesh watch` presence holder wakes on delivery instead of starving and
    dying silently when the watch exits.
  - Transport checkpoints (cursor + replay fingerprint) run only after the
    delivery handoff — inline after a successful emit for standalone
    watches, deferred until after the hook's own output in hook relay mode.
    A death before handoff leaves the frame re-deliverable: at-least-once
    instead of silent loss. Undeliverable frames are consumed exactly once
    and leave a visible activity trace.
  - Activity previews decode message envelopes (not just tasks), keeping
    the watch and MCP writers in agreement.
- The agent-session watch warning now names the one-shot
  `mesh watch --timeout` re-arm fallback for harnesses where the hook
  cannot wake yet; the defer-mode wake summary mentions the CLI drain
  commands for sessions without MCP tools.
- Known limb, tracked in #90: on Windows the harness-spawned async
  claude-hook process exits instantly and never receives — with this
  release that degrades to delayed redelivery, never loss; the one-shot
  re-arm loop remains the Windows posture.

## 0.16.0
- Per-node message signing (#62 phase 2): every node holds its own ed25519
  key, signs its outbound frames over the wire AAD + payload, and classifies
  inbound frames against a locally-pinned identity (verified / unverified /
  unsigned / mismatch). Trust is trust-on-first-use; the receive path is
  NON-ENFORCING — it surfaces a verdict and pins peers but drops no frame.
  Enforcement + the downgrade ratchet are still pending (#74). A node
  generates its key on first send, so upgrading an existing node is enough.
- Owner keys are passphrase-protected by default (#64): minting an approval then needs the passphrase on the terminal, which a harnessed agent cannot answer — so an owner signature proves a human acted, not just that a process read the key. `owner-init --no-passphrase` keeps the old unprotected key (loudly warned); `owner-trust --replace` rotates the trusted owner key.
- Owner-trust now prints a SHA256 fingerprint and requires a terminal
  confirmation before pinning; the owner private key's permissions are
  asserted (POSIX mode + Windows ACL).
- Bound the replay ledger with time-based eviction (#77); mesh status shows
  the held count.
- mesh peek no longer mislabels expired large-message attachments as
  [UNVERIFIED] (#65).
- `mesh watch --follow` warns when it would be a write-only pipe in an agent
  session; join steers to the lifecycle hook (#57).
- `mesh mcp-serve --harness` resolves identity from the pin at each startup,
  so `mesh iam` renames take effect (#59, #60).
- The generated .gitignore now uses a `.meshwire.*` glob, closing a gap that
  left the owner private key stageable.

## 0.15.1
- Security: invite bootstrap blocks now download `mesh.py` pinned to the
  inviting node's release tag (`v<VERSION>`) instead of the tip of `main`,
  so a bad or malicious push to main cannot break or compromise future
  joins.
- Add GitHub Actions CI: the unittest suite runs on Linux, macOS, and
  Windows across Python 3.8–3.13, a consistency job keeps
  `mesh.py`/`pyproject.toml`/plugin manifest versions in lock-step, and a
  gitleaks job scans full history for leaked secrets on every push.
- Add a PyPI publish workflow (trusted publishing, runs on GitHub release).
- Docs: list `mesh_delegate` in the MCP tools roster.

## 0.15.0
- Add an opt-in machine-wide worker pool with distinct Codex, Copilot, and
  Goose/Ollama identities.
- Add versioned isolated-worktree jobs, structured branch/commit results, and
  recipient-scoped task records so parallel supervisors cannot race.
- Add journaled execution, reply-only retries, health/cooldown routing, MCP
  delegation, conservative worktree cleanup, and macOS LaunchAgent lifecycle.
- Preserve the existing default-off, default-empty-allowlist Codex supervisor
  and document that worktrees are not security sandboxes.

## 0.14.1
- Fix (security): config writes are now durable read-modify-write under a
  lock — an incoming message (note_peer) or `mesh iam` can no longer clobber
  the curated `exec_allow` allowlist (#30). Also covers cmd_iam and my_node.
- Fix: `mesh codex-supervise` reloads the allowlist each poll, so
  `mesh codex-allow` takes effect on a running supervisor (#31).
- Fix: the supervisor runs its own relay receiver, so a headless Codex node
  (no session open) actually receives tasks instead of starving (#32).
- Fix: identity migration no longer claims the bare hostname — a node whose
  established name was just the old hostname default keeps the harness-aware
  `<host>-<harness>` name (#33).

## 0.14.0
- Identity migration: `mesh claude-setup`/`codex-setup` migrate an established
  generic `.meshwire.node` name into the per-harness pin, so nodes known by a
  plain name keep their identity under the harness-aware naming rule (no dark
  nodes on upgrade). `codex-setup` registers the migrated identity.
- Codex autonomy (opt-in): new `mesh codex-supervise` can make a joined
  Codex node an autonomous peer — it runs each delivered a2a task through
  `codex exec` and replies over the mesh.
- Security: autonomous execution is off by default. `mesh codex-setup` sets
  up presence only; pass `--supervise` to launch the actor. Even then,
  nothing auto-runs until you explicitly trust peers with
  `mesh codex-allow <node>` — a curated allowlist (default empty), not the
  auto-grown roster.
- Sandbox: `codex exec` runs `--sandbox read-only` by default
  (defense-in-depth; note that read-only still lets a task read and return
  repo contents, so only allow peers you trust). `--supervise-sandbox`
  widens it.
- Reliability: a task is claimed (state "working") before exec so it can't
  double-run; `codex exec` is bounded by a 600s timeout; failing or
  timed-out tasks are retried up to 3 times then dead-lettered (state
  "failed"); tasks stranded in "working" by a crash/stop are requeued on
  supervisor startup.

## 0.13.0
- Presence on session open: `mesh mcp-serve` is now the uniform presence
  watcher for Claude Code (`mesh claude-setup`), Codex CLI
  (`mesh codex-setup`), and Copilot (`mesh copilot-setup`) — the node
  answers pings, acks, and captures messages while the agent is idle.
- Single-subscriber rule: hook wake-watchers wait on the local delivery
  buffer when a presence server is live (no double relay subscription).
- Watch loop resubscribes forever on unexpected errors.
- Per-node activity files; harness-aware `mesh join`; `agent-hook-cleanup`
  resolves identity per-harness.
