# Codex review: MR-5 commit 3 — "bark: the cooperative close" (7 rounds to PASS)

Commit `5edf934cd` on `ark8-channels-stage1-close` (bark-stage1). Battery
at PASS: sdk close e2e (off-chain settlement, crash-at-PONR idempotent
completion, the fallback race riding the cooperative tail, a REAL
LDK-negotiated zero-output close) + the unilateral exit e2e green
standalone on the commit; units 138/138; a channels-less sqlite build
compiles.

## Round 1: FAIL — 2 P1, 3 P2, 1 P3

1. **P1 close initiation could wedge before `shutdown`.** The entry
   persisted the record flip and the action checkpoint separately, the
   send happened once in `close_channel`, recovery skipped `negotiating`
   records, and the Negotiating drive never re-sent. → `enter_channel_close`
   persists flip + checkpoint in ONE transaction; the send moved wholly
   into the Negotiating drive (idempotent, re-issued every drive; the
   direct send was REMOVED — it raced the daemon's concurrent drive of
   the same checkpoint); recovery re-derives `negotiating` records at
   their own phase.
2. **P1 the tail could select an unrelayable closing and park funds.**
   Selection committed even with no fee child, the tail never reselects,
   and the child was priced for itself alone (no parent-deficit make-up).
   → SELECTION IS COMMITMENT: the cooperative tail is selected only when
   the closing is already accepted (mempool/confirmed) or the pre-built
   claim child provably lifts the package; ours-but-unliftable HOLDS in
   `ChannelBridge` (re-evaluated every tick); nothing-of-ours resolves
   through OUR commitment (the monitor nudge). The child's rate converts
   the parent's exact fee deficit into an elevated child rate.
3. **P2 a sanctioned zero-user split could never settle client-side**
   (`ours.is_empty()` always refused). → permitted exactly when the
   retained outcome's user side is zero.
4. **P2 FALLBACK_WON cleanup was not crash-idempotent** (state flip,
   exit resume and movement failure were separate). → a replay that
   finds `fallback_only` re-runs the cleanup idempotently.
5. **P2 `close_resolution = 'exited'` was never recorded** (the store
   method had no caller). → `close_channel_exited` is ONE statement
   (`exiting → closed` + resolution COALESCE), called by both terminal
   reconcilers; the orphan setter deleted; the e2e asserts it.
6. **P3 e2e comments claimed more than asserted.** → assertions added
   (the fallback test pins the closing txid AND the cooperative-tail
   history; the unilateral test pins the resolution), narration trimmed.

## Round 2: FAIL — 2 P1, 1 P2, 1 P3

1. **P1 the child fee arithmetic ASSUMED a weight** (600 WU estimate vs
   ~487 real → ~81% of the deficit recovered; any child committed the
   tail unverified). → MEASURE, never assume: the built child's exact
   fee and weight decide acceptance; one corrected rebuild over the
   measured weight; still-short returns None and the selection holds.
2. **P1 a zero-output CHILD was possible** (change below dust collapses;
   LDK returns an output-less, consensus-invalid claim) and would have
   been treated as lifting. → refused before acceptance.
3. **P2 the FALLBACK_WON replay returned a different terminal variant**
   than the first execution (Done vs Failed), violating the action
   reentrancy contract. → both paths terminate `Advance::Failed`.
4. **P3 the Exited assertion had been lost to a silent patch-application
   failure; stale docs still described send-with-checkpoint.** → both
   fixed.

## Round 3: FAIL — 2 P1, 1 P2, 1 P3

1. **P1 the suspension-generation check was not atomic with the child's
   persistence** — a downgrade could cross its point of no return
   between the check and the BDK store, leaving an old-scope child
   persisted with wallet inputs locked in a transaction deliberately
   never rebroadcast. → `Exit::store_cpfp_if_current`: generation check
   AND store under ONE exit write lock (exit → onchain lock order — the
   suspension takes the same lock, so the schedules serialize); the
   callerless accessor deleted.
2. **P1 a sqlite build without the channels feature did not compile**
   (the trait method's impl was feature-gated while the trait
   requirement is unconditional). → unconditional method, cfg-branched
   body (`Ok(false)` without channels — exact: no channel can exist).
3. **P2 package acceptance rounded down** (`fee × weight / 1000` floor
   admitted one-satoshi-short packages that the committed tail then
   never reprices). → exact u128 cross-multiplication; `div_ceil` on
   the corrected derivations.
4. **P3 the correction loop rebuilt twice and discarded the second
   build unverified.** → explicit two-step: build, measure, correct
   ONCE, measure again, accept or None.

## Round 4: FAIL — 1 P1, 1 P3

