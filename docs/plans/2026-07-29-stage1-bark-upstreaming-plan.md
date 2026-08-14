# Stage-1 plan: ARK #8 channels (upgrade/downgrade on stock LDK) for upstream bark

**Status**: v2 — G0 rework applied (codex review at
`2026-07-29-codex-g0-review.md`, verdict REWORK; all 12 findings triaged and
folded in, F7 adopted as wording-fix-plus-documented-posture). 2026-07-29.
**Companion**: `2026-07-29-stage1-conformance-matrix.md` (requirement IDs
`IB/PV/BR/OP/RG/DC/WD/UE/EX/DA-*`, entanglements `E-*`, profile relaxations
`I-*`). The matrix is the acceptance checklist; §7 maps every ID to an owner.

## 1. Goal and non-goals

**Goal**: a fresh branch off `ark-bitcoin/bark` master implementing the stage-1
subset of ARK #8 — the complete channel-VTXO lifecycle (open = upgrade,
close = downgrade, unilateral exit, expiry, watches) with **stock released
LDK** channels on top, and intra-ark payments (client ↔ captaind ↔ client)
working. Minimal diff, upstream style, logically staged MRs, per-commit
buildability, test coverage per requirement ID.

**Non-goals (deferred to later stages)**: teleport/channel-refresh; the Ark
channel type (feature bits 400/401) and the HTLC-success-CSV script change;
any rust-lightning fork; the HTLC forwarding-race *ordering* mechanism (the
success-CSV head start — deferred operator policy, §3.6; note the CLTV
*floor* is NOT deferred, §3.2).

**Reference material, not cherry-picks**: the old branch
`ark-channels-bridge-2026-06-18` (local). Upstream reshaped everything since
its fork point (attestations, watchman, actions framework, lightning
flatten); logic, admission checks, watch designs, and e2e scenarios are mined
from it, code shapes come from master. Per-MR reference commits listed in §5.

## 2. Grounding facts (verified 2026-07-29)

**Upstream master** @ `c5f37986b`:
- Arkoor: `ArkoorBuilder`/`ArkoorPackageBuilder` with checkpoint txs
  (server hardcodes `use_checkpoint=true` today), `vtxos_in_flux` in-process
  lock + one atomic DB write (spent-mark + unregistered-vtxo insert +
  unsigned virtual-tx upsert) **before signing** — this IS the RG-14/RG-15
  reservation. Idempotent retry via `check_spendable_for_oor`.
- Policy: opaque-bytes encoding (zero proto change for a new variant),
  tag `0x08` free — which is exactly the byte the spec pins for
  `channel-funding`. Exhaustive matches are compiler-enforced;
  `decode_vtxo_policy` has a mirror-note for `ServerVtxoPolicy`.
- Registration: `RegisterVtxoTransactions` fills `virtual_transaction.signed_tx`
  (fill-once, idempotent) + flips `Unregistered→Spendable`; spends of
  unregistered VTXOs are refused everywhere (RG-1..5 machinery exists).
- Watching: `sync::ChainEventListener` (block/reorg/mempool fan-out) +
  `watchman/` (per-VTXO `decide_action_*` policy: Wait/Progress/Claim,
  batched claims, CPFP progress; runs embedded or as standalone `watchmand`);
  `VtxoExitFrontier` already implements parent-exit detection + response.
- Client: `WalletAction`/`WalletActionCheckpoint` crash-durable action
  framework (Board/Offboard/LightningSend/Receive); exit subsystem with
  hash-bound `BlockRef` reorg safety and an existing externally-signed-child
  seam (`set_wallet_child_tx`, CPFP provisioning); sqlite migrations `m0041`
  max; REST `api/v1` per-resource routers; movements subsystem pattern.
- Server: composition in `Server::start`; postgres refinery migrations `V53`
  max; `OptionalService<T>` runtime config gating (watchman precedent);
  `server-log` `impl_slog!` + CI precheck requiring a call site; `ln/` is a
  concrete CLN payment backend (names to avoid; no trait to implement).
- Conventions: `<scope>: <verb phrase>` commits; per-commit
  `cargo check --all --tests` in CI; `CHANGELOG/unreleased/<crate>/<MR#>`;
  tabs; no fmt/clippy gate; `just unit` / `just int` (never bare cargo test);
  in-repo agent docs (`contrib/agents/`, incl. database-schema and
  writing-tests skills) to follow. Checked-in generated artifacts that MRs
  MUST update: `server/schema.sql`, `bark/schema.sql`,
  `server/config.default.toml`, `bark-rest/openapi.json` + the generated
  `bark-rest-client` (codex F11).
- In-flight upstream branches adjacent to us: `2026-05-enforcevtxoreg`
  (registration enforcement on arkoor cosign), `lightning-to-ark-lib`
  (lightning structs move into lib), `lightning-checkpoint-arkoor-builder`,
  `htlc-manager`, `2026-07-ldkpathfinding` (client-side LDK router/gossip —
  precedent for deeper LDK usage). Watch for churn; coordinate.

