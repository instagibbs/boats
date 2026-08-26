# Codex review: MR-6 commit 2 — the watch feed (10 rounds to PASS)

Commit: `bark: the durable watch feed and counterparty observation`
(bark-stage1 `157171c51` at PASS). The hardest review of the arc so far:
the scan protocol was redesigned twice under adversarial pressure and
came out radically simpler and provably stronger.

Batteries at PASS: workspace units 611/611 (incl. 9 scripted-chain watch
protocol tests + sqlite registry/ledger/floor/update-id tests),
bark-channels 23/23 (incl. the Theirs LDK pin), bark-sdk channels 19/19,
server channel 22/22.

## What the commit does (final shape)

- LDK's `Filter` registrations land durably the moment they are made
  (fail-closed like monitor persistence; reload re-registration is
  idempotent via partial unique indexes).
- The wallet's watch sync (a daemon chain-safety duty, module
  `bark/src/channels/watch.rs`, generic over a `WatchChainView` seam)
  scans for the registered set and feeds RAW matches through the c1
  barrier — whole transactions at real positions, height-ascending, one
  apply per computed set.
- Counterparty observation: `obtain_commitment` probes the funding
  outpoint for a confirmed foreign spend, feeds it (observation gated
  with selection), and returns `Theirs`; the `Ours`-only assumption is
  gone. Descriptor LIVENESS is derived per read (parent canonically
  confirmed?) — no stored orphan flag to race.
- The server keeps its full-block feed (Filter slot None).

## The load-bearing protocol choices (each a round's survivor)

1. **One chain snapshot per sync** — every canonical-block lookup
   memoized; a mid-sync reorg cannot split the world view into the
   same-tx-different-block shape LDK asserts on (r7).
2. **Nothing shallow is ever suppressed** — LDK's chain-sync monitor
   persistence is conditional, so no observable proves a shallow
   confirmation is in every monitor's durable blob; everything within
   the finality horizon is re-fed idempotently each sync (r5/r6, killing
   a whole class of crash-divergence suppressions).
3. **Deep memory = the delivery ledger**, written only after the
   delivery's barrier AND after `persist_all_monitors` snapshots every
   monitor blob (id-stable read: update-id sampled on both sides of the
   encode; the store refuses update-id regression) — "ledgered" implies
   "every monitor durably knows it" (r7/r8/r9).
4. **Per-watch floors** (parent's canonical confirmation, height+hash,
   revalidated each sync, reverting to unresolved on reorg) cover deep
   history exactly once; unresolved floors drag the window down, which
   is what makes late / crash-lost-and-re-registered / reorg-moved
   watches safe (r2/r3/r7).
5. **The scanned-to watermark** (height+hash, atomic with the floors in
   one transaction) bounds the deep side after offline gaps longer than
   the horizon; non-canonical watermark backs off by the horizon (r8).
6. **The same-tx-different-block guard lives INSIDE the driver's apply**
   (single serialized owner — no caller-side check can be race-free),
   records mapped once per op (r8/r9).
7. **Every best-block advance is preceded by a stale-record withdrawal**
   (wallet-open catch-up, the exit tick's advance, the bridge feed, the
   sync's own reorg pass): an advance must never mature state off a
   reorged-away confirmation (r8/r9).
8. **Stale records judged per monitor** — the aggregate
   `get_relevant_txids` dedupes a shared transaction, hiding one
   monitor's stale copy (r4).

## Settled disputes

- **Independent-transaction height ordering across feeds**: codex raised
  it in r3 and r7, conceded permanently in r8 after being put to the
  burden — "I found no lightning-0.2.4 path that couples independent
  confirmations to their delivery order", citing the pinned
  historical-height release test. Dependent transactions stay
  topologically ordered structurally (a spend's watch can only exist
  after its parent was fed).

## Accepted residuals (on the record)

- Registry/ledger rows are never pruned (a handful per channel
  lifetime); pruning is operational-surface work.
- bitcoind pays a bounded finality-horizon block scan per sync while
  watches exist; esplora runs per-outpoint lookups at bounded
  concurrency.
- LDK-unrecorded shallow deliveries re-feed until deep (bounded,
  idempotent churn by design).
- One-tick delays: a registration slipping past the final quiescence
  check waits for the next sync (its unresolved floor covers it).
- The full-wallet Theirs e2e (a server commitment actually reaching the
  chain) defers to the server-relay seam the c6 HTLC force-close matrix
  needs anyway; the Theirs shape is pinned at the LDK-harness level
  (zero-local-output variant; the with-payments variant wedges the
  in-process harness inside `best_block_updated` — parked as a
  harness/LDK interaction, noted in the commit).

## Round trail (compressed)

R1 (5 P1/2 P2): Filter-failure ack/cursor hole; Boot-route mistaken for
a barrier; unordered HashMap delivery + reorg-tip mishandling; late
historical watches unscanned; orphan-flag write races (→ derivation);
coverage; esplora batching. R2 (3 P1/2 P2): rescan order; union skip
stranding a component; unvalidated Theirs spender; floors defeating the
cursor; ledger-vs-bail order. R3 (3 P1/1 P2): canonical-record skip
strand; floors stale after parent moved lower; cross-producer ordering
(disputed); per-iteration rescan cost. R4 (2 P1/1 P2): ledger vs
conditional monitor persistence; per-monitor dedupe hazard; prefix
ledger writes. R5 (2 P1/1 P2): recorded-union suppression; fan-out
rollback vs snapshot skip; quadratic unconfirms. R6 (3 P1/2 P2):
force-deliver outside the window; budget livelock; mid-sync reorg memo
gap; duplicate-delivery persist cost; partial memoization. R7 (5 P1/
2 P2): offline gap; deep-ledger vs never-persisted monitor; per-height
memo hole; unfenced other feeders; ordering re-raised (rejected);
pruning; two barriers. R8 (6 P1): snapshot overwrite; TOCTOU guard;
unguarded feeders; floor/watermark crash gap; height-only watermark;
unfenced advances — ordering conceded. R9 (2 P1/1 P2): snapshot
id-pairing; bridge-feed advance; guard cost. R10: PASS.