1. **P1 the atomicity ended before ADOPTION** — the exit lock was
   released between the child's BDK store and `provide_cpfp_tx`; a
   suspension queued on the lock could untrack the package in between,
   leaving a stored-but-never-adopted child with wallet inputs locked in
   a transaction deliberately never rebroadcast (potentially the only
   fee UTXO — blocking the fallback's own CPFP). →
   `Exit::store_and_adopt_cpfp`: the generation check, the BDK store AND
   the package adoption/broadcast under ONE continuous exit write lock
   (`provide_cpfp_tx` now locks and delegates to the split-out
   `adopt_cpfp_locked`). A suspension queued behind the lock finds the
   child adopted and untracks it like any live package. Accepted
   residual: an adoption ERROR (package rejected) leaves a stored child
   whose package remains tracked, retried by the next tick.
2. **P3 the close documentation attached to a test seam.** → seams moved
   above the doc.

## Round 5: FAIL — 1 P1

1. **P1 the adoption-ERROR residual was not retry-safe** — a rejected
   package left the child BDK-stored, LOCKING its confirmed input while
   the next tick's rebuild cannot select unconfirmed funds: with a
   single reserve UTXO the CPFP stalls on its own predecessor. →
   ATTACH-OR-EVICT, still under the one lock: when the adoption did not
   attach the child (a definitive rejection, or an ambiguous transport
   failure), it is EVICTED (`bdk apply_evicted_txs` + persist) before
   the lock releases — inputs return to coin selection immediately; a
   child that was in fact accepted returns via the next mempool sync
   and then stands as the package the pricing sees. The verbatim-retry
   alternative was rejected: it would defeat rate escalation.

## Round 6: FAIL — 1 P3

1. **P3 the inserted hand-off enum split `Exit`'s rustdoc from the
   struct.** → reordered.

## Round 7: PASS — no findings

## Key design decisions (for the arc record)

- **Two-phase close capture** (broadcaster candidates, fail-closed;
  `ChannelClosed` selects the latest closing-shaped candidate and
  records the PRE-FEE outcome) with an ATOMIC close entry (record flip
  + action checkpoint in one transaction) and a drive-owned idempotent
  `shutdown` re-send — no direct send anywhere else.
- **Phased record states**; `registration_pending`/`downgraded` IS the
  derived broadcast suspension (`past_downgrade_ponr`), enforced at
  exit initiation, rehydration, the CPFP hand-off and the generic
  rebroadcast.
- **The CPFP hand-off is one continuous exit-write-locked operation**:
  suspension-generation check → BDK store → package adoption/broadcast
  → attach-or-evict. No schedule leaves a stored-but-never-adopted
  child; a rejected child's inputs return to coin selection at once.
- **Selection is commitment** for the cooperative tail: chosen only when
  the closing is already accepted or the MEASURED claim child provably
  lifts the package (exact u128 rate comparison; the child's rate
  covers the parent's fee deficit); ours-but-unliftable holds in
  `ChannelBridge` and re-evaluates; nothing-of-ours resolves through
  OUR commitment via the monitor nudge (an output-less, funder-torched
  artifact is refused at selection).
- **The cooperative tail is its own recovering terminal** with a
  STRUCTURAL zero-aware obligation floor (the closing's own
  shutdown-output count — zero owes nothing and terminalizes).
- **FALLBACK_WON** is the one non-retryable registration answer
  (authenticated, digest-verified against the retained split); its
  cleanup re-runs idempotently on every replay and terminates with the
  same variant (action reentrancy).
- **`close_resolution`** (`downgraded`/`exited`) is recorded atomically
  with each terminal transition.
- **BDK broadcast ownership**: exit CPFP children are tagged
  (tag-before-changeset) and excluded from the generic rebroadcast; a
  startup backfill classifies pre-tag children; the persister's three
  new methods are REQUIRED (no fail-open defaults).
- **Test seams** (`test-util`): a registration hold and a channel
  feerate pin — the zero-output e2e drives a REAL LDK negotiation into
  the sub-dust window.
- **Accepted residuals**: no e2e for the output-less-artifact →
  monitor-nudge → commitment composition, none for the zero-user close
  (unreachable without payments in stage 1); an adoption error leaves
  no durable child (evicted) and the next tick escalates.

## Follow-up fold (2026-08-25, from the surface-work design review)

Two shipped-code P1s found while reviewing the NEXT MR's design note
were folded into this commit (Greg's call — the branch had just been
pushed and he authorized the rewrite; commit rewritten `5edf934cd` →
`671e7d635`):

1. **The exit-cancel one-way door**: `initiate_channel_exit` admits the
   close-phase origins this commit introduced, but cancellation
   unconditionally restored `ready` — resurrecting a channel whose LDK
   side was irreversibly shut down (or whose fallback had won). The exit
   now records its ORIGIN phase (`exit_origin`, folded into migration
   47); only a `ready`-origin exit is cancelable; a recorded close
   outcome refuses cancellation outright.
2. **The close capture was not crash-atomic**: the broadcast queue and
   the close-candidate capture committed in two store transactions —
   a crash in between left `ChannelClosed` permanently unable to
   identify its transaction (fail-closed = wedged). One transactional
   `queue_broadcast_and_capture` replaces the pair.