**LDK**: workspace already deps `lightning = "0.2.0"` (locked 0.2.2). Every
0.2.x release (0.2.1+) has `unsafe_manual_funding_transaction_generated`
(+`FundingType::Unchecked`, `Event::FundingTxBroadcastSafe`),
`accept_inbound_channel_from_trusted_peer_0conf`, and
`negotiate_anchor_zero_fee_commitments`. Bump lock to **0.2.4** (0.2.3 fixed
a zero-fee-commitment reserve bug + a ZeroConf features bug). Only
`TransactionType` on `broadcast_transactions` is 0.3.0+ (beta1 out
2026-07-23): on 0.2.x the broadcaster discriminates by "spends a known
virtual funding outpoint". Manual funding is **funder-only** (the client
calls it after `FundingGenerationReady`; the acceptor never does — codex F2).
Isolate all LDK-version-sensitive calls in one shim module so the 0.3.0
migration is one file.

## 3. Stage-1 profile decisions (ratified at G0 rework)

1. **Channel type** = stock `zero_fee_commitments` (TRUC/v3 commitments,
   single shared P2A anchor, no `update_fee`), `static_remote_key`. Server
   policy accepts only this type on Ark-backed channels (I-3 replacement).
2. **The unified floor** (matrix I-4/I-5/I-6, rewritten per codex F1 —
   Critical): `F = channel_max_vtxo_exit_depth + pinned_exit_delta +
   cltv_claim_slack` — every relative timelock on the path plus the unroll
   distance to the actual commitment tx (sequential genesis confirmations =
   unrolling lag, the bridge's CSV, then bridge/commitment/second-stage
   confirmations and fee-bump lag inside the slack config). Applied, with
   the quantifiers the completion review pinned into the spec (08-channels.md
   core, "The force-close deadline"): upgrade-admission runway ≥ `F`; a
   received HTLC checks the **actual receiving channel's** `F`; an invoice
   not bound to one channel advertises the **maximum** `F` across eligible
   receiving channels (keysend covered per-HTLC, no blanket reject); a
   forwarding node requires `incoming_cltv − outgoing_cltv ≥ F_in` (the
   **incoming** scope's floor — the channel whose HTLC it must claim if the
   outgoing leg resolves late); and force-close is triggered **no later than
   the point an unresolved HTLC has exactly `F` remaining**. Checked
   arithmetic, u16-representability where it feeds LDK config.
   Dropping only the CSV-derived *second* `pinned_exit_delta` is verified
   sound; dropping the prefix is a deterministic HTLC-loss vector.
3. **Two-threshold deadline** (codex F4): an earlier **cooperative lead**
   (initiate downgrade; close/HTLC-drain is unbounded so it gets its own
   head start) and the **hard force-close margin** `F` at which the node
   force-closes unless a complete split registration is already potentially
   final (DC-9/EX-4).
4. **Depth reservation at open** (I-9): resulting channel-VTXO depth ≤
   pinned `channel_max_vtxo_exit_depth − 2` (worst-case checkpointed split
   stays cosignable); client prefers `use_checkpoint=false` upgrades (OP-3);
   client re-checks headroom before initiating any close (DC-27/DA-9).
   **Plus** (codex F6): the server refuses a `max_vtxo_exit_depth` config
   *decrease* that would strand any live channel past downgrade eligibility;
   checked `≥ 2`; tested across restart.
5. **Arkoor-sourced input trust** (I-1): client-side default policy refreshes
   third-party arkoor receipts before upgrading them (rides
   `RefreshStrategy`); documented; no server rule (verified sound at G0).
   Self-origin chains (own downgrade outputs) exempt.
6. **Expiry treadmill** (I-2): stage-1 channels are expiry-bounded; the
   client schedules proactive downgrade (+ optional re-upgrade after a round
   refresh) at the cooperative lead. Documented as a stage-1 property.
7. **HTLC-forwarding posture** (decided): forwarding is ON when the
   channels subsystem is enabled, **capped** — floor-derived
   `cltv_expiry_delta`, small `max_htlc_value_in_flight`, per-HTLC size
   caps, and a config kill switch. The deferred stage-2 item is only the
   *ordering* mechanism (the success-CSV head start); the floor (decision
   2) is mandatory now. All stage-1 forwarding is intra-ark by construction
   (the server's node peers only with its own clients); exposure is
   per-HTLC-bounded and documented in the operator guide.
8. **Attestation**: upstream's `ArkoorCosignAttestation` unchanged (OP-29
   parked; tampered-`channel_id` caught at partial-sig verification, OP-28).
9. **Duplicate cosign requests** (codex F7): spec says replay-or-reject for
   byte-identical duplicates; upstream re-signs with fresh nonces —
   cryptographically safe server-side and identical for the bridge slot.
   Stage 1 keeps upstream parity and documents the divergence as a known
   conformance nit (no store-and-replay built).
