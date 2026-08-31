# Codex review: MR-6 commit 1 — the feed/drain barrier (3 rounds to PASS)

Commit: `channels: acknowledge chain updates only after their events are
durable` (bark-stage1 `e894c4f78` at PASS). Reviewer ran read-only
against the repo with pinned LDK 0.2.4 sources.

Batteries at PASS: workspace units 598/598, bark-channels 22/22 (13
release-contract + 5 new feed-barrier + 4 config), bark-sdk channels
19/19, server channel 22/22.

## What the commit does

Replaces `lightning-background-processor` with an owned driver loop that
is the single owner of the node's chain feed AND event drain. Chain
updates arrive through a `ChainFeed` mailbox; the driver applies them to
manager+monitors, drains both queues to quiet, persists the manager blob,
and only then acknowledges — so the server's shared block cursor
(`store_block` runs only after `on_block_added` returns Ok) and the
release fed-mark wait for durability by construction, and the client
exit's terminal accounting needs no output counting: terminality = no
opaque LDK balances AND every persisted descriptor spent by a confirmed
sweep, with the legacy amount-ledger dual-read retiring at a durable
full-drain watermark.

## Round trail

- **R1: FAIL — 3 P1 / 4 P2.**
  - P1 ack-after-post-drain-event-production → loop reordered: persist +
    ack (+`settled`) immediately after the quiet drain; peer events and
    HTLC forwards run after.
  - P1 `driver_running` TOCTOU / failed-driver-as-dormant → explicit
    `FeedRoute` lifecycle (Boot | Running | Stopped) under one mutex;
    Boot applies directly UNDER the lock, Running set before the driver
    task first polls, Stopped refuses feeds.
  - P1 no stable fence for terminal reads (a concurrent feed — another
    channel's open advancing the shared best block — can drop a balance
    while its descriptor write is in flight) → fenced read order in
    `pending_claims`: balances, then an empty-ops barrier flush, then
    the rows; monotonicity argument: an output matured before the
    balance read has its descriptor committed by flush-return, one
    matured after still showed its balance.
  - P2 repeated-events replay hole (LDK IGNORES handler errors on
    `BumpTransaction` events — cleared-on-read, regenerate only on later
    blocks/rebroadcast) → the one client handler path that replayed a
    bump failure without stopping (channel-record read) now stops the
    node; driver doc states the caveat.
  - P2 dormant-node exit progression → see R2 (first fix over-gated).
  - P2 watermark one-shot write → retried (5s backoff) until the node
    stops.
  - P2 missing monitor-side test → added
    `test_feed_ack_holds_for_matured_spendable_output`: a real
    force-close, the counterparty-balance maturity queues
    `SpendableOutputs` inside a fed advance, a failing handler holds the
    ack, healing delivers the descriptor before the ack.
- **R2: FAIL — 1 P1 / 1 P2.**
  - P1 the watermark retry loop was `tokio::join!`ed with the driver: a
    dead driver + failing store kept `driver_running`/`Running` live
    indefinitely → the watermark writer is now a SPAWNED task; driver
    end-of-life bookkeeping runs unconditionally.
  - P2 the R1 health gate over-blocked manual-sync mode (dormant node =
    exits permanently deferred through deadlines) → gate narrowed to
    STOPPED (driver ran and ended / stop token fired — feeds would only
    be refused); a dormant never-ran node progresses the working states
    exactly as before the commit (broadcast captures are durable without
    the driver; terminal separately gated on healthy + settled).
- **R3: PASS** — "both Round-2 fixes are sound, and dormant behavior
  matches the parent commit."

## Found by e2e during the commit (pre-codex)

The barrier's immediacy broke the cooperative tail: the chain advance
lived INSIDE `pump`, and a descriptor the advance materializes became
visible to that same pump call — which then derived a competing spend of
the shutdown output whose pre-built claim (seeded by the tail's
adoption, which runs BEFORE pump) was already confirmed; the rival can
never confirm and `all_confirmed` wedged
(`channel_close_fallback_won_rides_cooperative_tail` caught it, solo-
reproducible). Fix: the advance is its own driver-seam step
(`advance_chain`), and every exit tick orders advance → adopt → pump.

## Key design decisions (for Greg's eyeball)

1. **The barrier's scope is synchronous production**: an ack covers every
   event the ops queued inside the `Confirm` calls; peer-round-trip
   events (`ChannelReady` completing) belong to no update's barrier, and
   nothing event-producing runs between a drain and its ack.
2. **Bump events cannot rely on replay** (LDK clears them on read and
   ignores handler errors): every handler path that fails to capture one
   durably stops the node.
3. **`FeedRoute` lifecycle** (Boot/Running/Stopped) instead of a bool:
   Boot = wallet-load catch-up and manual-sync mode (direct, under the
   route lock); Stopped refuses feeds rather than lying.
4. **Terminal reads are fenced** (balances → flush → rows), which is the
   whole consistency story against concurrent feeders — no locks over
   the exit tick.
5. **`advance_chain` is explicit** in the exit driver seam; tick order
   advance → adopt → pump is load-bearing (the rival-sweep hazard).
6. **Manual-sync (dormant) semantics preserved**: working exit states
   progress (broadcast captures are durable without the driver), the
   claim parks before terminal; only a STOPPED node defers the tick.
7. **Legacy expectation ledger**: never written again, dual-read until
   the durable full-drain watermark (write retried while the node
   lives), then retired permanently. `record_expected_maturity`, the
   structural floor and `min_expected` are deleted.
8. **Server ordering upgrades for free**: the release fed-mark — which
   previously could not be ordered against manager encode times at all —
   now sets only after the same barrier; boot re-feed stays as belt and
   braces. Reorg withdrawal keeps per-component precision via scoped
   unconfirm ops.
9. The regenerated sqlite schema dump also picked up the `bark_channel`
   `exit_origin` column from an earlier migration that had not been
   re-dumped.
