# MR-5 design note: close by downgrade, end to end (revision 4)

The cooperative close: the standard BOLT-2 close terminally fixes the
final balances, then the Ark leg settles them off-chain by the
**split** — a single-part ARK #5 transfer of the backing VTXO into
`pubkey(A)` / `pubkey(S)` VTXOs matching the close-fixed balances
(spec 08-channels.md "The close" + "Downgrade: close into Ark
balance"). This MR gives both sides the whole flow over the rebased
stack; it is the first consumer of upstream's multi-piece arkoor
builder. Revision 3 = round 1's group-atomic registration, PONR exit
suspension, two-phase outcome capture, kind-scoped watches +
fallback tombstone, cooperative tail with derived obligations,
corrected split arithmetic — plus round 2's BDK broadcast ownership,
zero-aware ledger accounting, the well-defined group-registration
rule over the batched endpoint, and the authenticated `fallback_won`
wire result; revision 4 makes the BDK ownership tag crash-atomic
with legacy backfill.

## 1. Scope and non-goals

In scope: server close-outcome recording and split admission with
group-atomic registration; the client close flow (a phased record
state machine, not one flat state), the split constructor, the
PONR exit suspension, the cooperative exit tail; the symmetric
split-response watches; the cooperative-lead deadline rung.

Non-goals, explicit:
- **No payments.** The closing channel has never carried an HTLC, so
  the drain phase is degenerate. The composition e2e runs
  **open → downgrade → re-upgrade** without a pay step; the payments
  slot extends it. Balance diversity is NOT left untested: the server
  suite crafts arbitrary close balances via the reference client, and
  the client suite gets **injected close fixtures** (a test seam that
  substitutes arbitrary A/S balances into the recorded outcome) so the
  production builder, registration payload, watch lifetime, tail
  selection, and terminal accounting all run against non-degenerate
  splits — nothing below assumes all-to-A.
- **No `option_simple_close`** (upstream LDK stub); `closing_signed`
  with the fee policy of §4.
- **No third-party close destinations.**

## 2. What exists (the seams this MR fills)

lib `ChannelAuthorization::DowngradeInput` gates the split's
construction and admission. captaind: V57's channel row (with the
resolution seam), the parent-exit watch, the atomic per-VTXO
reservation. bark: the record lifecycle, the exit machine and its
terminal barrier, `maintain_channel_deadlines`, the capture-only
broadcaster. Known misfits the design must change, found by review:
the registration channel-hook keys on ChannelFunding OUTPUTS (a
downgrade's outputs are plain pubkey — invisible to it); V57 permits
one unresolved watch per channel and only `kind='upgrade'`; the exit
tail assumes P2A-anchored commitment resolution and floors the
obligation count at one.

## 3. Server side

**Close-outcome recording (commit 1).** The server's node observes the
same BOLT close; at completion the outcome — role-mapped pre-fee final
balances (msat), the closing txid, `closed_at` — is written durably,
keyed to the channel row + backing VTXO + funding outpoint, BEFORE the
close is acknowledged onward; a failed write stops the node. Retention:
until the backing resolves (split registered or expiry-swept) — the
row hangs off the channel record (V58), outliving the LDK state. The
channel row itself is the tombstone: it remains (with its locks) while
any watch references it, even after the outcome row is cleared
post-registration.

**Split admission and the downgrade group (commit 2).** Extends the
upgrade admission; the ordering mirrors it exactly (lock channel row,
then input; validate; persist; sign only after commit):

- input = one channel-funding VTXO with a recorded completed close for
  exactly this backing; single part;
- destinations: keys ∈ {A, S}; per-key totals on the
  floor-plus-remainder basis — each side's msat balance floored to
  sats; when the floors sum one short of `V`, the odd satoshi goes to
  `A`. **A side gets no output only when its floor is zero AND it
  receives no remainder** (A = 500 msat floors to zero yet takes the
  remainder satoshi). Fragmentation: multiple entries for one key are
  admitted ONLY in the exact lender-fragment shape (the lender's
  remainder in `outputs` plus its `D − d` fragment in
  `isolated_outputs`); any other same-key fragmentation is refused
  even when the conflict-winning transaction would be standard;
- standardness verified on the reconstructed conflict-winning
  transaction directly; a NONZERO sub-dust side ⇒ mandatory isolation
  with the lender fragment and the `V ≥ 660` floor — the floor is
  conditional on that case existing. A sole-output split (the other
  side absent) is judged by the direct check alone: a sole 500-sat
  output is not sub-dust and is valid if otherwise standard; a sole
  output below `P2TR_DUST` (330) has no standard conflict-winner and
  is refused;
- depth headroom (checkpointed split spends input+1);
- the spent-mark rides the atomic reservation, before signing, never
  unwound;
- **at cosign the server persists a durable downgrade group**: the
  operation identity, every expected leaf VTXO id of BOTH branches,
  the complete required transaction graph, and the response txids —
  plus an UNARMED downgrade watch row — all in the same transaction as
  the spent-mark. Signatures only after commit.

**Group-atomic registration (commit 2).** The generic registration
endpoint learns groups, well-defined over its batched/retrying
callers (arbitrary repeated VTXO sets; recovery batches then retries
singletons): after deduplication, for every INCOMPLETE group `G`
intersecting the request set `R`, require `members(G) ⊆ R`; unrelated
VTXOs and multiple complete groups in one request are allowed;
channel rows lock in a deterministic order. Once a group is
registered, validated partial re-uploads succeed idempotently. Under the chain lock and the
channel-row lock, atomically: store all signatures, mark ALL leaves
spendable, record the registration, and ARM the watch. Partial uploads
roll back entirely — no leaf becomes spendable, no watch arms, nothing
is recognized (closing the recognize-after-conceding hole: today's
hook keys on ChannelFunding outputs and would let an A-only upload
through as plain VTXOs).

**The response watch (commit 2 + 4).** V58 rescopes the watch table:
kind/input-scoped rows (uniqueness per (channel, kind, input), not per
channel), kind-aware lookups — the upgrade's parent-exit watch and the
downgrade watch coexist and cascade. Duties per spec: **any prefix**
of the old chain confirming is a cue — the armed response
progresses/broadcasts (fee-bumped via P2A) — while only the FINAL
transaction's confirmation decides: registered-first ⇒ the response
wins by construction; final-confirmed-first with the group
unregistered ⇒ a **monotonic `fallback_won` tombstone** — the server
refuses the group's registration from then on, permanently; a reorg
does not reopen it (the signed fallback remains independently
rebroadcastable, so recognition must never return), unlike
response-confirmed resolutions, which stay reorg-sensitive like the
upgrade's.

## 4. Client side

**Two-phase close-outcome capture (commit 3).** The closing
transaction reaches the broadcaster BEFORE `ChannelClosed` supplies
balances, and either peer may initiate — so capture is not an action
step:
- Phase 1: the broadcaster, on a transaction spending a known channel
  funding outpoint, synchronously stores the full candidate keyed
  (channel, funding outpoint, txid) — multiple candidates coexist,
  never one overwritable row — before returning; write failure stops
  the node (the audit queue remains never-consumed — this is a
  dedicated capture, like close events).
- Phase 2: a cooperative `ChannelClosed` event selects the UNIQUELY
  valid cooperative candidate, fills the role-mapped pre-fee
  balances, and **atomically adopts or creates a deterministic close
  action at `OutcomeReady`** — recovery, including a server-initiated
  close the wallet never asked for, always drives from this row. The
  SERVER needs the same broadcaster-side candidate capture for its
  `closing_txid` (its broadcaster currently drops transactions and
  `ChannelClosed` carries no txid) — commit 1 includes it.

**The record's close phases (commit 3; replaces the flat `closing`).**
`ready → negotiating → outcome_ready → registration_pending →
downgraded`, with `fallback_only` reachable from any pre-PONR phase
(and from `registration_pending` ONLY via an authenticated permanent
late-registration refusal). `closed` keeps its resolution
discriminator (downgraded | exited). Usability stays gated on `ready`.

**The `ChannelClose` action (commit 3).**
1. Depth pre-check before `shutdown` (the one-way door); refuse unless
   `ready`. A PEER-initiated close adopts via phase 2 regardless; if
   adopted too deep to split, it routes directly to `fallback_only`.
2. `negotiating`: send/observe `shutdown`, drive `closing_signed`.
   Fee policy per stock LDK: the non-funder accepts anything ≥
   `ChannelCloseMinimum`; the funder's concession ceiling is
   `force_close_avoidance_max_fee_satoshis`, configured and persisted;
   comparisons against the CURRENT relay minimum, not a pinned
   "shared floor".
3. `outcome_ready` (set by phase 2): build the split with a
   **downgrade-specific constructor** around the multi-piece builder —
   it owns the floor-plus-remainder totals, the conditional
   zero-side omission, and the explicit lender-fragment construction
   (`V=660, d=100` ⇒ `A:330 normal, A:230 fragment, S:100 isolated`;
   the generic builder deliberately leaves mixed dust below 660 and
   cannot be handed `[d, V−d]`), mirrored by a validator the server's
   admission reuses. Cosign with `DowngradeInput`; verify every
   partial.
4. The PONR: **`Exit::suspend_for_downgrade_registration`**, under the
   exit write lock, in one compound transition: persist the
   `registration_pending` marker, suspend any exit entry for the
   backing, untrack every old-scope parent and persisted child from
   the transaction manager, and bar restart re-initialization. Every
   broadcast entry point (exit sync, `provide_cpfp_tx`, channel
   sweeps, restart rehydration) rechecks the state under the same
   lock; `start_channel_exit` CASes against `registration_pending`.
   "Pending" ≠ "registered": the marker means eligible-to-have-sent.
   **Broadcast ownership**: exit CPFP children land in the BDK wallet,
   which independently rebroadcasts every unconfirmed transaction
   outside the exit lock — so suspension alone cannot stop them.
   Exit-owned BDK transactions are durably TAGGED and permanently
   excluded from generic BDK rebroadcast; the `ExitTransactionManager`
   is their sole broadcaster (and honors suspension). The tag is
   crash-atomic: under the generation check, the immutable ownership
   record persists BEFORE the BDK changeset is applied — a crash
   between the two leaves a tag without a transaction (harmless),
   never a transaction without a tag. Existing databases are
   backfilled at startup, before any sync: every BDK transaction
   spending a persisted exit-parent anchor is classified and tagged
   fail-closed. An exit-generation token revalidates any child built
   outside the lock before it is stored, closing the
   build-across-suspension race. Ownership resumes only after the
   typed `fallback_won` result. e2e: crash-between-writes and an
   upgrade fixture carrying a pre-existing untagged child.
5. `registration_pending`: register the complete group; on success →
   `downgraded` (retire the suspended fallback, finish the movement,
   store the new VTXOs); on ambiguous RPC results remain pending and
   re-register idempotently; ONLY the authenticated permanent refusal
   transitions to `fallback_only` and resumes the suspended exit —
   and that refusal is MACHINE-READABLE: the registration result (or
   structured status metadata) carries `FALLBACK_WON` plus the
   server-derived group digest, authenticated by the server; every
   other error (generic gRPC status included) keeps the record
   `registration_pending` and retrying.

**The cooperative exit tail (commit 3, replaces flat UE-3).** A
distinct `CooperativeClosing` tail state, selected at the exit's
commitment stage when a recorded outcome exists AND a current-feerate
fee child spending the user's shutdown output is constructible
(persisted: closing tx, the identified local shutdown outpoint,
signing material, the child); otherwise the latest commitment path is
selected as today (the closing tx has no P2A — its CPFP is the user's
own output, and LDK's eventual `StaticOutput` descriptor cannot fund
pre-confirmation bumping). The terminal barrier keeps every existing
layer (health, pending-claims union, state-derived sweep subtraction,
ledger, reorg demotion, deep finality) but the obligation count
becomes zero-aware end to end: the ledger NEVER records a zero-amount
balance (LDK reports `ClaimableAwaitingConfirmations` of zero for a
dust-absent cooperative output, and today's snapshot would record it —
existing zero rows are migrated away), and
`expected = max(selected-tail nonzero ledger obligations, the selected
transaction's owned-output count)` — zero or one per tail, never a
global `.max(1)`. The cooperative fee child is represented as the
tail's confirmed sweep so its shutdown outpoint subtracts like any
other. A zero-to-A close owes nothing and must not wedge; that case
gets a REAL zero-output monitor/ledger test, not a balance-only
fixture.

**The symmetric watch (commit 4).** From registration the client's
`DowngradeWatch` rides the daemon sync: any-prefix confirmation is the
cue to broadcast its retained split levels (its share's genesis),
P2A-bumped; the final transaction is only the decision point it may
have to answer. Scope-ends when no split output remains its own.
Armed-ness is derived (marker + registration state), never a fresh
flag.

**The deadline rungs (commit 4).** `maintain_channel_deadlines`
examines `ready` AND every pre-PONR closing phase: at
`F + cooperative_lead` (checked arithmetic) it starts the close on
`ready` records; at hard `F` any `ready`/pre-PONR/`fallback_only` record transitions
to (or actively resumes) `exiting` (a wedged negotiation, an
unregistered outcome, or a refused registration falls back in time to
confirm the bridge); `registration_pending` and `downgraded` have no
fallback rung. The cooperative tail is selected
inside the exit where viable.

## 5. e2e plan

Core: (1) off-chain settlement, zero mining; (2) balance vectors via
BOTH the server's crafted closes and the client's injected fixtures —
odd-satoshi remainder (incl. A = 500 msat: zero floor + remainder),
absent zero side, `V = 329/330/659/660` boundaries, sub-dust
isolation shape, refusal vectors (no close record, wrong totals,
foreign key, non-lender fragmentation, sub-660-with-nonzero-dust-side,
depth); (3) crash matrix — broadcaster-capture↔ChannelClosed,
cosign↔eligibility (fallback still safe and WINS), across the PONR
(old scope never broadcast; idempotent completion); (4) **partial
registration**: A-only, S-only, and missing-isolation-branch uploads
— no leaf spendable, no watch armed; (5) prefix-then-register
(response progresses, registration still lands) vs
final-then-register (permanent refusal, surviving a reorg of the
final tx); (6) both watches responding, and upgrade+downgrade watches
coexisting/cascading; (7) D5 concurrency — a live exit with a
persisted CPFP child crossing the suspension, restart rehydration,
stale `provide_cpfp_tx`, deadline-scheduler racing the PONR; (8)
server-initiated close adoption (incl. crash between capture and
ChannelClosed); (9) the cooperative tail — direct-output CPFP,
no-P2A handling, zero-local-output selection of the commitment path,
terminal + movement accounting; (10) wedged negotiation → hard-F
force-close via the commitment (no closing tx exists); (11) watch
lifetimes — server retention across onward movement, client
termination only after every owned output is gone, and a
response-CONFIRMED resolution reorged (reopens, unlike the
tombstone); (12) BDK ownership — a tagged exit child is never
generically rebroadcast, across restart; (13) a REAL zero-output
close at the monitor level (not a balance fixture) reaching terminal;
(14) composition:
open → downgrade → re-upgrade (depth ladder + pre-close depth
refusal), pay step added at the payments slot.

## 6. Commit plan (4 commits, each independently green)

1. **captaind: record the close outcome** — outcome row (V58 part 1),
   write-before-ack, retention to resolution, tombstone semantics.
2. **captaind: admit the sanctioned split** — admission + downgrade
   group at cosign + group-atomic registration + kind-scoped watch
   schema (V58 part 2) + armed response + `fallback_won` tombstone;
   full refusal-vector e2e.
3. **bark: the close flow** — two-phase capture, phased record states,
   the ChannelClose action, the downgrade split constructor, the exit
   suspension (PONR), the cooperative tail; sdk e2e (settlement,
   vectors, crash matrix, D5 concurrency).
4. **bark: the symmetric watch + deadline rungs** — DowngradeWatch,
   scheduler rungs over the new phases, composition e2e.

## 7. Decisions (revised per G2 R1; for review)

- **D1′**: no-pay composition + server crafted balances + client
  injected close fixtures (production builder/registration/tail all
  exercised on non-degenerate balances).
- **D2′**: explicit destinations via a downgrade-specific
  constructor/validator pair around the multi-piece builder; the
  change-pieces machinery never engaged.
- **D3′**: two-phase capture — broadcaster stores the candidate
  synchronously (fail-closed), `ChannelClosed` validates + fills
  balances + adopts/creates the action; the audit queue untouched.
- **D4′**: phased record states (`negotiating`, `outcome_ready`,
  `registration_pending`, `downgraded`, `fallback_only`), not a flat
  `closing`.
- **D5′**: `registration_pending` + `Exit::suspend_for_downgrade_
  registration` under the exit write lock (compound suspend/untrack/
  bar-restart; all broadcast entry points recheck under the lock);
  "registered" reserved for acknowledged complete registration;
  resume-to-fallback only on the authenticated permanent refusal.
- **D6′**: LDK-native fee knobs (`ChannelCloseMinimum` acceptance,
  funder concession via `force_close_avoidance_max_fee_satoshis`),
  policy persisted, compared against the live relay minimum.
