# Codex review record — MR-3 commit 4 (the gate's release half + the channel lifecycle's chain side)

**Subject**: the fourth and final commit of the captaind-channels MR:
"captaind: release the gate — registration, the confirmation feed, the
parent-exit watch, expiry", bookmark `ark8-channels-stage1-captaind`
(parents: the node `d7bab25c`, the scaffold `89c252e7`, the admission
commit `bbf2a492`). Hashes across the arc: `59c08edf` (round 1) → `e671a2b3`
(round 2) → `ab4520a0` (round 3) → `e743201f` (round 4) →
`72e523e1` (round 5) → `2e8a2c7b` (round 6) → `7024df5c` (round 7) →
`a5a798df` (round 8) → **`6760e83c` (round 9 = PASS)** — one commit,
amended in place.

**Scope**: linearized registration riding the signed-chain upload
transaction; the level-triggered release reconciler (SCID
persisted-before-feed, anchor-revalidated confirmation feed, the manager
durability barrier); the sanctioned `ChannelReady`; the parent-exit
watch's chain side (facts, resolutions with evidence, reorg reopen); the
foreclosure → terminal transition with the logical LDK close; the
channels ⇒ embedded-watchman config invariant; the depth guard; three e2e
suites plus postgres coverage.

**Process**: nine rounds. The CORE — registration linearization, the
release feed, watch facts, foreclosure→terminal, SCID, the depth guard —
was clean by round 4; rounds 5–9 were a sustained convergence on ONE
seam, the startup/activation ordering and its admission-anchor twin. The
lesson of that tail: each round I had hand-rolled a chain-synchronization
primitive ALONGSIDE the SyncManager (a bespoke catch-up, then a
bitcoind tip-poll handshake, then a height marker), and each had a race
codex caught; the fix that finally held pushed the handoff ONTO the
infrastructure (the SyncManager's own `sync_height == chain_tip`
barrier) and used a proper change token (a scan epoch bumped on block
AND reorg). Grinding it now was deliberate (Greg's call): part 4 adds
payments, which removes the no-payments invariant that makes the
pre-acceptance window benign, so a deferred hole would reopen next MR
with real money in it.

## Round 1 — REWORK (4 Critical / 4 Important / 1 Minor)

1. **C — reorg and release were not serialized**: the generation counter
   was check-then-act; a reorg could bump it after the final check but
   before the header fetch or between the manager and monitor feeds,
   violating `Confirm`'s en-bloc ordering.
2. **C — the durability barrier could be overwritten by a stale
   snapshot**: generations were allocated at write-call time, but the
   DRIVER encodes earlier — a pre-feed encoding could acquire a newer
   generation and clobber the barrier's post-feed snapshot.
3. **C — the row lock did not linearize the watch predicate**: the scan
   judged armed/unarmed from its cache, then merely held the lock while
   storing that stale judgement — a registration committing in between
   could see its armed watch resolved as unarmed.
4. **C — first-sight reconciliation omitted ancestor history**: fresh
   watches checked only the final exit and responses; an ancestor swept
   before the row existed left the final tx simply absent, and
   registration would release a channel whose backing can never exist.
5. **I — terminal outcomes incomplete and non-atomic**: an armed foreign
   spend of the input was not terminal, and the ancestor-swept terminal
   mark ran in a separate transaction after the resolution.
6. **I — a persisted SCID survived an anchor change with a stale
   height**, bypassing the uniqueness protection.
7. **I — terminal rows were excluded from the boot withheld-seed**, so a
   crash before terminal reconciliation plus a replacement chain carrying
   the real bridge could re-confirm under another header.
8. **I — registration reversed admission's lock order** (tree rows before
   the channel row), a deadlock with the idempotent re-cosign.
9. **M — missing concurrency coverage** (concurrent SCID allocation,
   crash/interleaving cases).

**Verified clean in round 1**: the one-transaction registration with
whole-upload rollback and idempotent retry; the sanctioned-`ChannelReady`
trichotomy; SCID band arithmetic and collision enumeration; same-header
re-feed idempotence; per-outpoint expected-spender derivation (sibling
exits cannot false-positive; armed prefixes do not resolve; the bridge is
excluded exactly); the generic-watchman response routing and the config
invariant; the capture-broadcaster suppression; the depth guard; both e2e
suites non-vacuous.

## The rework

- **One chain lock** serializes block delivery, the whole reorg sequence,
  and the release feed's validate-and-feed critical section; the
  generation counter survives only as a cheap pass-abort.
