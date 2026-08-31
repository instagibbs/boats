# CodeRabbit round — MR-3 (captaind channels), 2026-08-06

**Subject**: CodeRabbit's review of the posted MR-3. 10 inline comments +
10 nitpicks, each verified against the tree before any fix; Greg triaged
the four design-shaped ones one-by-one. 13 adopted, 7 declined with
reasons. All fixes folded into their home commits; battery re-green.

Final series after the folds: opener `5130da08`/`a7c154b8`, protocol
`d4b05c03`/`30ee616b`/`f64ca92b`, captaind `dc62dc8b` (nrslwtrv) /
`c30fda36` (mynnnooz) / `f7ef02c7` (syyqlnln) / `d18d33a1` (qzxwpnzr).

## Adopted

1. **`channel_state_backing_vtxo_ux`** — partial unique index on
   `channel_state(backing_vtxo)` (V55 + schema regen). The round's best
   finding: admission's row-level checks compare a channel only against
   ITS OWN recorded backing, and the arkoor exact-replay rule (same
   spending txid) plausibly lets a byte-identical upgrade package cosign
   the same backing into a SECOND channel row — each with its own bridge
   cosign against a different funding outpoint, a double-promise of one
   output. The by-backing lookups (registration, release) also silently
   assume at-most-one row. The index closes the class structurally; the
   23505 aborts the admission transaction into a clean refusal (an RPC
   path — no ReplayEvent semantics to protect, unlike the promote case).
   Postgres regression: `channel_backing_vtxo_binds_one_channel` (the
   refused channel stays intact and cosigns a different backing).
2. **Watch unresolved index upgraded to UNIQUE** — the exact index
   already existed unkeyed; one input per channel makes the invariant
   real, now DB-enforced.
3. **Dual pending-open cap** (Greg's call after the upstream-analog
   review): `MAX_PENDING_OPENS_TOTAL = 64` beside the per-peer 4, both
   counted under the same advisory lock, exact-replay carve-out bypasses
   both. Upstream's posture is per-flow: boards are cost-bearing by
   construction (state only after confirmed funding), LN receive got the
   proof-or-token `ln_receive_anti_dos` mechanism — but an
   `OpenChannelRequest` rides the BOLT transport with no field for a
   proof, so acceptance bounding is the only available lever. Trade-off
   accepted knowingly: a filled cap blocks NEW opens only (≤ the 1-hour
   reap age); existing channels and all normal VTXO operations are
   untouched. The semantics: the cap bounds the TOTAL (16 maxed-out
   identities fill 64), not 64-peers-times-4. Highlighted for reviewers
   in the constant's doc. Postgres regression:
   `channel_pending_open_global_cap`.
4. **`ORDER BY vtxo_id` before `FOR UPDATE`** in `get_vtxos_by_id_inner`
   — deterministic lock order for overlapping locked sets; caller-order
   reconstruction unchanged.
5. **Accept-loop backoff** — 250ms sleep on `accept()` errors (fd
   exhaustion must not hot-spin).
6. **Per-walk `tx_status` cache** in `exit_chain_foreclosed` — in a
   chain-shaped ancestry every prevout is the previous item's txid, so
   the uncached walk queried everything twice.
7. **Terminal-loop observability** — a corrupt persisted channel id was
   silently skipped forever (level-triggered), and `force_close` errors
   vanished; both now `warn!`.
8. **Cache retain without per-entry `to_string()`** — the unresolved set
   is parsed `VtxoId`s now; corrupt rows error exactly as the arming
   loop below always did.
9. **State-filtered per-block load** — `sync_watch_cache` runs per block
   and loaded the whole `channel_state` table (terminal rows accumulate
   forever); now `channel_states_in(["cosigned","registered"])`. The
   startup disable-guard keeps its load-once-at-boot shape.
10. **Registration channel-context binding** — subsystem + captured scan
    epoch now travel as ONE `Option<(subsystem, epoch)>`, with an
    explicit early badarg when channel VTXOs arrive while [channels] is
    disabled (previously an unreachable state that still burned RPC
    walks before failing deep in the transaction).
11. **Catch-up progress logging** — span announcement + a line every
    1000 heights (long silent boots looked hung).
12. **e2e drain-loop deadlines** — the watchman rebroadcasts every tick,
    so each `recv` kept succeeding with non-matching txids and the
    per-receive cap never fired: a genuine infinite-hang risk on
    regression. Both loops now carry 120s wall-clock deadlines.
13. **Trivia** — the `let _ = Mutex::new(())` placeholder removed (+
    import), the duplicated manager-persistence comment in
    `release_contract.rs` deduped, and an SCID boundary test at base
    16,777,215 (pins the modulo band mapping → position 4999 and the
    collision bump across the wrap → 5000).

## Declined (replies to be posted on the threads)

- **Shorten the `chain_lock` section / move the admission `db.write` off
  the lock**: the lock-held commit IS the design — it serializes the
  watch-row insert against block delivery, and `process_parent_watches`
  must finish before the epoch bump or a registration's epoch re-check
  proves nothing. Reviewed to PASS across the commit-3/4 arcs.
- **`confirm_block_transactions` txid micro-opt**: misreads the flow (no
  release-path txid exists on the block path); the filter is active only
  while an open is in flight, the empty-set fast path exists, and the
  cost is ~ms per block.
- **Log-ordering assertion restructure**: already `rposition` for the
  started line; first-close-before-last-start is the property; the close
  line is `tracing`, not slog — no structured id exists without
  production churn.
- **Watch-scan auxiliary indexes** and **first-sight bounded
  concurrency**: speculative perf at stage-1 channel counts; the
  fail-whole-and-retry atomicity on the rare first-sight path is the
  deliberately simple contract.
- **Activation-barrier shutdown arm**: shutdown already unblocks it (the
  SyncManager exits on the signal, its watch senders drop, the barrier's
  `.changed()` errors out with context).
- **Test-suite fixture refactor**: large churn on a five-round-hardened
  suite; the in-place setup chaining is deliberate scenario sequencing.

## Battery after the folds

Per-commit `cargo check --all --tests` green at all 9; units
**515/515**; channels + block_index + postgres e2e **15/15** (the two
new regressions included); clippy `ark-lib` 0, `bark-channels` 0,
`bark-server` 235 baseline; `schema.sql` regen zero drift at the tip.

Design-note amendments in step: the pending-open bullet now names the
dual cap + the proof-carrying-open deferral; (this round rode on the
v0.6.0-rebased stack — see `2026-08-06-v060-rebase-record.md`).