10. **Keysend** (I-6): allowed, gated by the per-HTLC floor check.
11. **supports_channels** (decided): a plain `bool supports_channels` on
    ark-info — exactly the spec's name (PV-8) — advertised only when the
    server's channels subsystem is enabled (PV-8/PV-9). Stage-2 capability
    signaling arrives later as LN feature bits, not ark-info; floors and
    depth bounds are already published generically.

## 4. Architecture

```
lib (ark-lib)
  vtxo/policy: ChannelFundingVtxoPolicy{user_pubkey}, kind 0x08,
    taproot = keyspend musig(A,S) + single timelock-sign(expiry,S) leaf   [PV-1..5]
    NOT generically arkoor-spendable (input or destination): generic
    builders refuse; upgrade/downgrade get explicit entry points          [PV-6]
  channel (new module): canonical bridge construction + reconstruction
    (TRUC v3, in nSeq=pinned_exit_delta, out0 P2WSH 2-of-2, out1 P2A,
    0-fee) + MuSig2 bridge-cosign helpers shared client/server            [BR-*]

server-rpc (proto)
  ArkoorCosignRequest += channel_id (7), bridge_pub_nonce (8)
  ArkoorCosignResponse += bridge server pub_nonce + partial_sig
  ArkInfo += supports_channels                                            [OP-23..26, PV-8..10]

NEW crate: `bark-channels` — the shared LDK harness (base rust-lightning
  — NOT the ldk-node project, see §4.1)
  node assembly (ChannelManager/ChainMonitor/PeerManager +
  lightning-net-tokio; stock lightning-background-processor for the
  event/persist loop, fit proven in MR-0)
  virtual-funding chain feed (real heights, reorg-coherent: OP-18..22);
    THE GATE (RG-6): ordinary inbound accept with minimum_depth ≥ 1 and
    the funding confirmation FED ONLY after complete Ark registration —
    confirmation-injection is the registration gate; no trusted-0conf, no
    message interception (codex F2); funding-unconfirmation fed only on a
    genuine deep anchor reorg — stock force-close accepted as fail-closed
    (I-10); synthetic SCID position: index derived from the bridge txid
    (24-bit, ≥1) with a persisted per-node collision bump — node-local
    uniqueness + restart stability are the hard requirements; peer
    agreement is NOT required (old branch's sides never agreed and worked;
    aliases carry all wire uses) and the channel is never announced (I-10;
    spec paragraph LANDED 2026-07-30 in the virtual-funding trust bullet)
  broadcaster capture: nothing spending a virtual funding outpoint is
    relayed; the co-op closing tx is committed DURABLY before the capture
    callback returns (LDK hands it over before any persisted event, and
    ChannelClosed does not carry it — codex F9)                           [DC-5, UE-3]
  BumpTransactionEvent handling (codex F5): ChannelClose/HTLCResolution
    events wired to a host WalletSource (external confirmed inputs,
    claim_id reservations, RBF) — zero-fee commitments make this the ONLY
    fee path for force-close and HTLC claims
  LDK-version shim (0.2.x ↔ 0.3.0), persistence backends the two hosts
  implement, open/close orchestration helpers

server/src/channels/  (sibling of ln/, names disjoint from CLN backend)
  OptionalService config [channels] enabled=false default
  postgres V54+ (+ schema.sql + config.default.toml artifacts): channel
    state (channel_id, funding keys, negotiated sats, pinned_exit_delta,
    pinned max depth, backing vtxo, backing_registered_at, close-outcome
    columns), channel_parent_watch (kind upgrade|downgrade, package txids,
    armed/resolved), LDK monitor+manager blobs
  cosign admission hooks in arkoor path (upgrade variant + downgrade split)
  registration-completion → feed funding confirmation (the gate)          [RG-6..7]
  parent-exit + split-response watches as ChainEventListener; expiry sweep
    of channel-funding VTXOs as a watchman policy arm                     [WD-*, EX-1/7]
  max_vtxo_exit_depth config-decrease guard                               [I-9 G0]

bark (client)
  WalletAction ChannelOpen / ChannelClose (checkpointed multi-step,
    PONR durability via the checkpoint blob)                              [OP-10..13, RG-*, DC-31]
  sqlite m0042+ (+ bark/schema.sql artifact): channel record, close
    outcome, downgrade watch
  floor enforcement: invoice final delta, received-HTLC acceptance,
    per-HTLC force-close scheduling                                       [I-6 G0]
  exit extension: new ExitState phases (bridge CSV wait → commitment →
    stock BOLT-3 claims) riding set_wallet_child_tx; BumpTransaction
    WalletSource over the bark onchain wallet                             [UE-*]
  deadline automation: cooperative lead + hard force-close margin         [EX-3/4/8]
  symmetric downgrade watch riding Wallet::sync()                         [WD-8/9]
  movements subsystem CHANNELS; REST /channels (+ openapi + generated
  client regen); CLI `bark channel ...`
```

### 4.1 Why base rust-lightning, not the `ldk-node` project

Investigated 2026-07-29 against ldk-node 0.7.0 (June 2026). ldk-node
packages exactly the decisions stage 1 must make differently, and exposes
no seam to change them:

