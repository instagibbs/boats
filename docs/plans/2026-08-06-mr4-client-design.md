# MR-4 design — bark client: channel open + unilateral exit (lifecycle only)

**Status**: G1 draft, pre-codex. **Base**: the posted stack on master
`900cd3ca` (opener + protocol + captaind). **Repo**: bark-stage1, bookmark
`ark8-channels-stage1-client` to be cut off the captaind tip at code
start (re-fetch + rebase-to-tip first, per cadence).

## 1. Scope and non-goals

The client half of everything MR-3 shipped the server half of: bark can
**open** a channel to captaind by upgrade, **hold** it correctly across
restarts, and **always recover** unilaterally — the full exit story
through the bridge to on-chain claims. Economically the channel stays
inert: captaind refuses every HTLC (stage-1 posture), so no payment
flows; usability is proven by the MR-3-style cooperative HTLC probe that
fails back.

**Out of scope, with owners:**
- *Payments* (captaind forwarding + caps + kill switch, the real
  fee-bump reserve, the claim-race tightening, per-HTLC/invoice floor
  enforcement) — the payments MR. Floor checks on HTLC acceptance and
  invoice creation would be dead code here (nothing carries HTLCs); the
  series review's B1 lesson (no machinery without a production caller)
  applies verbatim. M4 keeps only the **open-time** timing obligations
  (§7), which are live immediately.
