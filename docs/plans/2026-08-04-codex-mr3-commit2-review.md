# Codex review record — MR-3 commit 2 (captaind channels-subsystem scaffold)

**Subject**: the second commit of the captaind-channels MR: "captaind: the
channels subsystem scaffold, an inert embedded LDK node", bookmark
`ark8-channels-stage1-captaind` (parent = the commit-1 node commit
`d7bab25c`). Hashes across the arc: `05867e06` (round 1) → `d43cf05f`
(round 2) → `5d8dc815` (round 3) → `fd2139f5` (round 4) →
**`88937cb2` (round 5 = PASS)** → `07997102` (post-PASS: the window arithmetic extracted + unit-pinned with the round-4 counterexample) — one commit, amended in place.

**Scope**: `[channels]` OptionalService config + `supports_channels`
tracking enablement; the V55 migration (five tables); the postgres
persistence seams (dedicated-runtime `DbExecutor`, monitor `Persist`,
manager async `KVStore`); the capture broadcaster; the refuse-everything
event handler; node boot/reload with the dedicated hardened-child seed;
the chain feed; the peer listener; `Server::start` wiring; e2e tests.

**Process**: five codex rounds (REWORK ×4 converging 8 → 5 → 2 → 1 open
findings, then PASS). Round 4's first dispatch died to a session-lifecycle
kill (no verdict); rerun at Greg's direction — and it caught a real bug
the whole test battery could not see (below), vindicating the extra
round.

## Round 1 — REWORK (5 Important / 3 Minor)

1. **I — bootstrap not catch-up safe**: the driver started on a stale
   view, and the SyncManager never replays already-indexed blocks to a
   NEW listener — enabling channels on an established database left the
   node behind forever; an idle-tip restart never started the peer
   listener at all.
2. **I — reorg rollback unsafely anchored**: empty header buffer at boot
   (1-block reorg after the first fed block = spurious
   deeper-than-buffer), withdraw-before-anchor-check partial mutation,
   merged relevant-txid lists cross-applied between manager and monitors.
3. **I — a failed monitor write left the server limping**: LDK panics the
   task seeing `UnrecoverableError`, and inbound peer futures are
   untracked tasks whose panic only gets logged.
4. **I — missing `[channels]` broke pre-existing config files** (no serde
   default).
5. **I — parent-watch resolutions without chain evidence** representable.
6. **M — V55 violated CONTRIBUTING/postgres.md conventions** (plural
   names, protocol-id PKs, BYTEA for queryable ids, booleans).
7. **M — channel_scid permitted SCID-unencodable values.**
8. **M — tests missed the risky paths** (advertisement-only boot test; no
   persistence-seam/reorg/constraint coverage).

## Rounds 2–3 — the fixes that survived refinement

- **Chain feed redesigned** (r2 caught per-block `best_block_updated`
  during historical replay regressing already-ahead components; r3 caught
  replay-before-withdraw violating LDK's unconfirm-before-reconfirm plus
  the above-shortened-tip edge): final shape = withdraw every stale
  confirmation FIRST (stale = above tip, or recorded hash no longer
  canonical), replay the canonical chain confirmations-only from
  `min(lowest_node_view, tip − DEEPLY_CONFIRMED)` (the server-wide
  100-block finality horizon standing in for exact fork discovery), one
  best-block announcement at the tip; live blocks confirm always but only
  ADVANCE the announced view; the reorg path fetches and verifies the
  fork from the canonical chain before touching anything (mismatch =
  transient, re-delivered) and is the one sanctioned backward move. The
  header buffer was deleted outright — no depth cliff exists.
- **UPSTREAM BUG FOUND AND FIXED (r3): the SyncManager committed its
  rollback before its listeners ran** (`update_sync_height` first in
  `org_out_blocks_above`), so a listener error during a reorg was never
  retried: the next cycle saw no reorg, skipped `on_reorg`, and collided
  forever inserting over the orphan rows — the module's own documented
  retry contract was broken, and the channels subsystem is the first
  listener whose `on_reorg` can actually fail. The rollback now commits
  last (listeners → DB delete → sync height), matching `add_block`'s
  ordering. Regression-tested with a fail-once listener driving a real
  reorg; the test is **mutation-verified** (restoring the old ordering
  fails it: one delivery and a wedge instead of a retried second
  delivery).