- **Staleness-proof persistence**: `ManagerStore` now re-encodes the live
  manager under its write lock on EVERY write (the handed bytes are
  ignored) — the freshest state wins by construction. Soundness:
  monitors persist synchronously before any state they secure is acted
  on, so a fresh manager encoding is always covered by durable monitors.
- **Facts, judged under the lock**: the scan emits chain facts with
  evidence; each is judged in one row-locked transaction against the
  watch row as it is THEN, with terminal outcomes (foreign input spend,
  unarmed final exit, ancestor sweep) committing atomically with their
  resolution.
- **The viability walk** (`exit_chain_foreclosed`): a foreign spend of
  any exit-chain ancestor output refuses registration (under the lock)
  and the cosign (as an admission screen); the live path needs no
  backfill because the sync manager replays every block to the listener
  across downtime, and pre-cosign foreclosures are refused up front.
- SCID allocations re-walk when their anchor moved; terminal rows seed
  the boot filter; registration locks channel rows (sorted) before the
  tree update; concurrent-allocation and reallocation regressions added.

## Round 2 — REWORK (1 Critical / 2 Important / 1 Minor); R1-1/3/5/6/7/8 closed

The round-1 fixes held except two whose replacements had their own flaws:

1. **C — historical watch delivery still had a gap**: disabling channels
   while the shared cursor advances skips blocks the watch can never
   revisit; plus the pre-cosign window between the viability walk and the
   watch row's insertion.
2. **I — the fresh-encode store violated `KVStore::write`** (it
   deliberately persisted bytes other than the handed ones — stale-safe,
   but contract-invalid).
3. **I — registration held database locks across an unbounded RPC walk**
   (up to depth × channels sequential Core calls with tree and channel
   rows locked; ten-connection pool).
4. **M — interleaving coverage still thin.**

**Round-2 fixes:**