- **No manual funding**: `open_channel`/`open_announced_channel` only —
  funding is built and broadcast by its mandatory internal BDK wallet. No
  `unsafe_manual_funding_transaction_generated` passthrough. Our funding
  outpoint never exists on-chain. Fatal.
- **Closed chain source**: five presets (esplora / electrum / bitcoind
  RPC / REST), no custom source, no manual `Confirm` feeding — but virtual
  funding *is* a manually fed confirmation at the anchor's real height,
  withdrawn on reorg (OP-18..22). ldk-node would sync the real chain and
  never see the funding. Fatal.
- **No broadcaster seam**: cannot capture-not-relay transactions spending
  a virtual funding outpoint (closing tx capture, commitment handoff to
  the exit driver). Fatal.
- **No manual/trusted-0conf channel acceptance, no zero-fee-commitments
  config exposure**, and **no access to the underlying
  ChannelManager/ChainMonitor** to work around any of the above.
- The one pluggable seam — KVStore via `build_with_store` — doesn't help
  with any of it, and its mandatory internal wallet would sit uselessly
  beside bark's/captaind's own wallets.

Base rust-lightning exposes every needed seam as a trait
(`BroadcasterInterface`, `Confirm`/`Listen`, `Persist`, manual funding,
trusted-peer accept), and bark already depends on the `lightning` crate.
Middle ground adopted instead of hand-rolling everything: the official
companion crates from the same workspace/release line —
`lightning-net-tokio` (peer I/O) and `lightning-background-processor`
(event/persist loop; present in the 0.2.x releases with the async variant,
and it already implements the event-driven manager-persistence pattern the
old branch hand-rolled as a fix). The new crate then carries only the
Ark-specific pieces. Revisit only if ldk-node ever grows external-funding
seams — nothing in its current API or issue tracker suggests that, and it
would not arrive on stage-1's timeline.

Key seam decisions inherited from the old branch (still correct against
master): funding keys are ordinary per-channel LDK keys (LDK never sees
MuSig2) [BR-14/16]; the server correlates cosigns to channels via a
funding-key/channel registry populated at LDK establishment, keyed by the
permanent `channel_id` [OP-23/25]; registration — not LDK's
`is_channel_ready` — is the usability boundary [RG-6], now realized by
confirmation-injection rather than message interception.

## 5. MR sequence

**Granularity decision (G0)**: codex proposed splitting further (→ ~10
MRs); the maintainer constraint is NOT to exhaust upstream with a long
dependent PR stream. Resolution: answer reviewability at the **commit**
layer — upstream CI checks `cargo check` per commit, i.e. they review
commit-by-commit, so each MR below is an internally staged,
per-commit-buildable narrative (the units codex wanted as separate MRs
become commit stages inside one MR). Every MR leaves the tree inert
(feature-gated off) and independently mergeable. After the protocol MR
lands, ask the maintainers their preferred chunking for the remainder and
re-cut if asked — calibrating to the actual reviewer beats theorizing.

**DISPOSITION (revised 2026-07-30): the series is SIX upstream MRs —
`MR-0` (the `bark-channels` release-contract crate) is now the upstream
**opener** and engagement vehicle (a draft MR, description
`2026-07-30-draft-mr-description.md`), planned for posting by Greg;
`MR-1..MR-5` stack after it.** (This supersedes the earlier
design-issue-first / MR-0-stays-local plan; the spec-restructure track is
complete.) The `MR-0/MR-1 = protocol` labels below are unchanged from the
work breakdown; the maintainer-facing draft uses a plain list, not these
internal numbers.

