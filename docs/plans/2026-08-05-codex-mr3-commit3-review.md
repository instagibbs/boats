# Codex review record — MR-3 commit 3 (open-by-upgrade admission + the open state machine)

**Subject**: the third commit of the captaind-channels MR: "captaind: admit
the open-by-upgrade — admission and the open state machine", bookmark
`ark8-channels-stage1-captaind` (parents = the commit-1 node `d7bab25c` and
the commit-2 scaffold, itself amended across this arc: `07997102` →
`89c252e7`). Hashes across the arc: `108c9d46` (round 1) → `03802091`
(round 2) → `054c3186` (round 3) → `a1afc0c3` (round 4) →
**`bbf2a492` (round 5 = PASS)** — one commit, amended in place; the two
V55 CHECK lines the rework added (`'reaping'`) folded into the scaffold
commit where the migration lives.

**Scope**: the §6 admission checks on the arkoor-cosign channel branch
(dispatch, self-spend binding, checkpoint carve-out, reconstructed-output
doctrine, amount/delta/headroom/runway, bridge reconstruction + funding-
outpoint equality, bridge partial); the §6b durable open state machine
(opening → awaiting_upgrade → cosigned, per-peer pending bound, reaping);
the gate's blocking half (withheld funding confirmations + fail-closed
`ChannelReady`); two e2e suites (the admission matrix over the public
endpoint; a real Lightning establishment against the peer listener).

**Process**: five rounds converging 7 → 5 → 3 → 1 → 0 open findings
(REWORK, REWORK, REWORK, PASS-WITH-CHANGES, PASS), every fix folded into
the one commit and the spec (design note §5/§6a/§6b/§13) amended in step.
Each round's Critical/Important findings were real — the arc's pattern
was reviews catching the previous fix's own tail (the filter fix
introduced the untrusted-txid hole; the replay carve-out missed the cap
interaction; the qualified delete was itself the fix to the collision
fix's cleanup).

## Round 1 — REWORK (1 Critical / 4 Important / 2 Minor)

1. **C — spent-marking could outlive the channel row** (admission.rs): the
   Ark spent-marking committed before the guarded channel transition; a
   concurrent `ChannelClosed` or reaper deletion could remove the awaiting
   row in between, leaving the endpoint erroring with the input already
   OOR-spent and no signatures delivered — and retries dead because the
   row was gone.
2. **I — the final spendability "re-check" used the stale snapshot** read
   at request start: a ban applied during admission was invisible, and the
   spent-transition SQL does not check `banned_until_height`.
3. **I — `ChannelPending` promotion not replay-idempotent**: LDK re-delivers
   a handled event whose removal did not persist (the documented
   crash-replay window); the exact replay returned `false` and the handler
   force-closed a good channel.
4. **I — the reaper deleted rows before force-closing LDK** and ignored the
   close result: a crash or API error left an accepted LDK channel without
   its durable record.
5. **I — a real bridge confirmation could walk a cosigned-but-unregistered
   channel to operational**: the returned partials let the client complete
   and broadcast transfer + bridge for real; the unfiltered block feed then
   confirmed the funding, and `ChannelReady` fell through the no-action
   arm. The gate's blocking half cannot wait for the gate commit.
6. **M — the bridge signed against a request-metadata key**
   (`dest_user_pubkey`) rather than the reconstructed channel VTXO's.
7. **M — test gaps**: the "retry" used fresh client nonces (not
   byte-identical, so it never proved fresh server nonces); the watch
   assertion checked only non-emptiness; no concurrency/deletion-race, no
   self-binding, no malformed-funding-binding, no different-backing, no
   pre-registration-confirmation case.

**Verified clean in round 1**: admission arithmetic and the strict runway
guard; canonical funding-script reconstruction; bridge sighash
amount/script/tweak; fresh server-nonce generation; response-part
placement; pins/backing/watch atomicity and the unarmed default; SQL
guards, parameterization, hex conventions; open-request policy checks; the
genuine LDK establishment; the structured startup log.

