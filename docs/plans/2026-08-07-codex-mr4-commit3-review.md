# Codex review record — MR-4 commit 3: "bark: the channel open"

Commit under review: bark `32dc8cbe` (jj change `tuwkvvsx`) on
`ark8-channels-stage1-client`, child of commit 2 (`fe8b1fa6`).
Reviewer: codex, `model_reasoning_effort=max`, read-only sandbox, three
rounds. Design contract: `2026-08-06-mr4-client-design.md` §4–§7.

Verdict history: R1 **FAIL** (6 findings, all adopted) → R2 **FAIL**
(the 6 fixes verified, 3 new findings in the fixes, all adopted) → R3
**PASS**.

The SDK e2e suite (real captaind + bitcoind + postgres) was green at
every amendment: the open end to end (~9 s), a three-boundary restart
matrix (Funding / Feeding / post-Done) re-driving to done through
wallet reloads, and the client-side refusal surface. bark-wallet units
122/0; full workspace check clean; bark-cli (which builds bark-wallet
WITHOUT the channels feature) compiles.

## Round 1 — six findings, all adopted

1. **A crash after `Done` could strand a ready channel** (P1). The
   checkpoint is removed at `Done` while the manager blob persists
   asynchronously — a crash in between leaves a `Ready`/fed record and
   a reloaded manager without the virtual confirmation, with no action
   left to re-drive. Fix: the feed became a shared
   `Wallet::feed_channel_confirmation` (level-triggered, idempotent,
   same-position re-feeds safe) and a new
   `Wallet::reconcile_channel_feeds` runs on the daemon's sync cadence,
   re-feeding any registered/ready record whose manager view lacks
   readiness. A registered record the manager does not know at all logs
   an error (the exit machinery's territory).
2. **A lost `FundingGenerationReady` parked a pre-funding remnant
   forever** (P1). The event is deliberately volatile (ack-before-
   durable — pre-funding channels are remnants); but after a restart
   nothing closed the remnant. Fix: the node tracks which opens ran
   `create_channel` in THIS process; a pre-funding channel without its
   event and without that marker is closed and the next drive starts
   fresh. Validation failures re-stash the event payload so a
   deterministic refusal stays re-derivable.
3. **Funding re-entry lacked adopt-with-validation** (P1) — including a
   same-process bug where the establish path consumed the in-memory
   `ChannelPending` stash and the funding path then closed a perfectly
   good funded channel. Fix: `ChannelPending`'s payload (the funding
   redeem script exists ONLY in that event; LDK 0.2.4 exposes no public
   accessor to recover it from the monitor) is now captured durably
   BEFORE the event is acknowledged — a `bark_channel_establishment`
   table in m0044, `ReplayEvent` on write failure — and both re-entry
   paths adopt from the capture straight to the cosign.
4. **A cosign refusal left the funded LDK remnant live** (P2). Fix:
   both terminal `on_rejection` arms close the remnant and clear the
   establishment capture before `Failed`.
5. **Movement terminalization was best-effort** (P2). Fix: finish
   failures propagate; the checkpoint is retained and the next drive
   retries the idempotent block.
6. **`bark/schema.sql` lacked m0044** (P1, CI artifact). Regenerated.

## Round 2 — fixes verified; three findings in the fixes, all adopted

1. **The adopt validation was circular** (High): the bridge was built
   from the captured redeem script and the capture's outpoint compared
   against that same bridge — self-consistent for ANY capture a prior
   attempt of the same action legitimately wrote, including one whose
   channel is closed. Fix: three-way binding — (a) the plan (the
   capture-derived bridge pins the input and amount), (b) the LIVE
   manager state (a channel for this open must exist with the same
   permanent id and funding outpoint as the capture), (c) the
   checkpointed negotiated funding script when the phase pinned one.
   Any mismatch retires the capture and whatever channel exists, and
   re-establishes from the plan. (The capture table's primary key on
   `user_channel_id` means a newer attempt's `ChannelPending` always
   overwrites the older capture, so retirement can never hit a valid
   funded channel — R3 confirmed.)
2. **A capture with no redeem script retried forever** (Medium): now
   stale (same retire path) — a manually funded channel always carries
   its redeem script; its absence is not adoptable.
3. **`FundingGenerationReady` was not bound to its attempt** (Medium):
   a late event from a closed earlier attempt could fund a FRESH
   channel through LDK's unchecked manual-funding API, binding it to a
   stale script. Fix: the stash keeps the event's temporary channel id;
   `created_this_run` maps each open to the temp id its latest
   `create_channel` returned; consumption requires the match
   (superseded events are discarded); and the manual-funding call
   refuses a pre-funding channel whose id differs from the created one.
   `ChannelClosed` also best-effort clears the closed channel's
   capture, backstopped by the live binding.

## Round 3 — PASS

All three fixes verified: the three-way binding is sound, stale
retirement closes only mismatched attempts, ordered LDK events make
teardown/re-establishment converge, and temp-id matching rejects
superseded and post-restart events with no remaining in-scope safety
bypass.

## Notes

- e2e debugging (before R1) had already surfaced and fixed three real
  transport/recovery bugs the unit tests could not see: LDK's async
  background processor does not disconnect peers on termination (only
  its CPU-starvation branch does) — the driver now tears transports
  down on exit; an in-flight dial could register a connection after
  teardown, leaving an orphaned transport answering pings forever and
  pinning the node id at the server through LDK's duplicate-connection
  guard — the dial is stop-aware with active disconnect on abandon,
  plus a post-drain daemon sweep; and commit 1's reload-with-monitors
  refusal became reachable — replaced by `advance_chain_view` (a
  best-block advance, complete while every funding is virtual; the
  exit machinery brings the on-chain reconcile) plus the
  level-triggered re-feed. `Wallet::stop_daemon_wait` was added for
  drain semantics.
- The e2e restart matrix restarts at the observable park boundaries —
  the cosign→register chain is deliberately one uninterruptible drive,
  so `Cosigning`/`Registering` are never park states.
- e2e 6 (the fail-back HTLC operability probe) stays covered by the
  server suite's reference client until the payments milestone gives
  the production wallet a send path; stated in the commit message.
- The reviewer could not run cargo (read-only sandbox); all
  checks/tests above were run outside the review at each amendment.