**MR-0 — the release-contract crate (the upstream opener). ✅ its review is
CLOSED; the branch is ready to post as the draft MR.** The eleven enumerated
virtual-funding / fee-bump behaviors of stock lightning 0.2.4 are pinned, so
the releases-only posture is confirmed with no escape hatch needed. (Scope
note: `lightning-background-processor` *fit* is NOT proven here — the suite
uses a hand-rolled pump; the background-processor integration decision moves
to the captaind MR. The "eleven contracts" is the accurate claim, not "every
LDK behavior the feature depends on".) bark-stage1 branch
`ark8-channels-stage1`, two commits: scaffold + release-contract tests
(`tests/release_contract.rs` + `tests/common/mod.rs`). Suite **11/11** (~3s,
per Greg's local run — the codex review sandboxes could not bind the socket
tests) through three codex review cycles (records in this directory): pins
for the funder's empty-witness panic and historical-height virtual
confirmation (depth is computed against the live best block — the upgrade
shape works natively), restart-of-either-side SCID stability with the
fresh-KeysManager-start-time contract, unannounced/alias wire behavior with
an alias-routed payment, and structural+fee assertions on both bump legs
against in-test-known data (derived prevout fees, descriptor-vs-commitment
cross-checks, cryptographic wallet-signature verification, witness-script-hash
binding — full script-interpreter validation deliberately out of scope).
**Final codex verdict: PASS-WITH-CHANGES, residuals applied — the
release-contract review is CLOSED 2026-07-30** (the upstream draft MR itself
is not yet opened) at
scaffold `a79035a9` + tests `ea33bbf4`; four review records in this
directory. The three SCID pins: restart +
identical re-feed keeps the SCID — structurally inert since assignment is
gated on `funding_tx_confirmation_height == 0`; peers feeding different
positions both reach ready and payments route — agreement not required;
same position for two channels = synchronous duplicate-SCID assert.
Reload facts for MR-2: `ChannelManagerReadArgs` never calls `chain::Watch`
— each restored monitor is `watch_channel()`ed after read; reestablish
converges unprompted after restart+reconnect.) Key pins for later MRs: the funder PANICS on a
confirmed funding input with an empty witness (the presigned bridge always
carries one; synthetic tests must too); a hand-rolled event pump MUST call
`process_pending_htlc_forwards()`; zero-fee commitments never reach the
broadcaster standalone — the commitment rides the BumpTransaction package
(the exit driver takes it from the event, not the broadcaster); the co-op
closing tx is handed to the broadcaster independently by BOTH sides (same
txid); use `pending_htlcs[].cltv_expiry` off the bump event rather than
guessing CLTV margins; funding-unconf force-close and same-height distinct
SCID requirements confirmed exactly as I-10 assumed. Original scope
follows (its assertions now live as the harness crate's integration
tests).
Two stock LDK 0.2.4 nodes over lightning-net-tokio, capture-only
broadcaster, manually fed confirmations. Proves, in order:
(a) manual funding **funder-only** (client calls
`unsafe_manual_funding_transaction_generated` after
`FundingGenerationReady`; acceptor ordinary accept with
`minimum_depth ≥ 1`) + zero-fee commitments negotiation;
(b) **the gate**: no `channel_ready`/usability before the funding
confirmation is fed; ready + payable immediately after injection;
(c) payment both directions; cooperative close with durable capture;
(d) **force-close + HTLC resolution** end to end via
`BumpTransactionEvent::{ChannelClose,HTLCResolution}` with an
external-input WalletSource (codex F5);
(e) funding-unconfirmation behavior (expect stock force-close — the
accepted I-10 disposition);
(f) SCID synthesis: two channels fed at the same height with distinct
deterministic tx indices; collision behavior pinned;
(g) `lightning-background-processor` fit (event-handler composition,
persist cadence vs the dedicated-DB-executor requirement).
Any gap here → decide the releases-posture escape hatch (wait for 0.3.0)
before any Ark code is written.

**MR-1 — protocol surface (lib + proto).**
`channel-funding` policy + taproot + encode/decode (+ ServerVtxoPolicy
mirror) + arkoor-compat gates; bridge construction/reconstruction +
bridge-sighash MuSig2 helpers; proto: `channel_id`/`bridge_pub_nonce` on
the cosign part, bridge nonce+partial on the response, ark-info
`supports_channels`. Unit tests: script/taproot vectors, encode round-trip,
unknown-policy rejection (PV-7/9), both-sides bridge determinism (BR-9),
TRUC/P2A shape (PV-11), field optionality (PV-10).
Small, pure, protocol-reviewable — the trust-building first MR; opens the
design conversation with upstream on the wire/policy surface only.
Old-branch refs: B1/B2 bridge builder; `159cc168c` (what NOT to re-add).

