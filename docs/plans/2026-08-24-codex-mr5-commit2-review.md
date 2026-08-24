# Codex review: MR-5 commit 2 — "captaind: admit the sanctioned split" (4 rounds to PASS)

Commit `b04f8020a` on `ark8-channels-stage1-close` (bark-stage1). Battery
at PASS: 9/9 (3 downgrade e2e + 4 close e2e + stale-manager e2e + postgres
downgrade-group suite incl. the production-path registration test),
prechecks clean.

## Round 1: FAIL — 2 P1, 2 P2

1. **P1 registration can succeed after the final confirmation became
   observable.** The pre-lock bitcoind screen plus the scan-epoch check
   left a window: final confirms, listener lags, epoch unchanged →
   spendable leaves commit; the later fact only no-ops on an armed
   watch. The stand-alone tombstone transaction also raced concurrent
   registrations. → the race decision moved UNDER the subsystem chain
   lock every channel-bearing registration serializes on
   (`screen_group_race`): re-screen final/response by RPC, write the
   monotonic `fallback_won` tombstone in its own transaction BEFORE any
   tree update, return the authenticated permanent refusal. A
   confirmation landing after this screen and before commit degrades to
   the ordinary armed-watch race, which the response answers with the
   whole `exit_delta` window in hand (accepted residual, same posture
   as the upgrade's release).
2. **P1 cursor-lagged ancestor foreclosure registers an unbacked
   split.** Cosign walked the exit chain but registration checked only
   the final tx; an ancestor foreign-spent while the listener lagged
   would register leaves whose funding chain can never exist. → the
   full foreclosure walk runs in the registration preconditions too
   (pre-lock, RPC), and the in-lock hazard is closed by finding R2-1's
   expiry-margin bound (below).
3. **P2 same-block final+response misclassified as fallback-won.** The
   split response consuming the contested output IS the split winning,
   even in the final's own block. → the screen admits a confirmed
   response at height ≤ the final's; live judgment orders facts by
   `(block_height, response-first-on-ties)` in `apply_watch_facts`.
4. **P2 response timing not guaranteed ahead of `exit_delta`.** Generic
   watchman grace could eat the whole window. → registered channel
   splits bypass grace (Progress immediately) and the deadline is
   `confirmed_at + exit_delta − 1` (last safe height, strict).

## Round 2: FAIL — 2 P1, 2 P2

1. **P1 the ancestor walk is pre-lock; a sweep confirming while
   registration holds the chain lock still lands unbacked leaves** (the
   lock itself blocks the epoch bump — the scan cannot warn). → the
   PURE-STATE bound: an ancestor expiry sweep is only CONFIRMABLE past
   the scope's expiry, so cosign AND registration refuse when the
   observed tip is within `DOWNGRADE_EXPIRY_MARGIN` (6 blocks) of the
   backing's expiry. Below the bound a confirmed sweep cannot exist
   regardless of listener lag; a downgrade that close to expiry has no
   business settling (the client's deadline machinery falls back far
   earlier).
2. **P1 successful registration removed stale-manager resurrection
   protection** (registration cleared the close outcome; the defunct
   predicate keyed on outcomes). → `closed_channels()` includes any row
   with a downgrade GROUP in any state — a group means the close
   reached cosign and the channel must never operate again.
3. **P2 response-first ordering ignored block height** (a response at
   H+1 admitted behind a final at H when facts arrived batched). →
   height-aware dominance everywhere: the screen compares heights; the
   watch orders batches by `(block_height, response-first-on-ties)`.
4. **P2 arming not persisted over a resolved-`responded` watch** (reopen
   preserved the null `armed_at`; the re-confirming final judged as an
   unarmed fallback on a registered group). → `arm_channel_parent_watch`
   is independent of resolution; `register_downgrade_groups` arms in
   both admissible branches (unresolved and resolved-responded).

## Round 3: FAIL — 2 P1, 1 P3

1. **P1 the exact commit did not compile** — the server's group-digest
   delegation referenced a lib helper that only existed in the
   next-commit worktree. → `ark::channel::downgrade_group_digest` ships
   in this commit: the single-sourced group identity (sha256 over
   sorted member id bytes) both sides must derive identically for the
   authenticated refusal to bind anything.
2. **P1 the expiry bound was scheduler-dependent** — the tip was
   snapshotted once before the registration transaction, and nothing
   bounded how long the transaction waited on row locks; a stall past
   the margin could commit leaves whose ancestors had entered the
   sweepable window. → codex's sanctioned alternative:
   `recheck_group_expiry_cutoffs` re-fetches the tip by RPC — never the
   cached cursor, whose advance the held chain lock itself blocks — and
   re-applies the cutoff as the FINAL statement of the registration
   transaction, after every row lock. A stalled registration re-fails
   there and the whole upload rolls back; the residual is one COMMIT
   round-trip (bounded work, not a scheduler). Codex's preferred
   durable sweep-intent marker has no broadcast site to instrument in
   this tree (the VtxoSweeper is a stub); it becomes the natural
   companion of the future sweeper.
3. **P3 the claimed registration-arm test bypassed registration** (it
   armed directly). → the postgres test drives the production
   `register_downgrade_groups` inside a locked transaction over a watch
   resolved `responded`: registers, arms with the resolution preserved,
   accepts the idempotent re-upload, keeps the armed marker across a
   reorg-reopen, and refuses the tombstone on the registered group.

## Round 4: PASS — no findings

Digest single-sourcing, the commit-time RPC recheck placement (after all
transactional work, rollback on failure), and the production-path test
verified in-place; no regressions.

## Key design decisions (for the arc record)

- **Group-atomic registration**: a durable downgrade group is written at
  cosign (members = all leaves of both branches); an incomplete upload
  registers nothing; the group registers exactly once, idempotent on
  re-upload.
- **Monotonic `fallback_won` tombstone**, written under the chain lock
  in its own transaction before any tree update; reorgs never reopen
  recognition; the refusal is authenticated and machine-readable
  (`FailedPrecondition` + `downgrade-result: FALLBACK_WON` +
  `downgrade-group-digest`), digest single-sourced in
  `ark::channel::downgrade_group_digest`.
- **Race decision**: response confirmed at height ≤ the final's height
  wins (same-block split wins); later responses are the spec's
  late-armed refusal; watch facts apply in
  `(block_height, response-first-on-ties)` order.
- **`DOWNGRADE_EXPIRY_MARGIN` = 6**: pure-state expiry cutoff at cosign
  and registration (sweeps are only confirmable past expiry), closing
  the window the chain lock blinds the scan epoch to; re-checked
  against a fresh RPC tip as the last statement of the registration
  transaction so no scheduler stall can carry a decision past the
  margin.
- **Defunct predicate widened**: any group-bearing row (registered or
  fallen) is logically force-closed before peer acceptance — a settled
  channel can never be resurrected by a stale manager snapshot.
- **Watchman policy for registered splits**: no grace, progress
  immediately, deadline `confirmed_at + exit_delta − 1`.