- **The disable-guard**: captaind refuses to start with `[channels]`
  disabled while any cosigned/registered/terminal row exists — the
  monitors, watch and confirmation filter must never go blind over live
  channels, and the cursor may not advance past blocks they care about.
  With the guard, the coverage ladder closes: pre-cosign history =
  admission's viability screen + DB swept-marking; the
  screen→row-insert window = registration's walk; the walk→commit window
  = live listener facts refused by the transaction's resolution re-read;
  post-commit = the armed watch (cursor-lag degrades to the ordinary
  armed-watch race the response answers). Spec amended accordingly
  (the converse of §2a's invariant).
- **Exact-bytes store + acknowledgement barrier**: the store persists
  exactly the handed bytes (the driver is the sole, sequential writer);
  the reconcilers' barrier became `await_fresh_persist` — resolve only
  once a write that STARTED after the call completed, i.e. codex's
  original round-1 suggestion. Timeout → the level-triggered caller
  retries. Postgres tests pin both properties.
- **Registration split**: all RPC preconditions (state screen, recorded
  resolution, final-exit look, viability walk) run before the
  transaction and mutate nothing; the transaction is pure database work,
  with the in-tx resolution re-read as the late-refusal authority.
- The restart e2e gains the disable-refusal case.

## Round 3 — REWORK (2 Critical / 1 Important / 1 Minor); R2-3 closed

Round 2's acknowledgement barrier was still unsound — the started-counter
advances at write-call time but the driver ENCODES before calling the
store, so a pre-feed encoding could satisfy the barrier (and the sole
fresh write could start before the barrier sampled, timing it out). The
ancestor cursor-lag window also remained (a sweep confirming between the
walk and the commit, its block undelivered — and unlike final exits,
ancestor foreclosure is terminal, not answered by the armed response),
plus a stale release worklist could feed a channel a block had just
walked terminal.

**Round-3 fixes:**

- **The barrier is deleted, not repaired.** No write-side acknowledgement
  can prove encode ordering against `process_events_async`, so the design
  stops needing one: `confirmation_fed_at` is a per-process latch, and
  boot clears every latch and re-feeds all registered channels — sound
  because a same-header re-feed is idempotent (round-1-verified) and a
  moved anchor is withdrawn by the catch-up first. The terminal
  reconciler's level check is the live manager, so an unpersisted close
  re-drives after reload. `ManagerStore` is a trivial exact-bytes
  KVStore. The spec's §5 barrier prescription was amended to record why
  it changed.
- **The cursor handoff**: the tip is captured after the precondition
  walks, and the registration transaction requires the block listener's
  cursor to have reached it (transient error otherwise). Everything at or
  below the screened tip is walk-covered or fact-covered; beyond it is
  indistinguishable from post-registration and resolves to terminal when
  the fact lands (no funds can move meanwhile — stage 1 has no payments).
- **The in-lock release re-read**: the feed skips unless the row is still
  `registered` when read under the chain lock.
- Postgres pins the latch lifecycle (mark → boot-clear → re-feed →
  re-mark) in place of the deleted barrier test.

## Round 4 — REWORK (2 Critical / 1 Minor); R3-1/3/4 closed

The latch design held; two ordering holes remained around it:

1. **C — startup exposed channels before watch catch-up**: boot replayed
   blocks into LDK only, then activated the peer listener and reconcilers
   before the sync manager delivered watch scans — a sweep mined while
   down left a channel operable until live delivery terminalized it.
2. **C — the cursor handoff compared height without branch identity**: an
   equal-height reorg let a stale-branch cursor satisfy the screen.
3. **M — stale barrier documentation** (module docs, commit message,
   restart-test text, the now-unused store field).

**Round-4 fixes:**

- **Two-phase startup**: before the peer listener, the latch clear and
  the reconcilers, the subsystem scans every block above the shared
  cursor through the watch/foreclosure machinery (the cursor only
  advances once every listener processed a block, so at-or-below was
  scanned in the previous life; live sync re-delivers the rest
  idempotently). New e2e: a cosigned channel's presigned exit raw-mined
  during an outage is TERMINAL by the time start() returns, and a late
  registration refuses — with a load-bearing pre-outage liveness assert
  (an early draft placed the scenario after the expiry grinding, and the
  assert correctly caught the watchman's own expiry sweep foreclosing
  the channel first).
- **Branch-identity handoff**: the screened tip is a BlockRef; the
  registration transaction requires the indexed block at that height to
  carry the screened HASH.
- Documentation swept to the latch design; the unused store field
  removed; spec §5 documents the handoff.

## Round 5 — REWORK (2 Critical / 3 Major / 1 Minor); R4-2 closed

The hand-rolled startup phase had its own holes — codex's structural
point: it badly duplicated what the sync infrastructure should provide.
Stale tip capture, height-only cursor, peers activating before
terminal/watchman convergence, a genesis-to-tip scan on fresh databases,
an unproven e2e ordering, the admission screen→insert window for
never-registered rows, and residual stale docs.

**Round-5 fixes:**

- **Dormant start + awaited activation**: the hand-rolled walk is
  deleted. The subsystem starts dormant (node, LDK catch-up, driver,
  bound-but-silent peer socket); the server registers all listeners,
  starts the SyncManager and the watchman, drives every listener to an
  EXACT captured tip (BlockRef, verified by indexed hash, re-captured if
  the chain moves), and only then calls `activate()`: terminal channels
  closed and VERIFIED gone from the manager (fail-closed), latches
  cleared, reconcilers spawned, peer acceptance last.
- **The terminal-reconnect e2e**: a foreclosure recorded while down;
  on restart the reconnecting LDK client receives `ChannelClosed` and
  ends with zero channels — the close verifiably preceded peer
  acceptance.
- **Admission anchors like registration**: the indexed cursor BlockRef
  is captured before the chain screens and must be UNCHANGED (by hash)
  at the atomic commit — a mid-admission block delivery aborts
  transiently, closing the screen→insert window.
- Genesis scans are structurally gone; docs and the commit message swept
  to the final design.

## Round 6 — REWORK (2 Critical / 2 Major / 1 Minor); R5-4 closed

Round 5's dormant/activate structure was right, but the hand-rolled
activation drive and admission anchor still had ordering holes: the
captured tip was never revalidated against bitcoind after the drive (a
mid-sync reorg could activate on a stale branch, and a forward extension
left the "recapture loop" not recapturing); the watchman's first pass
was unawaited; the admission anchor used the DB cursor, which LAGS the
listener's scan (the sync manager commits the cursor only after all
listeners return); and the new e2e proved a post-start reconnect, not
pre-acceptance ordering.

**Round-6 fixes:**

- **Quiescent-tip handshake**: activation captures a tip, drives the
  listeners to it, verifies the indexed hash, then RE-READS bitcoind and
  requires the tip unchanged — looping on any reorg or extension. It
  activates only on a tip that stood still across the whole drive.