**MR-2 — captaind: LDK harness crate + channels subsystem + server open
path.** Theme: *captaind can be the channel counterparty.*
Commit stages: (1) the crate's PRODUCTION harness — assembly, shim, capture
broadcaster, chain feed, BumpTransaction interface — added as `src/`
**alongside the opener's existing tests** (the crate is not recreated; the
MR-0 test-harness helpers inform these production types, and MR-0's suite
must stay green against them); **evaluate and wire
`lightning-background-processor` here** (the fit deferred from MR-0);
(2) captaind subsystem scaffold (`[channels]` OptionalService, V54 blobs +
channel state + schema.sql/config.default.toml artifacts, node lifecycle,
dedicated DB executor, peer listener gated on chain catch-up); (3) upgrade
admission (OP-4/5/25/26, DA-6/7 + reservation, runway floor F, pinned
params, IB route-through, bridge reconstruction + cosign BR-18, rides
`vtxos_in_flux`); (4) the registration gate (confirmation-injection on
complete registration, late-registration refusal RG-7 with not-exited
re-check) + parent-exit watch (ChainEventListener, WD-2..5) + watchman
expiry arm (EX-1/7) + config-decrease guard. **Restart lifecycle (from the
MR-0 review, load-bearing): quiesce and await pumps/transports BEFORE
snapshotting; restore dormant; re-register monitors and chain/funding
state; only then start processing and reconnect** — see cross-cutting notes.
Tests: harness integration (extends MR-0's suite), admission vectors via crafted
requests/proxy (tampered channel_id, amount, depth, runway, bridgeless
destination), gate holds under restart, whole-suite green with the
subsystem disabled AND enabled (PV-10), concurrent-open DB-executor guard.
Old-branch refs: `c9e691ca..211331fe` (persistence arc), `c0bde8f2`
(DbExecutor), `5fb08d12` (listener gate), `0595bb59` (admission set),
`97b05748` (register-after-exit).

**MR-3 — bark client: open + unilateral exit + payments.** Theme: *the
client can open, use, and always recover.*
Commit stages: (1) client LDK embedding (harness host: sqlite persistence,
onchain-wallet WalletSource for BumpTransaction); (2) ChannelOpen action
(establish-first ordering OP-8/10, verify-every-partial OP-12, both-exit-
stories staging OP-11, register, persist bridge + channel record, fresh-
nonce retries OP-27, arkoor-input refresh-first policy I-1); (3) floor
enforcement (invoice final delta, received-HTLC acceptance incl. keysend,
per-HTLC scheduling); (4) exit extension (ExitState phases through bridge
CSV → commitment → stock claims, watched-output spend feeding,
BumpTransaction claims) + deadline automation (cooperative lead → force-
close margin; pre-MR-4 the cooperative lead falls through to force-close);
(5) movements + m0042 + bark/schema.sql.
e2e: open from boarded funds; instant usability post-registration and NOT
before; intra-ark payment A→captaind→B; full unilateral exit
(genesis→bridge→commitment→claims) incl. reorg across the bridge; deadline-
triggered force-close; below-floor HTLC rejected (incl. keysend vector);
open crash matrix (post-cosign/pre-register, post-register/pre-ready).
Old-branch refs: `17f441de` (client flow + eager re-key), `772bcbbf`
(registration release), B6/M1/I-3 exit arc, `7090246a` (reorg e2e),
`6edd047f` (watched-output feed), `0bbd61cc6`/`5b0cd7cdc` (shape pins).

**MR-4 — close by downgrade, end to end.** Theme: *close.*
Commit stages: (1) server close-outcome recording (`ChannelClosed`
cooperative reasons → update-only columns; durable closing-tx capture
before the broadcaster callback returns, DC-5/F9) ; (2) split admission
(sanctioned-only PV-6/DC-28, floor+odd-sat totals DC-16/17, keys ∈ {A,S},
standardness on the reconstructed conflict-winning tx DC-19..23, depth
DA-8, atomic reservation DC-30, split-response watch + armed-on-complete-
registration RG-9, late refusal RG-12/WD-13); (3) client ChannelClose
action (pre-close depth check DC-27, shutdown → outcome record, unfailable-
fee policy DC-12, split build incl. sub-dust isolation + 660-floor refusal,
cosign/verify/register with pending-registration durability RG-11,
old-scope broadcast guard RG-13, closing-tx-as-exit-fallback UE-3);
(4) symmetric watch riding sync keyed on the old chain's FINAL tx
(WD-8/9/11) + cooperative-lead scheduler activation (I-2).
e2e: off-chain settlement (zero mining); odd-satoshi split; sub-dust
isolation; refusal vectors (no recorded close, wrong totals, foreign key,
fragmented dust, sub-660); crash injection between capture and record and
across the PONR; symmetric-watch response both sides; late-registration
refusal after old-chain confirmation; config-decrease guard across restart;
composition loop: open → pay → downgrade → round-refresh outputs →
re-upgrade (depth ladder + pre-close depth refusal + consolidation).
Old-branch refs: `b1a73c5e6`, `e21f5e13e`, `c265b3b01`, `d97e56ec8`,
`487877b56`, `6c954e14f`, `7a45ac225`.

**Addendum (2026-08-09) — the payments slot and its accumulated debts.**
Payments were descoped from the client MR as shipped (both sides refuse
HTLCs; the client MR delivered its stages 1, 2, 4, 5 — LDK embedding,
open, exit, movements — without stage 3). The payments work must be
re-slotted when its design note is cut, and note the ordering
constraint: MR-4's composition e2e (open → pay → downgrade →
re-upgrade) presupposes it. Beyond the original stage-3 list (floor
enforcement, received-HTLC acceptance incl. keysend, per-HTLC
scheduling, the A→captaind→B e2e), the exit and coverage arcs
accumulated debts that come due in the same slot:

- **Terminal accounting generalization**: an output-specific obligation
  count replacing the single-output floor (`expected.max(1)`), and
  node-owned confirmation delivery with a durable drain acknowledgement
  (the general fix for the event-persistence races the ledger + floor
  close today under the one-output invariant).
- **On-chain HTLC resolution**: the preimage scrape is stock LDK
  (`is_resolving_htlc_output` parses claim witnesses inside
  `transactions_confirmed`), but it only sees what the client feeds.
  Needed: (a) a general watched-output confirmed-spend scanner — today
  the exit feeds only self-originated transactions, sound solely
  because no third-party spend can exist without HTLCs; (b) a real
  relay + fee path for OUR HTLC-success/timeout claims — the
  capture-only broadcaster never relays and the `BumpTransaction`
  HTLC-resolution arm fails closed by design; (c) an e2e proving the
  backwards claim: counterparty claims on-chain with the preimage →
  feed → scrape → upstream/inbound HTLC claimed.
