# Codex review record — MR-4 commit 1: "bark: the embedded channel node"

Commit under review: bark `c53eaedf` (jj change `urlmzyyp`) on
`ark8-channels-stage1-client`, cut from the MR-3 captaind tip
(`d18d33a1`; upstream rebase deferred). Reviewer: codex,
`model_reasoning_effort=max`, read-only sandbox, three rounds.
Design contract: `2026-08-06-mr4-client-design.md` §3/§5 (commit 1 of
§11's plan).

Verdict history: R1 **FAIL** (7 findings, all adopted) → R2 **FAIL**
(the 7 fixes verified, 2 new findings, both adopted) → R3 **PASS**.

## Round 1 — seven findings, all adopted

1. **Queue-write failure was log-and-continue** (P1). The broadcaster's
   doc claimed queued transactions are re-derivable from monitors, but
   LDK treats a handed transaction as handled — a lost write could be
   the only durable copy of a recovery transaction at the wrong moment.
   Fix: `QueueBroadcaster` carries the node's stop token and cancels it
   on a failed write — same fail-closed rule as a failed monitor
   persist.
2. **Reload never re-established the chain view** (P1). The
   `bark-channels` reload contract requires the caller to feed the
   chain back to current before events run; a monitor reloaded behind
   the chain misses a mined (possibly revoked) commitment. The real
   catch-up belongs to the exit machinery (commit 4). Fix for this
   commit: `load_or_build` refuses a reload with live monitors
   outright — no open flow exists yet, so live monitors can only mean
   state from a future version; commit 4 replaces the refusal with the
   actual catch-up. (Manager-with-zero-monitors reloads stay allowed —
   nothing to watch.)
3. **A panicked driver task could be hot-restarted over a poisoned
   store** (P1). LDK panics on `UnrecoverableError`; the daemon's
   `supervised` wrapper restarts panicked tasks. Fix: the entry guard
   returns immediately (clean exit → supervision ends) when the node's
   stop token is already cancelled — `NodeMonitorPersist` cancels
   before returning `UnrecoverableError`, so the token is always
   cancelled by the time the panic unwinds.
4. **Unexpected `BumpTransaction` (HTLC-resolution) events were
   acknowledged** (P1). Impossible by policy in M4 (no HTLCs), but if
   one arrives it carries claim material this build cannot persist,
   and acking drops it forever. Fix: stop the node AND return
   `ReplayEvent` — the event stays unconsumed for a build that can
   handle it. (R2 verified the replay actually rides serialized
   monitor claim state and later regeneration, not `ReplayEvent`
   itself.)
5. **`channels` leaked into wasm builds** (P1). `bark-channels` is
   native-only (LDK background processing + tokio networking). Fix:
   `compile_error!` on `channels` + `target_arch = "wasm32"`.
6. **`bark/schema.sql` was stale** (P2) — the checked-in DDL artifact
   lacked the m0043 tables. Regenerated (`just dump-bark-sql-schema`).
7. **Monitor upsert left `archived_at` set** (P2). A later LDK persist
   of an archived monitor stayed hidden from reload. Fix: the upsert's
   `ON CONFLICT` clears `archived_at` — LDK actively writing overrides
   its earlier prunable opinion. Regression test added.

## Round 2 — fixes verified; two new findings, both adopted

1. **Driver `Err` did not fail closed.** `run_driver` logged a driver
   error and returned with the token live; a daemon stop/start could
   re-run the same in-memory node over a manager snapshot that never
   persisted. Fix: `run_driver` cancels the token on any driver `Err`.
2. **Non-monitor panics still escaped the restart guard.** An LDK
   invariant/poisoned-lock panic elsewhere in the driver unwinds past
   the guard with the token live, and `supervised` reruns the node.
   Fix: the daemon runs the driver in its own tokio task and awaits
   the `JoinHandle` — any panic is captured as a `JoinError`, the
   token is cancelled, and the wrapper returns cleanly (ending
   supervision). R3 confirmed shutdown still drains (the propagate
   task cancels the token; the driver completes its final persistence
   before the handle resolves) and that repeated cancellation is
   harmless.

## Round 3 — PASS

No new in-scope issues. Confirmed: token cancelled on every driver
error; shutdown reaches the spawned driver and drains; `JoinError`
(including cancellation) cancels the token; `Wallet::open` reload
panics stay fail-fast (node constructed before the daemon starts).

## Notes

- The reviewer could not run cargo in the read-only sandbox (target
  lock); all checks/tests were run outside the review: full workspace
  `cargo check --all --tests` clean, bark-wallet 119 unit tests green
  (11 channel-specific: reload identity/chain-view, both reload
  refusals, codec round-trips, store lifecycles incl. the un-archive
  regression).
- Round 1's fix for the broadcaster deliberately supersedes the
  design note's "failed queue write is safe to lose" rationale — the
  note's §3 broadcaster paragraph stands corrected by the fail-closed
  rule (note amendment folded into this record rather than the note).