- **MonitorPersister requests `rtmgr.shutdown()` before returning
  `UnrecoverableError`** — a node that cannot persist must not keep
  running.
- **`OptionalService` gained `Default` (Disabled) + `#[serde(default)]`**
  on the channels field: configs predating the subsystem parse unchanged.
  (Codex's suggestion to retain/validate DISABLED payloads was declined:
  discard-on-disabled is the established watchman semantic, and a
  disabled service validating trivially matches the design note.)
- **V55 rewritten to the house postgres conventions** (singular names,
  synthetic ids + natural keys UNIQUE, hex TEXT, timestamps not booleans,
  motivated-BYTEA comments for the LDK blobs); watch resolutions are
  all-or-nothing with their chain evidence and `cardinality(...) >= 1`
  on the retained responses — `array_length` of an empty array is NULL
  and a NULL CHECK passes; my own constraint test caught that, and the
  regenerated schema.sql initially lagged the fix (r3 minor).
- **SCID columns carry 24-bit encodability bounds**; `updated_at` added.
- **Tests**: the boot e2e proves an ACCEPTING peer listener (retried
  connect); a real `invalidateblock` reorg e2e synchronized on captaind's
  block cursor (old tip indexed before invalidation, new tip after — so
  the reorg is provably delivered); persistence-seam tests exercise the
  `DbExecutor` write path and the `ManagerStore` KVStore contract; schema
  constraints are negative-tested. Found along the way: **all in-process
  test servers bind the same fixed public-RPC port** (a pre-existing
  harness collision that kills whichever server loses the race under
  parallel runs — worth an upstream fix); the channels tests use
  ephemeral ports.

## Rounds 4–5 — the replay-window anchor

Round 4 confirmed the SyncManager fix (with its mutation-verified test)
and both minors, raised nothing new, but caught the catch-up replay
window anchored to the WRONG end: `min(lowest_node_view,
tip − DEEPLY_CONFIRMED)` measures the finality horizon from the CURRENT
tip, while reorg depth is measured from the node's OLD view — after a
long downtime spanning an offline reorg (saved view 1000, fork 949, tip
1200), a withdrawn confirmation re-included at height 950 was never
replayed. Fixed as codex prescribed:
`lowest_node_view().min(tip).saturating_sub(DEEPLY_CONFIRMED)` — for any
permitted fork `F ≥ old_view − 100`, replay begins no later than `F + 1`.
Round 5 verified the closure and the edges (shortened tip, near-genesis
saturation, the fresh node's ~200-block replay): **PASS, no findings.**

## Final state and verification

- Commit **`88937cb2`**, stacking: opener → protocol → node (commit 1,
  PASS after 3 rounds) → this (PASS after 5).
- `cargo check --all --tests` green; unit suite **522/522**; clippy:
  bark-channels 0 warnings, bark-server byte-identical to the 235
  baseline; schema.sql regenerated in sync with V55.
- E2e against real bitcoind v29 + postgres 16: the channels set (boot
  accepting/disabled/reorg + persistence seams + constraints) plus both
  block_index tests — 7 tests green across repeated full-group runs; a
  94-test non-channel slice earlier in the arc at 91 passed (3 = the
  machine's known lightningd env gap).
- Deferred, tracked: the offline-reorg re-confirmation BEHAVIORAL e2e (a node holding a relevant transaction across a below-view fork) needs a live channel — comes with the admission work, alongside the daemon-level restart (reload-path) e2e which needs daemon
  config plumbing (comes with the admission commit); the reserve default
  amounts remain provisional until the response/sweep package weights
  exist (the ledger itself was reviewed in commit 1). An upstream issue
  for the SyncManager reorg-retry bug was drafted for Greg to file.