- **`Theirs` sweeps become reachable** (StaticPayment on a counterparty
  commitment) once a counterparty commitment can reach the chain; the
  enum and sweep machinery support them, unexercised today.
- The **amount-keyed expectation ledger** assumes one client output per
  commitment; with HTLC outputs the (channel, amount) identity breaks
  and the ledger needs the same output-specific rework as the floor.

**MR-5 — surface + hardening.**
REST `/channels` (+ openapi.json + bark-rest-client regeneration), bark-json
DTOs, CLI subcommands, barkd integration; consolidated adversarial sweep via
`ArkRpcProxy` (registration replay/partial-upload idempotency RG-4, IB
bypass attempts, remaining WD-15 crash matrix); operator docs (stage-1
profile statement, forwarding posture §3.7, WD-1 exit_delta guidance,
expiry-treadmill UX).

## 6. Cross-cutting engineering notes (lessons carried from the old branch)

- LDK sync-context DB writes (monitor persist) MUST NOT share the main pool
  (`block_in_place` starvation deadlock under concurrent opens — proven);
  dedicated executor from day one, concurrent-open e2e guards it.
- Manager persistence: event-driven (`get_and_clear_needs_persistence`)
  with a timer backstop — stock `lightning-background-processor` behavior;
  monitors synchronous+authoritative.
- Crash-consistency tiers: monitors fail the node (UnrecoverableError);
  manager lags safely; reload failure = fail-fast, never fresh-state
  (OP-22/WD-15).
- Restart ordering (MR-0 review, applies to both captaind and bark hosts):
  a node reload MUST quiesce first — stop AND await the event pump, tear
  down and observe the transport disconnected — BEFORE serializing state;
  restore dormant; re-register each monitor (`ChannelManagerReadArgs` never
  calls `chain::Watch`) and re-feed chain/funding state; only then start the
  pump and reconnect. Snapshot-before-quiesce or pump-before-refeed is a bug.
- Registration release must re-check the input hasn't exited (RG-7 was a
  real money bug found by adversarial review).
- Watches arm on COMPLETE registration, correlate by txid-in-package;
  unregistered-resolved rows refuse late arming.
- The gate keys on registration state, never `is_channel_ready` (flips at
  generation, not delivery) — realized as confirmation-injection.
- e2e discipline: board-wait helpers must mine exactly the needed
  confirmations then poll WITHOUT mining (background mining wedged
  near-deadline HTLC choreography); structured slog only (plain `warn!` is
  invisible to the harness log watcher); serial-suite run before declaring
  green (parallel load masked the deadlock once).
- BDK/emitter reorg-wedge is an upstream client bug we should NOT
  reintroduce as a dependency of any assertion: exit-state assertions read
  the chain source, not wallet balance.

## 7. Conformance ownership (ID → owner → mechanism)

Every `MUST`/`MUST*` maps to an owner; codex verifies per MR (G2).
Mechanisms: **T** = test in that MR, **S** = structural (compiler/reuse of
guarded upstream path, noted in MR description), **D** = documented
property/operator docs, **W** = profile waiver.