## The rework (all seven + one proactive)

- **F1+F2 — one transaction**: everything that persists commits in a single
  `db.write` — the channel row locked (`SELECT … FOR UPDATE`, new
  `get_channel_state_locked`), the fresh/idempotent/refuse state decision
  repeated authoritatively in-tx, the input rows re-read fresh AND locked
  (new `get_user_vtxos_by_id_locked`), `check_spendable_for_oor` on that
  fresh view, the spent-marking, the `cosigned` transition and the
  (unarmed) watch row. A refusal rolls the whole exchange back untouched;
  `server_cosign` and `server_cosign_bridge` run only after the commit.
  The row lock is what serializes admission against a concurrent close or
  reap; the pre-tx state screen remains only for a clean early error.
- **F3 — promotion trichotomy**: `ChannelPromotion
  {Promoted, AlreadyApplied, Contradiction}`; AlreadyApplied = the exact
  ingredient match under the final id in any post-`opening` state — a
  no-op, not a contradiction. **Proactive same-class fix**:
  `insert_channel_opening` tolerates the exact `OpenChannelRequest` replay
  (identical `opening` row → proceed to re-accept, which on a true replay
  is back in awaiting-accept after the manager rollback); mismatched
  collisions still bail.
- **F4 — mark-then-close reaping**: a new `'reaping'` state (V55 CHECKs,
  folded into the scaffold commit); the reaper durably marks, force-closes,
  and the row is deleted only when the close lands (`ChannelClosed`), with
  `Err(ChannelUnavailable)` = orphan (deleted directly) and any other error
  retried every block (marked rows are re-returned on every call).
  `reaping` rows still count against the peer's pending bound.
- **F5 — the gate's blocking half**: a `withheld_funding` txid set on the
  subsystem — seeded from the channel rows at boot BEFORE any replay
  reaches the node, inserted at promotion, re-asserted at cosign, removed
  by nothing until the gate's release half exists —
  `confirm_block_transactions` filters withheld txids out of the feed (LDK
  accepts a positional subset), and `Event::ChannelReady` fails closed.
- **F6**: the bridge signs against `channel_vtxo.policy().user_pubkey()`;
  the request-level self-spend equality stays as a screen only.
- **F7 — tests**: byte-identical duplicate (same bytes, same client nonce)
  asserting the server's bridge nonce DIFFERS across responses; exact
  watch `response_txids == [transfer txid]`; self-binding refusal (policy
  swapped to a rogue key post-build); different-backing against a cosigned
  channel; wrong recorded funding outpoint; the reap race (row flipped to
  `reaping` under the request → refusal + an explicit
  `oor_spent_txid IS NULL` assert → restored → the same input cosigns to
  completion); the establishment e2e gains the peer-claimed-confirmation
  case — the fully SIGNED bridge confirmed on a fabricated client-side
  chain, the client sends an early `channel_ready`, and neither readiness
  nor the record moves. postgres coverage for the promotion trichotomy,
  the insert carve-out, and the reaping lifecycle.
- **Spec amendments in step** (the design note is the spec): §5/§6b now
  name the gate's blocking half and require it in the commit that accepts
  opens; §6a describes the one-transaction commit; §6b gains `reaping` and
  the crash-replay idempotence rule; §13's commit-3 entry updated.

**Implementation gotcha worth keeping**: LDK panics "funding_transaction_
generated with bogus transaction" if a FUNDER's confirmed funding tx has an
empty witness (anti-malleability) — the fabricated-confirmation test must
attach the aggregated keyspend signature to the bridge before confirming it
client-side.

**Battery on the reworked tree**: `cargo check --all --tests` green; units
523/523 (nextest); clippy ark-lib clean, bark-server at the 236-warning
baseline; channels e2e 8/8; the 40-test slice at 39 passed — the single
failure is the machine's lightningd env gap, now precisely identified
(`lightningd: gmp version mismatch: compiled 6.2.1, now 6.3.0`).