- **Scanned-height marker**: the subsystem tracks the highest fully
  scanned block (bumped at the end of `on_block_added` under
  `chain_lock`, so it leads the lagging DB cursor). Admission and
  registration capture it before their chain screens and, under
  `chain_lock` at commit, require it unchanged — serializing the
  watch-row insert against block delivery and closing the in-flight
  window with the right signal.
- **Log-ordering proof**: the terminal-reconnect e2e asserts the
  "logically closing terminal channel" line precedes the
  `ChannelsSubsystemStarted` slog — activation closes terminal channels
  before it spawns peer acceptance, and `start()` gates RPC on
  activation.
- **R5-2 held with pushback**: the parent-exit response deadline is
  `exit_delta` from the input's exit confirming, never peer-connection —
  in any stage — so no awaited watchman pass is built; the frontier is
  driven current as a listener and the watchman runs its first pass at
  startup. (Codex to confirm or refute in round 7.)
- Docs swept.

## Round 7 — REWORK (2 Critical / 1 Major / 1 Minor); R5-4 (again) closed

The round-6 primitives were still hand-rolled alongside the sync
infrastructure, and codex's point sharpened to: the handoff must come
FROM the SyncManager, external tip polling can't provide it; and a plain
height store is not a reorg-safe change token.

**Round-7 fixes:**

- **SyncManager-owned barrier**: activation waits on the sync manager's
  own `sync_height == chain_tip` (both set by its one loop — internally
  consistent, no external RPC), replacing the bitcoind tip-poll
  handshake. `activate()`'s terminal-close + verify now run under
  `chain_lock`. The residual last-block race is confirmed as the
  inherent reactive-foreclosure window (block-driven in every stage),
  not a startup hole — part 4's HTLC-safety layer owns it.
- **Scan-epoch change token**: `scanned_height` (a plain height, blind to
  equal-height reorgs) becomes `scan_epoch`, bumped at the end of BOTH
  `on_block_added` and `on_reorg`. Admission and registration capture it
  before their first chain screen and require it unchanged under
  `chain_lock` at commit.
- **Watchman prompt pass**: after the barrier (frontier caught up) the
  server triggers a watchman sweep, so an offline parent-exit response
  is driven on restart rather than after `process_interval`.
- Docs and the commit message swept to the SyncManager-barrier design.

## Round 8 — REWORK (1 Major / 1 Minor); R5-1-core / R5-2 / R5-3 closed

The SyncManager barrier and scan-epoch were confirmed sound; two tails
remained. The barrier awaited only `sync_height.changed()`, so a tip
REVERSAL to the durable cursor (chain_tip drops to the already-synced
height, no new delivery) could sleep activation until the next block;
and the commit message still carried the old quiescent-tip wording.

**Round-8 fixes:**

- The barrier now holds both the `sync_height` and (new)
  `chain_tip_watcher()` receivers and selects on EITHER changing, so a
  tip move toward the synced height wakes it and it converges.
- The commit message's activation/admission paragraph rewritten to the
  SyncManager-barrier + scan-epoch design.

## Round 9 — PASS (narrow confirmation; no findings)

R8-1 CLOSED (the barrier watches both values; tokio's version tracking
makes `changed()` miss nothing and not busy-spin; reversal-to-cursor
converges) and R8-2 CLOSED (message wording accurate). The residual
post-barrier reactive window is confirmed inherent and accepted.

## Final state at `6760e83c`

- `cargo check --all --tests` green; units 524/524; clippy ark-lib clean,
  bark-server at the 236 baseline.
- Channels e2e: the admission matrix (incl. the concurrent reap race),
  the real-establishment suite (release → ChannelReady → usable → HTLC
  probe → restart with the depth-guard and disable-guard refusals →
  reload/reestablish → the terminal-reconnect with its log-ordering
  proof), and the parent-exit watch lifecycle (armed response, unarmed
  terminal + late refusal, sanctioned bridge, expiry foreclosure, the
  offline-exit restart). Postgres: registration transitions, the SCID
  allocator (idempotence, collision walk, anchor-change realloc, 8-way
  concurrency), the feed/latch lifecycle, the watch resolve/reopen
  lifecycle.
- The single slice failure remains the machine's lightningd env gap
  (gmp version mismatch), pre-existing and unrelated.

**Accepted residual (part 4's to own, not a stage-1 defect)**: a channel
whose backing foreclosure confirms is closable one block later, in every
stage — foreclosure detection is reactive/block-driven. Under stage-1's
no-payments invariant nothing can move in that window; part 4's
HTLC-safety layer inherits it alongside the other no-live-HTLC
deferrals.