| IDs | Owner | Mechanism |
|---|---|---|
| IB-1..7 | MR-2 (upgrade path), MR-4 (split path), MR-5 (bypass vectors) | S+T (IB-3 exiting-at-cosign gets its own test) |
| PV-1..5, 7, 11 | MR-1 | T |
| PV-6 | MR-1 (the shared-validator refusal gate, input + destination) + MR-2 (upgrade auth) + MR-4 (split auth) | T (MR-1: reject before mutation on all 3 generic paths) |
| PV-8 | MR-1 (field) + MR-2 (advertise-when-enabled) + MR-3 (client refusal) | T |
| PV-9 | MR-1 | T (pre-channel decoder rejects 0x08) |
| PV-10 | MR-1 (optionality) + MR-2 (suite green, subsystem off/on) | T |
| BR-1..2, 5..7, 9 | MR-1 | T (construction; BR-7 delay-only-on-bridge-nSequence; BR-9 both-sides determinism) |
| BR-3, 4, 8 | MR-1 (construction/schema primitive only) + MR-2 (server runtime: pinned-source/storage/never-reread-live, negotiated-amount equality) + MR-3 (client stores + reuses the pinned value) | T |
| BR-10, 11 | MR-3 | T (exit e2e; bridge persisted + crash-resume) |
| BR-12, 13 | MR-2 | D (MAY: stage-1 server keeps close outcome, not the bridge) |
| BR-14 | MR-2 | S+T (ordinary LDK keys; registry keyed by channel_id) |
| BR-15, 18 | MR-1 (helpers) + MR-2 (cosign integration, equalities) | T |
| BR-16 | MR-0→MR-2 harness test + MR-3 e2e | T |
| BR-17 | MR-2 | T (fresh session nonces) + D (duplicate posture, §3.9) |
| OP-1..7 | MR-2 (server view) + MR-3 (client build) | T |
| OP-8..13 | MR-3 | T (ordering, checkpoints, crash matrix) |
| OP-14..16 | MR-2 + MR-3 | T (floor/depth/pin, both views) |
| OP-18..22 | MR-0→MR-2 harness | T (OP-22 restart e2e in MR-2/MR-3) |
| OP-23..28 | MR-1 (wire shape / schema primitive only) + MR-2 (server runtime: OP-23 identifier lookup, OP-24 at-most-one-part admission, OP-25/26 equalities + tamper vectors) + MR-3 (client runtime: OP-27 fresh nonces on retry) | T |
| OP-29 | — | W (parked; candidate upstream issue) |
| RG-1..5 | MR-2 | S (upstream registration) + T (all-or-nothing, idempotent re-upload, crash) |
| RG-6..8 | MR-2 (gate) + MR-3 (not-ready-before-registration e2e) | T |
| RG-9..13 | MR-4 | T (PONR durability, late refusal, old-scope guard) |
| RG-14..16 | MR-2 (in-flux ride, concurrent test) + MR-4 (split vs round/offboard) | S+T |
| DC-1..14 | MR-4 | T (durable capture + crash injection for DC-5) |
| DC-15..33 | MR-4 | T (odd-sat, isolation, 660 floor, refusal vectors) |
| WD-1 | MR-5 | D |
| WD-2..5 | MR-2 | T (arm + response e2e) |
| WD-6..14 | MR-4 | T (symmetric, both sides) |
| WD-15 | MR-2/3/4 per flow + MR-5 sweep | T |
| WD-16 | MR-2 | S (watchman arm is the only self-initiated broadcast) |
| UE-1..2, 4..7 | MR-3 | T (full exit; UE-6 BumpTransaction fee reserve) |
| UE-3 (closing-tx variant) | MR-4 | T |
| EX-1, 7 | MR-2 | T (watchman expiry arm; whole-output sweep) |
| EX-2..5, 8 | MR-3 (thresholds; cooperative lead activates in MR-4) | T |
| EX-6 | MR-5 | D |
| DA-1..5 | upstream + MR-2 asserts | S |
| DA-6..7 | MR-2 + MR-3 | T |
| DA-8..10 | MR-4 (+ MR-2 config guard) | T |
| Floor F (I-4/5/6) | MR-2 (runway, forwarding delta) + MR-3 (invoice, received, scheduling) | T (boundary cases) |
| I-1 policy | MR-3 | T |
| I-2 scheduler | MR-3 (hard margin) + MR-4 (cooperative lead) | T |
| I-9 amendment / I-10 | MR-2 (guard, SCID, anchor-unconf) + MR-0 proofs | T |

## 8. Process

- **Spec track: ✅ COMPLETE.** `08-channels.md` restructured into layers
  (core lifecycle / the two extensions / implementation profiles),
  E-1..E-17 resolved and I-1..I-10 codified in normative text (incl. the
  unified floor F); passed the two-part codex refactor review
  (PASS-WITH-CHANGES, applied). The stage-1 MRs link the layer boundary.
- **Codex gates**: G0 = done. G1 = per-MR design note (before code) —
  MR-1's ran REWORK → reworked → re-review. G2 = per-MR diff vs the §7
  table. G3 = whole-series review before marking the series ready (not a
  gate on the opener, which is already posting).
- **Explicit go/no-go**: implementation code for any MR starts only on
  Greg's word, after he has seen that MR's final design note + review
  verdicts. A G1 PASS is necessary but not sufficient.
- **Branch mechanics**: branch `ark8-channels-stage1` off `upstream/master`,
  pushed to `origin` (gsanders87/bark); MRs target `ark-bitcoin/bark` per
  their CONTRIBUTING.
- Upstream engagement (decided 2026-07-30): **the opener IS the
  engagement** — post the release-contract branch as a draft MR (superseding
  the earlier design-issue-first plan), then fold its feedback (naming,
  chunking, sequencing vs `enforcevtxoreg` / `lightning-to-ark-lib` /
  `lightning-checkpoint-arkoor-builder` / `htlc-manager`) into the stacked
  MRs.

## 9. Decisions log

All open decisions resolved 2026-07-29 (Greg):

1. **Naming**: crate `bark-channels`, module `server/src/channels/`, CLI
   noun `channel`, REST `/channels`.
2. **Forwarding**: on when the subsystem is enabled, capped (§3.7).
3. **Engagement** (revised 2026-07-30): the opener draft MR *is* the
   engagement (§8) — supersedes the earlier design-issue-first resolution.
4. **ark-info**: `bool supports_channels`, spec-exact name (§3.11).

Resolved at G0 rework: MR granularity (commit-layer staging, re-cut on
maintainer feedback); `cltv_claim_slack` exists as config with a documented
default (a deployment matter — the exact default is proposed in the captaind/
bark MR design notes, not the spec). Revised 2026-07-30: the release-contract
crate is the upstream opener (`MR-0`), not a local spike — six upstream MRs
total (§5).