## Round 2 — REWORK (1 Critical / 2 Important / 2 Minor); F1–F4, F6 closed

The confirmation review closed five of round 1's seven findings on evidence
and found the rework's own flaws — the Critical being a hole in MY F5 fix:

1. **C — untrusted funding txids poisoned the global confirmation filter**:
   the rework inserted the peer-declared `funding_txo.txid` from
   `ChannelPending` into the withheld set. LDK takes that outpoint straight
   from the peer's `funding_created` with no validation — a peer could name
   ANY txid (another channel's commitment transaction!) and blind the
   node's manager and every monitor to it, persistently until restart, with
   unbounded set growth for free.
2. **I — the exact `OpenChannelRequest` replay failed at the pending cap**:
   the count check ran before the replay-tolerant insert, and the replayed
   row itself counts — at four pending rows a genuine crash replay was
   refused (and force-closed) before classification.
3. **I — a final-channel-id collision replayed forever**: rekeying onto an
   occupied `channel_id` raises the postgres UNIQUE violation before the
   promotion's own Contradiction logic, and the handler maps every DB error
   to `ReplayEvent` — a poisoned queue, reachable cross-peer because LDK's
   collision check is per-peer and an `opening` row has no monitor.
4. **M — the "reap race" test never reached the in-transaction decision**
   (it committed `reaping` before the RPC, so the early screen refused).
5. **M — a concurrent-close winner surfaced as INTERNAL** (`None.context`)
   instead of the intended client refusal.

**Round-2 fixes:**

- **Authenticated-only filter** (the Critical): the ONE insertion point is
  admission — the bridge txid the server's own reconstruction produced,
  inserted after the commit and before the partial leaves; boot seeds from
  `cosigned` rows only; `ChannelPending` inserts nothing (comment pins the
  reason). A pre-cosign decoy confirmation at the peer-declared outpoint is
  answered by the fail-closed `ChannelReady` arm — the channel dies with no
  Ark state, which is the correct posture. Nothing removes entries in this
  commit (the release half does); the set is bounded by real cosigned
  channels, not by free peer messages. Spec §6b amended to name the
  untrusted-txid rule.
- **Classify-first opens**: `admit_channel_opening` (one transaction)
  classifies exact-replay/fresh/mismatch BEFORE the pending bound, which
  now applies to fresh opens only — also closing a latent TOCTOU where two
  concurrent opens could both squeeze past the count-then-insert. The
  accept-failure row cleanup now runs only for freshly inserted rows (an
  exact replay's row may record a genuinely accepted channel).
- **UNIQUE violation → Contradiction**: the occupied-id rekey maps SQLSTATE
  23505 to `ChannelPromotion::Contradiction` (force-close), never
  `ReplayEvent`; the violation aborts the surrounding transaction, so the
  method is documented as its transaction's only mutation (it is).
- **The reap race is now genuinely concurrent**: a second connection holds
  the channel row's `FOR UPDATE` lock, the RPC passes its early screen and
  blocks on the authoritative locked lookup, the reaper's mark commits
  under it, and admission re-decides on the changed row — refusal, full
  rollback (`oor_spent_txid IS NULL`), then the same input cosigns after
  restore. Degrades to the early-screen refusal (never false-fails) if the
  RPC hasn't reached the lock when it releases.
- **Vanished-row refusal**: the locked lookup's `None` maps to the same
  client-shaped refusal as every other authoritative state decision.
- New postgres coverage: replay-at-cap (fresh refused, replay passes),
  cross-peer id-squat promotion → Contradiction.

## Round 3 — REWORK (1 Important / 2 Minor); R2-1/2/5 closed

The round-2 fixes held (authenticated-only filter, classify-first opens,
badarg on the vanished row), but two of them had their own tails:

1. **I — the squatter's close deleted the victim's row**: after the
   occupied-final-id contradiction force-closes peer B, B's
   `ChannelClosed` carries the FINAL id — which names peer A's legitimate
   pre-cosign row, and the delete matched on id alone.
2. **M — the "one transaction closes the TOCTOU" claim was false**: READ
   COMMITTED does not serialize COUNT + INSERT; two concurrent admissions
   for one peer could both count three and both insert. (Production is
   incidentally serialized by the single event drain — the claim, not the
   deployment, was wrong.)
3. **M — the concurrent reap-race test could false-pass** via the early
   screen: the sleep prevented false failures, not false passes.

**Round-3 fixes:**

- `delete_channel_pre_cosign` is qualified by `(channel_id, counterparty)`
  — a close only ever cleans the closed peer's OWN row; a `ChannelClosed`
  without a counterparty is left to the reaper (which deletes by the row's
  own recorded counterparty). Regression: the squat scenario now asserts
  the victim's row survives the squatter's close and the squatter's own
  row goes by its temporary id.
- `admit_channel_opening` takes the house `pg_advisory_xact_lock`
  (`ChannelOpenAdmission`, transaction-scoped) before the count — the
  bound is now DB-serialized, not event-loop-incidental. Regression: 8
  concurrent fresh admissions against a bound of 4 → exactly 4 Inserted /
  4 OverLimit / 4 rows.
- The reap-race e2e releases the held lock only after `pg_stat_activity`
  OBSERVES the admission blocked on it (`wait_event_type = 'Lock'` on the
  `FOR UPDATE` lookup), with a 15s deadline — the concurrent path is now
  proven, not probable.

## Round 4 — PASS-WITH-CHANGES (1 Minor); R3-1/2 closed

The round-3 fixes held (counterparty-qualified deletes — all three callers
verified; the advisory lock precedes classify/count/insert with no
reciprocal lock-order cycle). One precision tail on the reap-race test:

1. **M — the lock-wait predicate was not correlated to THIS test's
   blocker**: `pg_stat_activity` is cluster-wide, so a parallel test
   blocked on the same query text could satisfy it and release early —
   the false-pass hazard R3-3 targeted was narrowed, not closed.

**Round-4 fix**: the lock holder reports its `pg_backend_pid()` through
the oneshot, and the release predicate now requires
`datname = current_database()` AND `holder_pid = ANY(pg_blocking_pids(pid))`
— the observed waiter is provably blocked by THIS test's lock.

## Round 5 — PASS (scoped; no findings)

R4-1 and R3-3 CLOSED on evidence: the waiter is tested against this
test's holder via `pg_blocking_pids(waiter_pid)`; prepared execution
reports the original query text and the row-lock wait reports `Lock` with
the holder pid; the pid is sent only after `FOR UPDATE` acquires; daemon,
holder, and polling pools share one database. The inter-round diff was
solely the test change (14 additions / 9 deletions).

**Final battery at `bbf2a492`**: `cargo check --all --tests` green; units
523/523; clippy ark-lib clean, bark-server at the 236 baseline; channels
e2e 8/8 (admission matrix incl. the observed-blocked concurrent reap
race, real establishment incl. the peer-claimed confirmation); postgres
suite green (promotion trichotomy, replay-at-cap, 8-way concurrent
admission bound, squat-cleanup, reaping lifecycle); the timing-sensitive
admission e2e 3/3 back-to-back; the 40-test slice at 39 with the single
failure the machine's lightningd env gap (gmp version mismatch,
pre-existing).

## Deferred, tracked for the gate commit

The release half (level-triggered reconciler, `confirmation_fed` barrier),
SCID allocator, watch arm/detect/respond/resolve, expiry arm,
config-decrease guard; the server-side real-confirmation-through-the-feed
e2e (needs the input's full exit chain confirmable on-chain plus the
registration seam — the peer-claimed case covers the client-side angle
now); the daemon-restart reload e2e; reserve sizing from real package
weights. OP-27 (client retry nonce discard) is part 4.