- *Close by downgrade* — the next MR (M5, before payments: stage 1's
  goal is the upgrade/downgrade lifecycle, and until it lands the
  expiry treadmill's only rung is the unilateral exit — which is why
  M4's deadline automation falls through to force-close, §9).
- *CLI subcommands, REST `/channels`, bark-json DTOs* — the surface MR,
  per the plan. M4 exposes the `Wallet` API; the e2e drives it there.
- *LDK `BumpTransactionEventHandler` / `WalletSource` adapter* — not
  needed until HTLC claims exist (payments MR). M4's post-force-close
  claims are: commitment CPFP (P2A, through bark's existing
  `make_signed_p2a_cpfp`), `to_local` sweep (self-paying,
  `SpendableOutputs`), justice (self-paying). The `BumpTransaction`
  events are drained only to *obtain* the commitment (old-branch
  pattern), never handled for fees.

## 2. The wire addition (the MR's only protocol change)

`ArkInfo` gains three optional fields, populated iff `supports_channels`:
`channel_node_id` (33 bytes), `channel_addresses` (repeated string), and
`channel_claim_slack` (u32). Today the channel node's identity/address
are obtainable only from the daemon's structured log — the e2e reads
`ChannelsSubsystemStarted` off stdout, which no production client can
do. Additive, next free field numbers, openapi regenerated.

Population rules: `channel_node_id` from the subsystem;
`channel_addresses` from a **new operator config field**
`[channels].advertise_addresses` — the bound listen address is NOT
dialable in general (the default binds `0.0.0.0`), so the server never
advertises it; with the config empty the field is absent and a client
MUST refuse to open (clear error) rather than guess. And
`channel_claim_slack` advertises the server's `cltv_claim_slack`: it is
admission policy exactly like the already-advertised
`max_vtxo_exit_depth`, and without it a client's local runway check can
pass while the server refuses (the values must agree; the client uses
the advertised value in §7, not a local default).

**Retry-signal fix (small server-side commit).** The two scan-epoch
aborts ("the chain view changed during admission/registration; retry")
and the RPC decode failure currently surface as `Code::Internal` with
the opaque default body — indistinguishable from bugs. Fix: map the
epoch aborts to `Code::Aborted` (the canonical "concurrency conflict,
retry" code; a tag in the error machinery + `ToStatus` arm), and
`.badarg` the decode context so malformed requests are
`InvalidArgument` as `server-rpc`'s own conversion already intended.
Client retry policy then keys on codes, not strings — closing the
deferred "not-accepted substring" wart for the channel paths.

## 3. The client node

**Reuse `bark-channels` as-is** — it is already deployment-agnostic
(node assembly, designated-type config, `process_events_async` driver,
all behind `Seams{broadcaster, fee_estimator, logger, persist}` +
`ManagerStore`). The client supplies:

- **`Persist<Signer>` over sqlite** (synchronous; monitors table in
  m0043). rusqlite is already blocking, so the server's dedicated-
  runtime deadlock dance does not carry — but the contract does:
  persist-before-sign, never report success on failure, and a failed
  monitor write must stop the node (fail closed), mirroring captaind.
- **`KVStore` over sqlite** for the manager singleton (one-row table,
  key-gated to exactly LDK's channel-manager key, like `ManagerStore`).
- **A routing broadcaster — the one real divergence from captaind.**
  The server captures and drops every LDK broadcast; the client must
  *keep* them: LDK-originated transactions (commitment on force-close,
  justice, sweeps) are the client's own recovery path. Design: the
  broadcaster is a **durable queue** (m0043 table) drained by the exit
  driver (§8), which tracks, packages, and CPFPs through the existing
  `ExitTransactionManager` machinery — never a raw relay from inside
  LDK's callback. Pre-actualization broadcasts (spending a funding that
  isn't on-chain) queue harmlessly and become valid exactly when the
  exit actualizes the bridge; nothing is lost, nothing relays early.
- **`FeeEstimator`** wrapping `ChainSource::fee_rates()`.

Lifecycle: a `channels: Option<...>` field on `WalletInner` (the
`onchain` pattern), assembled at wallet open when the config enables
it; a fifth `supervised(...)` task in the daemon runs the
`bark-channels` driver + a peer-reconnect loop (old-branch
`run_lightning_peer_process` shape: reconnect when the peer set is
empty, `timer_tick_occurred` on the LDK cadence).

**Event ownership (every LDK event the client's handler consumes, with
its durability-vs-acknowledgement rule; 0.2.4 does not redeliver an
acknowledged event):**
- `FundingGenerationReady` / `ChannelPending` / `ChannelReady`:
  consumed by the ChannelOpen action's polling; the durable authority
  is the action checkpoint + channel record, both **reconcilable from
  the reloaded manager** (funding outpoint, `is_usable`) — losing the
  event is recoverable, ack-before-durable is safe.
- `FundingTxBroadcastSafe`: emitted by the manual-funding path —
  handled as an explicit **no-op** (the "funding tx" is the bridge,
  unsigned at that moment and never LDK's to broadcast; the routing
  broadcaster would only queue it anyway, but the event is consumed
  deliberately, not dropped by omission).
- `SpendableOutputs`: the descriptor is **persisted durably before the
  handler returns** (a `bark_ldk_spendable_outputs` row in m0043) — the
  driver runs no LDK `OutputSweeper` and the event never returns;
  the exit driver's pump sweeps from the persisted rows.
- `BumpTransaction(ChannelClose)`: full payload persisted before ack
  (§5).
- `ChannelClosed`: a durable **signal, never terminality** (§9) —
  idempotent upsert of `bark_channel.close_observed_at` (+ the closure
  reason) before ack; re-emission overwrites nothing meaningful.
- `PaymentClaimable`/forwarding events: none exist in M4 (no HTLCs);
  the handler fail-closes on them defensively. Reload facts from
MR-0 carry: monitors read before the manager, `watch_channel` (here:
`load_existing_monitor`) after, KeysManager gets the preserved seed +
a **fresh** start time, reestablish converges unprompted after
reconnect. The old branch's funding-key-override dance existed for its
fork-specific signer; stock LDK derives channel signers
deterministically from the `channel_keys_id` stored in each monitor
given the same seed — **settled by the G1 review against the 0.2.4
source**: reload-stable, no override machinery needed. The persisted
channel record (§5) still carries `user_channel_id` as the stable join
key for the client's own bookkeeping.

## 4. The open flow (the ChannelOpen action)

Modeled as a `WalletActionCheckpoint::ChannelOpen` variant driven by
the action executor — which structurally fixes the old branch's honest
gap: its upgrade flow had *no* pending-intent record and *no* recovery
loop; a crash between registration and the feed left a channel nobody
re-drove. Three framework facts the design binds to explicitly (the
framework does none of this "automatically"):

- **Resumption is wired per action kind**: `Wallet::sync` gains a
  `sync_pending_channel_opens` arm loading `ChannelOpen` checkpoints
  and re-driving them (`UntilParkOrDone`), exactly like the other four
  kinds; the daemon's sync task therefore re-drives opens with no new
  loop.
- **The executor persists the returned checkpoint *after* `advance()`
  returns**, so side effects inside a step must be idempotent against
  re-execution from the *previous* checkpoint — each phase below states
  its re-entry behavior under that rule.
- **Initial ordering follows the arkoor pattern**: lock the input
  (`VtxoLockHolder::Action{id}`) *before* writing the first checkpoint,
  so no window exists where a checkpointed action references an
  unlocked input. The inverse window (crash between the lock and the
  first checkpoint ⇒ an orphaned lock nothing re-drives) is closed
  **atomically, not by sweeping**: a new persister method performs the
  lock and the initial checkpoint write in one sqlite transaction, and
  `ChannelOpen` creation uses it. (A reconciling sweep was considered
  and rejected — `Wallet::sync` runs concurrently with action creation,
  so "lock exists, checkpoint absent" is transiently true for LIVE
  actions and a sweep would race them.) The arkoor path keeps its
  historical ordering; migrating it to the atomic method is
  noted as an optional follow-up, not churned here.

**The checkpoint carries the complete deterministic plan.** The
identity fields persisted at birth (before any RPC): the input VTXO id,
the funding amount, the change key indices and derived change policy,
`use_checkpoint`, `user_channel_id`, the server node id, and — once
establishment assigns them — the permanent `channel_id` and the
negotiated funding redeem script. From these the client can rebuild the
**byte-identical unsigned package** (and therefore the same backing
VTXO id and bridge txid) on any re-drive: that is what makes the
server's same-backing re-cosign carve-out reachable after every crash,
with only the MuSig2 secret nonces volatile (fresh per attempt by
design). A plan that cannot be reproduced exactly is a bug, and the
crash-matrix e2e (§10.2) proves reproduction at every boundary.

Ordering (the old branch's eager-re-key sequence, confirmed by the
reference e2e):

1. **Select + lock the input** (`VtxoLockHolder::Action{id}`): exactly
   one `pubkey` VTXO covering the amount (else "refresh or consolidate
   first" — I-1's refresh-first policy); movement row born
   (`Subsystem::CHANNEL`, `get_or_create_movement_with_action`).
2. **Open-time checks** (§7) against the cached `ark_info` + the input.
3. **Plan the destination before establishment**: build the unsigned
   package (`ArkoorPackageBuilder`, `ChannelAuthorization::
   UpgradeOutput`, channel-funding destination = input's own user key,
   change via checkpointed shape when present; pass-through
   single-output shape may set `use_checkpoint = false`). Txids are
   witness-independent, so **`channel_vtxo.point()` is known now** —
   that is what lets establishment run against a definite funding
   *input*. The `bridge_txid` is NOT known yet: the bridge's output
   commits to the negotiated funding keys, which only exist after
   `FundingGenerationReady` (step 4).
4. **LDK establishment — two persisted substeps** (the crash boundary
   codex R2 named): *(4a)* connect (node id + address from the new
   `ArkInfo` fields), `create_channel(server_node_id, amount, 0 push,
   user_channel_id, ..)`, await `FundingGenerationReady`, verify the
   negotiated funding script's `.to_p2wsh()` shape, **persist the
   checkpoint carrying the funding script + derived `bridge_txid`**
   (`Advance::Next` boundary — durable before any LDK state depends on
   it); *(4b)* build the bridge locally (`ark::channel::bridge_tx`),
   then
   `unsafe_manual_funding_transaction_generated(temp_id, peer,
   bridge_txid:0)`. Await `ChannelPending` → **the permanent
   `channel_id` exists now** (eager re-key: no post-hoc rewrite ever);
   persist it into the checkpoint. Verify the channel monitor exists at
   `bridge_txid:0`.
   *Crash semantics — inspect before acting, never blind-retry*: a
   post-funding channel survives an LDK restart, so re-entry FIRST
   looks the channel up by `user_channel_id` in the reloaded manager.
   Found and past `ChannelPending` with our recorded `channel_id` →
   **resume this same channel** (fall through to the cosign phase; the
   server row is `awaiting_upgrade` and waits). Found past
   `ChannelPending` but the checkpoint has **no** `channel_id` yet (the
   crash landed between the event and the checkpoint write) → **adopt
   the manager's channel id**: validate it against the checkpointed
   plan (funding outpoint == the plan's `bridge_txid:0`, funding script
   matches), persist it into the checkpoint, then resume. Found but pre-funding,
   or not found → explicitly close the remnant
   (`close_channel`/force-close of the unfunded channel is a local
   no-op on-chain) and only then start a fresh attempt — never two
   live LDK channels for one action, never a leaked server pending row
   beyond the one the reaper owns. Attempts are bounded; exhaustion
   parks the action with the error.
5. **Cosign**: rebuild the package from the checkpointed plan, generate
   user nonces + the bridge nonce pair (secret halves in memory only —
   never persisted; MuSig2 nonces must not survive a crash), attach
   `BridgeCosignRequest{channel_id, pub_nonce}`, call the cosign RPC.
   *Crash/retry semantics — the C2 window*: the server flips the row to
   `cosigned` **before** the client sees the response, so a crash here
   recovers by rebuilding the byte-identical package from the plan
   (same backing id) and re-requesting: the same-backing carve-out
   re-cosigns with fresh nonces on both sides. The client always
   completes against whichever response it actually received; nothing
   from a lost response is needed.
6. **Verify before completing** (old-branch rule, kept verbatim):
   verify the server's bridge partial and the aggregated 64-byte
   signature (`finish_bridge_cosign` does both) *before* `user_cosign`
   — completing without a valid bridge would mint a channel-funding
   VTXO with no unilateral exit.
7. **Persist the channel record** (§5): backing VTXO (signed bytes),
   bridge tx + aggregate signature, funding outpoint/redeem script,
   pinned `(exit_delta, max_vtxo_exit_depth, expiry_height)`,
   `user_channel_id`, state `cosigned`. Durable **before**
   registration — the exit story exists from this point regardless of
   anything else.
8. **Register** (`RegisterVtxoTransactions` with the signed chain) —
   the point of no return on the Ark side; idempotent server-side.
   Then mark the input spent, store change VTXOs (ordinary path — the
   channel-funding VTXO itself deliberately never enters
   `store_vtxos`, which refuses its policy by design).
9. **Feed + ready**: self-feed the node the **signed** bridge
   (funder-side LDK verifies witness completeness) at the backing
   VTXO's chain-anchor height — synthetic `tx_index` per §6 — then
   `best_block_updated` to the real tip (never backdate; never
   synthesize future blocks). Await `Event::ChannelReady`, flip the
   record to `ready`, finish the movement, `Done`.

The gate property the e2e must re-prove from the client side: usable
**after** registration and **not before** — the self-feed alone (step
9 run early) must produce no `ChannelReady`, exactly as the reference
test pins.

## 5. The channel record and storage (m0043–m0045)

Three sqlite migrations, one per owning commit so each commit is
independently green (m0043: the LDK persistence tables + broadcast
queue; m0044: the channel record; m0045: the exit table's
`channel_data` column, §8). Contents:
- `bark_channel` — one row per channel: `channel_id` (PK, 32B),
  `user_channel_id`, `counterparty_node_id`, backing VTXO id + signed
  bytes (Full), bridge tx bytes + aggregate signature, funding
  outpoint + redeem script, pinned `exit_delta` /
  `max_vtxo_exit_depth` / `expiry_height`, synthetic SCID allocation
  (`height`, `tx_index`), state
  (`cosigned | registered | ready | exiting | closed`) plus
  `confirmation_fed_at`, `close_observed_at`, and `close_reason` (TEXT,
  LDK's `ClosureReason` rendered) — §3's `ChannelClosed` signal writes
  the pair as one idempotent upsert (first observation wins; a
  re-emission with the same reason is a no-op, a different reason is
  logged and ignored); a `ChannelClosed` for a channel with **no
  record** (an open refused or reaped pre-record) is acknowledged as a
  no-op — there is nothing to mark. Plus timestamps. The `registered` state
  is load-bearing for recovery: a reload must distinguish
  cosigned-but-unregistered (feeding would violate the readiness gate —
  the channel must stay unusable) from registered-but-unfed (feed now,
  level-triggered) from fed (no-op). This is the client's exit story
  and MUST survive anything (old-branch criticality: bridge/exit
  material = CRITICAL).
- `bark_ldk_channel_monitor` (name PK, blob) and
  `bark_ldk_channel_manager` (singleton) — the two LDK persistence
  tables (CRITICAL / HIGH).
- `bark_ldk_broadcast_queue` — the routing broadcaster's durable queue
  (txid PK, bytes, seen_at, drained_at). A companion
  `bark_ldk_channel_close_events` table durably captures the
  **complete** `BumpTransaction::ChannelClose` payload — keyed by
  `claim_id` with `channel_id` + `counterparty_node_id` columns (the
  event's identity fields), plus target feerate, commitment fee, the
  commitment transaction (+ its txid column for the exit driver's
  join), the anchor descriptor, and the pending-HTLC list (empty in
  M4, schema-present) — persisted before the handler acknowledges:
  0.2.4 does not redeliver, and losing one would orphan the commitment
  from the exit driver's view. LDK may re-emit for the same claim:
  writes are idempotent upserts on `claim_id`, latest payload wins.
  `drained_at` semantics on both tables: written only **after** the
  exit manager has durably tracked the transaction (and, for a CPFP
  child, persisted it via the existing `store_exit_child_tx`) — a crash
  between drain and track re-drains idempotently.
- `bark_ldk_spendable_outputs` — persisted `SpendableOutputs`
  descriptors (§3's event-ownership rule), consumed by the exit
  driver's sweep step, marked swept when the sweep confirms. Carries
  the `abandoned_uneconomical` lifecycle as columns
  (`abandoned_at`, `abandoned_feerate`): set under §8's rule, cleared
  (un-abandoned) if fee rates fall below the recorded threshold while
  the exit is live, kept for audit after terminal — this is the
  persisted half of the `pending_claims()` filter.
- The channel-funding VTXO is stored **only** here — the `store_vtxos`
  refusal stays the guard; balance surfaces never see it.

## 6. The client confirmation feed and SCID

Mirror of the server's release mechanics with the funder-side
differences: the client feeds its **own** node, the **signed** bridge,
anchored at the backing VTXO's real chain-anchor confirmation height
(no backdating — old branch's backdate machinery is confirmed dead and
is not ported). Synthetic `tx_index`: allocated once per scope,
persisted in `bark_channel` before the node ever sees it, drawn
deterministically from the bridge txid into the `2500..2²⁴` band with
persisted collision handling — same rules the spec's virtual-funding
bullet requires (node-local fiction, node-unique, stable per scope).
Route hints, when they come (payments MR), use `inbound_scid_alias`,
never the synthetic SCID. Boot behavior: level-triggered idempotent
re-feed (same-header re-feeds are safe per the MR-0 pin) whenever the
record says `ready`/`cosigned`-registered but the reloaded manager's
view lacks the confirmation; ordinary reloads carry it in the manager
blob and no-op.

## 7. Open-time checks (the timing obligations that are live in M4)

Client-side, before the cosign (all mirrored from admission so honest
clients never burn a refusal, and refusals stay meaningful):
- `input.exit_delta() == ark_info.vtxo_exit_delta` (drifted input ⇒
  refresh first);
- resulting depth (pass-through +1 / checkpointed +2) ≤ pinned
  `max_vtxo_exit_depth − 2` (the split headroom — reserved for M5's
  downgrade);
- runway: `expiry_height − tip` must exceed
  `max_vtxo_exit_depth + exit_delta + claim_slack`, with `claim_slack`
  taken from the **advertised** `channel_claim_slack` (§2) — never a
  local default, or the client's check and the server's admission can
  disagree;
- **exit-fee reserve**: the on-chain wallet must hold a configured
  minimum for the unilateral path's CPFP work (the bridge's and the
  commitment's P2A anchors are fee *attachment points*, not fee
  *sources* — bark's CPFP selects confirmed wallet inputs, and a
  zero-balance wallet has no usable exit). Default: refuse the open
  below a conservative estimate — per-level bump × depth + bridge +
  commitment + a sweep-family headroom multiplier, at a configured
  feerate — overridable by an explicit config/flag. **Stated honestly
  as necessary-not-sufficient**: it is a point-in-time check; nothing
  earmarks the balance afterward (deliberate — wallet-UX cost of real
  reservation is out of proportion here), a later spend can recreate
  the exposure, and the existing CPFP loop's skip-and-retry semantics
  are the backstop (an underfunded exit stalls and resumes when funds
  appear rather than failing). Mitigations that ARE in scope: the
  maintenance loop re-checks the reserve for every open channel and
  surfaces a warning movement/log when below; the same estimate gates
  `initiate_channel_exit` with a warn-not-refuse (an exit must never
  be blocked on a warning). The residual is documented in the config
  docs;
- `supports_channels` advertised + `channel_node_id`/addresses present
  (PV-8's client half), server pubkey consistency via the existing
  `check_and_store_server_keys`.

No per-HTLC or invoice floor in this MR (§1). The pinned values are
stored per channel at open and **never** re-read from live ark info
for an existing channel (BR-3/4/8 client half).

## 8. The exit extension

The channel VTXO's unilateral exit = the generic exit machine plus a
channel tail. The G1 review established the graft is NOT free — the
generic machine hydrates from the wallet VTXO store (which refuses
`ChannelFunding` by design), initializes from `Vtxo::transactions()`,
and its `ProgressContext` assumes a generic VTXO — so the integration
is specified concretely, on the old branch's proven m0029 pattern (its
exit table grew a nullable `channel_data` column and the machine
learned one alternate hydration path):

- **Hydration is ONE seam, not per-call-site branches.** Three G1
  rounds of enumeration-chasing (load, per-tick `progress()` reloads,
  `initialize()`, `StoredExit` rebuild, status/listing `get_full_vtxo`,
  cancel `get_vtxo`, `drain_exits`, claim signing, CPFP inspection)
  establish the structural fact: every surface of the exit machine
  loads VTXOs from the wallet store. The design therefore introduces
  an **`ExitVtxoSource`** seam — `bare(id) -> WalletVtxo-equivalent`,
  `full(id) -> Vtxo<Full>` — with two implementations (the wallet
  store; the channel record via the exit entry's payload), and a
  mechanical refactor routes **every** VTXO load in the exit machine
  through it. The rule is checkable by grep ("the exit crate never
  calls `get_vtxo`/`get_full_vtxo` directly") and closes the whole
  enumeration class at once. `StoredExit` gains the optional
  `channel_data` payload (m0045): `{channel_id, pinned_exit_delta,
  bridge tx bytes + aggregate signature, signed genesis chain,
  cooperative-close placeholder (M5), full vtxo bytes}` —
  `pinned_exit_delta` is IN the payload (the Processing handoff
  consumes it; no join with `bark_channel` on the hot path), validated
  at initiation against the bridge bytes' input sequence. State-update
  writes preserve the payload column untouched (it is not part of the
  state JSON). `exit_txids` and the transaction manager's tracking are
  initialized from the payload's signed chain **plus the bridge** —
  never from `vtxo.transactions()`.
- **The handoff is two explicit branches, and the first sits where the
  generic machine would otherwise fail.** A `ChannelFunding` VTXO has
  **no user-signable unilateral-exit clause** — its delay lives on the
  bridge input's `nSequence` — so generic `Processing`'s
  `find_signable_clause` step would error before `AwaitingDelta` is
  ever reached. Branch 1 therefore sits at the `Processing →
  AwaitingDelta` transition: payload present → the wait delta is the
  payload's `pinned_exit_delta` (validated against the bridge bytes'
  input sequence), and the clause lookup is skipped. Branch 2 is
  `AwaitingDelta`'s transition: payload present → `ChannelBridge`,
  never `Claimable`. Consequently a channel exit **never occupies a
  generic post-delta state**: `list_claimable`,
  `sign_exit_claim_inputs`, and `drain_exits` see nothing to do;
  `warrants_exited_vtxo` is never consulted on a store-absent VTXO
  because the reconcile branches on the payload *before* the store
  call — the channel arm updates the record and never calls
  `mark_vtxos_as_exited`. The progress driver's pre-state `Spent`
  check routes through the source like every other load.
  **Cancelability and its rollback**: generic pre-broadcast rule
  through the source; refused from `ChannelBridge` onward (no
  un-broadcast exists). A pre-bridge cancellation restores
  `bark_channel.state → ready` and cancels the exit movement in the
  same compound transaction that removes the exit entry — the channel
  is simply open again, nothing on-chain happened.
- **CPFP surfaces** are extended by state, not by store access:
  `exits_needing_cpfp` surfaces `ChannelBridge`'s bridge and
  `ChannelCommitment`'s commitment as bumpable parents (both tracked
  in the same transaction manager, both P2A).
- **Compound atomic persister operations — new trait methods, not
  call sequences.** The current `BarkPersister` runs locks,
  checkpoints, movements, and exit rows in separate transactions, so
  the two multi-write moments in this MR get **explicit compound
  methods** every backend implements in one transaction:
  `create_action_with_locks(checkpoint, vtxo_ids)` (used by
  ChannelOpen creation — closes the orphan-lock window by
  construction) and `initiate_channel_exit(channel_id, ...)` (exit
  row + payload + movement creation + `movement_id` linkage + record
  flip to `exiting`); its inverse
  `cancel_channel_exit(channel_id)` (entry removal + movement cancel
  + record restore) is the §cancel rollback. Adaptor backends
  implement the same methods over their own transaction primitive.
  **Replay/CAS contracts (crash-recovery semantics, part of the
  design):** `create_action_with_locks` is idempotent on the same
  action id — a retry finds the existing checkpoint and returns it,
  never duplicating locks; `initiate_channel_exit` is a compare-and-set
  on `ready → exiting` — a retry with the row already `exiting` reuses
  the existing exit entry and movement, and any other state (`closed`,
  never-existed, mid-cancel) refuses rather than resurrecting;
  `cancel_channel_exit` is valid only before bridge broadcast,
  performs the whole inverse atomically, and is idempotent (a repeat
  on an already-canceled exit is a no-op).
- **Movement ownership**: created and linked by the compound
  initiation above. The terminal reconcile finishes it
  (`finish_movement_with_update`: exited backing id; swept amounts as
  received-on-chain) — level-triggered off `record.state != closed ∨
  movement pending`, re-run every tick until both flip, so a failed
  best-effort finish retries durably.
- **Context**: `ProgressContext` carries the optional channel payload
  + the optional driver handle; the generic states ignore both. The
  channel states are new `ExitState` variants with **fully specified
  persisted shapes** (all `BlockRef`s hash-bound; this is what makes
  every reorg recovery reconstructible from the row alone):
  - `ChannelBridge { bridge_txid, confirmed: Option<BlockRef> }` (the
    bridge bytes live in the payload);
  - `ChannelCommitment { commitment_txid, commitment_tx: Vec<u8>,
    confirmed: Option<BlockRef>, sweeps: Vec<SweepEntry> }` where
    `SweepEntry { txid, tx: Vec<u8>, kind: ToLocal | StaticPayment |
    Justice, confirmed: Option<BlockRef> }` — the commitment bytes are
    persisted at selection time (from the captured close event or the
    counterparty's confirmed spend), so a reorged
    `ChannelSwept` can rebuild this state from the row;
  - `ChannelSwept { commitment_txid, commitment_tx: Vec<u8>,
    commitment_confirmed: BlockRef, sweeps: Vec<SweepEntry> }` — the
    commitment bytes are carried here too, so demotion back to
    `ChannelCommitment` on a reorg reconstructs entirely from the row
    (never from `history`).

  Lifecycle and finality: `ChannelSwept` **stays in `LIVE_STATES`**
  (it must reload and keep re-validating). Finality = every retained
  `BlockRef` at least `DEEPLY_CONFIRMED` deep, at which point the
  state advances to a true terminal
  `ChannelClaimed { commitment: (Txid, BlockRef),
  sweeps: Vec<(Txid, BlockRef)> }` (`FINISHED_STATES`; the finality
  evidence is retained, the transaction bytes are dropped — a
  finality-deep state never demotes): the record closes and the
  movement finishes. Monitor archival is **not** part of this
  transition — it follows LDK's own archival clock (§9). The
  recovering window is bounded by the same finality assumption the
  rest of the system uses, and nothing closes early. When a reorg changes the **winning commitment**, sweep
  entries derived from the losing commitment are dropped (their
  inputs no longer exist) and the driver re-derives from the monitors
  against the new winner.
- **Genesis + delta**: existing `Start → Processing` over the persisted
  chain; `AwaitingDelta` with the delta sourced from the bridge input's
  `nSequence` (`pinned_exit_delta`) — a one-branch change where the
  generic code reads `clause.sequence()`.
- **The driver trait** (exit crate stays LDK-free; the impl lives with
  the node): `feed_confirmation` / `feed_unconfirmation` /
  `obtain_commitment` / `pump(tip)` / `pending_claims()` — the old
  branch's driver surface, re-cut. Durability lives in the exit state
  + the channel record + the broadcast queue, never in the driver.
- *ChannelBridge*: broadcast + CPFP the bridge (its P2A anchor is a fee
  **attachment point**; fees come from confirmed wallet inputs — the §7
  reserve exists precisely so this step is fundable) via
  `ExitTransactionManager`; hash-bound confirmation recheck; a reorg
  that unconfirms the genesis parent falls all the way back to
  `Processing`. On confirmation: feed the node, then obtain the
  commitment — a counterparty commitment if one already spent the
  funding, else force-close and take LDK's commitment **from the live
  monitor, never a stored snapshot** (the revoked-snapshot hazard, kept
  verbatim). The `ChannelClose` bump event that carries it was durably
  captured by the broadcaster queue (§5) at emission time.
- *ChannelCommitment*: pump the driver each tick (drain the broadcast
  queue → track/CPFP through the exit manager; feed watched-output
  spends — the old branch's justice-loop fix: spends of commitment
  outputs must be fed back or LDK re-broadcasts a spent claim forever;
  feed unconfirmations on reorg and re-ask for the winning commitment).
  **Both sweep families are handled**: on our commitment, `to_local`
  arrives as a delayed `SpendableOutputs` descriptor; on a counterparty
  commitment, our balance arrives as `StaticPaymentOutput`. Sweeps go
  through the output spender with the honest caveat pinned: a
  descriptor whose value cannot cover its fee at the target rate is
  unsweepable standalone — batch it with other descriptors or wallet
  inputs, and accept a documented dust residual below that. The dust
  allowance and the terminal predicate are reconciled explicitly: the
  driver's `pending_claims()` view is the persisted descriptor rows
  **minus** rows marked `abandoned_uneconomical` (marked when the
  value cannot cover its marginal fee and batching cannot rescue it
  across a configured number of ticks; rows are kept for audit, and
  un-abandoned if fee rates fall enough during the exit's life).
  **`pending_claims()` is monitor-backed, not descriptor-backed**: its
  primary source is the live monitors' `get_claimable_balances()` —
  LDK knows of delayed/maturing outputs *before* any
  `SpendableOutputs` descriptor is emitted, so an empty descriptor
  table proves nothing — unioned with persisted descriptors, minus
  the abandoned set. Terminal only when the commitment and every
  known sweep are hash-stably confirmed and that monitor-backed view
  is empty.
- *ChannelSwept*: a **recovering terminal** — re-validates its sweeps'
  block hashes each tick and drops back to `ChannelCommitment` on a
  reorg (contrast with the blind `Claimed`).
- Reorg e2e ports the `7090246a` design: orphan the *commitment's*
  block specifically (the only stably-catchable phase) and assert
  hash-bound recovery to a different block + terminal completion.

**The seam-audit checklist (implementation obligations, enumerated so
the review can tick them):** the `StoredExit` model + `bark_exit_states`
schema + every SQL reconstruction path + the storage adaptors gain and
preserve the payload column; `ExitVtxo` initialization and per-tick
`progress()` reloads go through the source; `start_exit_for_vtxos` /
whole-wallet exits assert store-backed inputs only (channel exits enter
solely via `initiate_channel_exit`); status/listing/cancel/drain load
through the source; the `ExitTransactionManager` tracks the payload's
signed chain + bridge (not `vtxo.transactions()`) for channel entries;
the generic on-chain claim builder's direct load is documented as
safe-by-state (only `Claimable`, which channel exits never reach) with
a debug assertion; the grep rule covers `get_vtxo`, `get_full_vtxo`,
AND `get_wallet_vtxo`, exempting only the wallet-backed
`ExitVtxoSource` impl itself; `ChannelClaimed` integrates everywhere a
state variant must (serialization, `ExitStateKind::ALL` +
`LIVE/FINISHED_STATES`, progress dispatch, `is_exiting`/pending/
confirmation predicates, reconcile, status surfaces, CPFP filters,
bark-json conversions, tests); the `ChannelClose` capture verifies
`channel_id`/`counterparty_node_id`/commitment-txid consistency on
upsert.

**Deadline automation (lifecycle, in-scope).** A maintenance check
(the `RefreshStrategy` slot): when a `ready` channel's
`expiry_height − tip` reaches the floor plus a configured lead, start
the exit (`initiate_force_close` equivalent). Until M5 lands there is
no cooperative alternative — the plan's "the cooperative lead falls
through to force-close" — and M5 replaces the trigger's first rung
with the downgrade.

The server side of a client exit is already shipped and e2e-proven
(MR-3's parent-exit watch responds to the unroll; the mined bridge is
the sanctioned spend). The client e2e re-drives that path from the
production client for the first time.

## 9. What happens to the channel record at terminal states

The record closes on **exit finality** (`ChannelClaimed`, §8 — every
retained block reference `DEEPLY_CONFIRMED` deep), never earlier:
`ChannelSwept` is still live and recovering, and an LDK
`ChannelClosed` event is a **durable signal only** — LDK documents that
closure events can arrive while resolving transactions still await
confirmation, so the event marks the record (`close_observed_at`) but
terminality is always gated on commitment/sweep finality and the
monitor-backed pending view emptying (§8). Two finality clocks are
deliberately distinct: **exit finality** (bark's `DEEPLY_CONFIRMED`,
closes the record and finishes the movement with the exited amounts)
and **monitor archival** (LDK's own, much longer archival signal —
monitors soft-archive when LDK says so, exactly like the server, and
never as a side effect of the record closing). The record itself is
kept forever (history); [[no-unnecessary-backwards-compat]] applies to
shapes, not history.

## 10. e2e plan

Against real captaind + bitcoind + postgres (the barkd suite pattern):
1. **Open from boarded funds** through the production `Wallet` API —
   board, mature, open; assert usable-after-registration and
   not-before (both directions of the gate), pinned record contents,
   SCID banding, movement history.
2. **Restart matrix**: kill/restart at each phase boundary
   (post-establishment, post-cosign/pre-register, post-register/
   pre-feed, post-ready) — the action re-drives to `Done`; reload
   reestablishes and the channel stays usable; double-drive reentrancy
   run.
3. **Full unilateral exit**: genesis → bridge (CSV served) →
   commitment → `to_local` sweep → `ChannelSwept`, against the running
   server whose watch answers the unroll; then the reorg variant
   across the commitment.
4. **Deadline-triggered exit**: shrink the lifetime, let the
   maintenance trigger fire, assert the exit starts before
   `expiry − floor` and completes; server expiry-claim race variant
   (client exits late → server claim wins where designed).
5. **Refusal surface**: depth/runway/drift refusals client-side (no
   RPC fired) and server-side (drive one of each against admission);
   `supports_channels=false` refusal.
6. **Operability probe**: the fail-back HTLC round-trip.

## 11. Commit plan (4 commits, each independently green)

1. **bark: the embedded channel node** — Seams impls (sqlite persist ×2,
   routing broadcaster + durable queue, fee estimator), node lifecycle
   in `WalletInner` + the daemon task, m0043, reload correctness tests
   (crate-level, no server).
2. **server-rpc, server, bark: channel node discovery + retry codes** —
   the three `ArkInfo` fields + `advertise_addresses` config +
   population, the `Code::Aborted` epoch mapping + decode `badarg`,
   openapi regen, client consumption.
3. **bark: the ChannelOpen action** — action variant + module + sync
   arm, channel record (m0044), open-time checks (incl. the exit-fee
   reserve) + config, confirmation feed + SCID allocation, movement
   wiring; e2e 1/2/5/6.
4. **bark: the channel exit** — m0045 + hydration branch, exit-state
   tail + driver trait + LDK driver impl, deadline automation, record
   terminal handling; e2e 3/4 + the reorg case.

## 12. Conformance discharge (client-owned rows)

OP-8..13 (ordering, checkpoints, crash matrix → §4/§10.2);
OP-14..16 client view (§7); BR-3/4/8 client half (pinned storage, §5/§7);
BR-10/11 (bridge persisted + crash-resume exit, §5/§8);
BR-16 (e2e vs real server); PV-8 client refusal (§7);
RG-6..8 client half (not-ready-before-registration, §4/§10.1);
I-1 refresh-first (§4.1); UE-* (the exit story, §8);
IB route-through (unchanged paths re-proven by the suite).

## 13. Decisions taken here (for review)

1. Payments/floor/downgrade/surface split out (§1) — pending Greg's
   ratification of the M4/M5/M6 sequencing.
2. The wire addition rides this MR: node discovery (unusable gap
   today), the advertised-addresses server config (the bound address is
   not dialable), the claim-slack advertisement (local defaults can
   disagree with admission), and the `Code::Aborted` retry mapping.
3. Broadcaster = durable queue drained by the exit driver; nothing
   relays from inside LDK callbacks; `ChannelClose` bump-event payloads
   are captured durably before the handler acknowledges (0.2.4 does not
   redeliver).
4. No `BumpTransactionEventHandler` in M4 (B1 rule); the commitment
   rides the captured event + bark's own CPFP; anchor fees come from
   the wallet, gated by the §7 exit-fee reserve check at open.
5. Secret nonces never persisted; the checkpoint carries the complete
   deterministic plan, so every crash recovers via byte-identical
   package reconstruction + the server's same-backing re-cosign.
6. Client SCID: deterministic bridge-txid derivation, persisted, band
   `2500..2²⁴`, collision-handled — symmetric with the server's
   allocator rules, independently allocated (peer agreement is not
   required; empirically proven in the old branch).
7. Exit-graft shape: the m0029 pattern (nullable `channel_data` on the
   exit table), not a parallel exit subsystem — one machine, TWO
   seams: `ExitVtxoSource` for every VTXO load (grep-checkable: the
   exit crate never calls the store directly) and the LDK driver trait
   for chain actions; plus exactly TWO generic-state branches (the
   `Processing` delta derivation — `ChannelFunding` has no signable
   clause — and the `AwaitingDelta` handoff), which is what keeps
   channel exits out of every generic post-delta surface.
8. Re-entry into establishment inspects the reloaded manager by
   `user_channel_id` and resumes or explicitly closes before any fresh
   attempt — never blind-retry (duplicate-channel hazard).
