# G2: the close-by-downgrade design review (2026-08-24, 4 rounds to PASS)

The MR-5 design note (2026-08-24-mr5-downgrade-design.md) converged
through four adversarial rounds, each grounded in reads of the rebased
stack. Every adopted mechanism below is codex's own suggestion.

## Round 1: FAIL — four structural blockers

1. **Partial registration = recognize-after-conceding.** The shipped
   registration hook keys on ChannelFunding OUTPUTS; a downgrade's
   outputs are plain pubkey, so an A-only upload would register as
   ordinary VTXOs — the exact double settlement the spec's
   complete-split rule exists to prevent. → the durable **downgrade
   group** at cosign (operation identity, both branches' leaf ids, the
   full graph, response txids, an unarmed watch — one transaction with
   the spent-mark) and **group-atomic registration** (all-or-nothing,
   arm-on-commit).
2. **The PONR "force-close guard" was unenforceable**: the transaction
   manager rebroadcasts persisted packages, provide_cpfp_tx broadcasts
   immediately, sweeps broadcast directly, the deadline scheduler
   races — all outside any marker the action could write. → distinct
   `registration_pending` + `Exit::suspend_for_downgrade_registration`
   under the exit write lock (compound suspend/untrack/bar-restart;
   every broadcast entry point rechecks under the lock; CAS on
   start_channel_exit; "registered" reserved for acknowledged).
3. **UE-3 misfit the shipped tail**: the closing tx has no P2A (its
   CPFP is the user's shutdown output), and the terminal barrier's
   `.max(1)` floor either wedges a zero-output close or silently
   assumes all-to-A. → a distinct `CooperativeClosing` tail selected
   only when a current-feerate child is constructible, with the
   obligation count derived from the selected transaction.
4. **Close capture and counterparty-initiated closes**: the closing tx
   reaches the broadcaster BEFORE ChannelClosed supplies balances, and
   either peer may initiate. → the two-phase capture (broadcaster
   stores the candidate synchronously fail-closed; ChannelClosed
   validates, fills role-mapped balances, adopts-or-creates the
   action).
   Plus: kind/input-scoped watch rows (V57 allows one watch per
   channel, upgrade-kind only) with a monotonic `fallback_won`
   tombstone (a reorg must not reopen the refusal); the corrected
   split arithmetic (zero-floor-plus-remainder; conditional 660;
   lender-fragment-only fragmentation; a downgrade-specific
   constructor/validator — the generic builder deliberately leaves
   mixed dust below 660); deadline rungs over the new phases.

## Round 2: FAIL — four second-order gaps

BDK's generic sync rebroadcasts every unconfirmed wallet transaction
outside the exit lock (suspension alone can't stop persisted CPFP
children) → durable exit-ownership tags + sole-broadcaster rule +
generation token. The ledger records zero-amount balances (LDK reports
zero ClaimableAwaitingConfirmations for dust-absent cooperative
outputs) → never record zero, migrate existing, expected =
max(nonzero ledger, selected tx's owned-output count), the fee child
counted as the tail's sweep. "Exact complete group" undefined over the
batched endpoint → members(G) ⊆ R for incomplete groups after dedup;
unrelated VTXOs and multiple complete groups fine; idempotent
re-uploads post-registration. The permanent refusal needs a
machine-readable identity → authenticated FALLBACK_WON + group digest;
every other error stays pending.

## Round 3: FAIL — one residual

The BDK tag lacked crash-atomic creation and legacy backfill → tag
persists before the BDK changeset (tag-without-tx harmless, never the
reverse); startup backfill classifies transactions spending persisted
exit-parent anchors, fail-closed; crash-between-writes and
upgrade-fixture e2e.

## Round 4: PASS

"Durable tag-first ordering prevents tx-without-tag, while fail-closed
pre-sync backfill covers legacy transactions and both crash boundaries
have e2e coverage."

The note (revision 4) is the implementation contract; its §7 carries
the revised decisions D1'-D6' for the go/no-go.
