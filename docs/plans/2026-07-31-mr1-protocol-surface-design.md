# MR-1 design note (G1): the protocol surface — `channel-funding` policy, the bridge, the wire fields

**Status**: G1 rework (codex REWORK → this revision). 2026-07-31.
**Series position**: MR-1 — the protocol surface, stacking on the upstream
opener (the `bark-channels` release-contract crate). **Baseline**:
bark-stage1 `ark8-channels-stage1` @ `ea33bbf4` (= upstream master + the
opener). **Spec targets**: matrix IDs PV-1..5, PV-7, PV-8(field), PV-9,
PV-10, PV-11(bridge), BR-1..9, BR-14..17(construction), OP-23..24(wire
shape), OP-28(note) — see §5 for the construction-primitive-only scoping.

## 1. Scope and the (narrowed) inertness property

Types, construction, wire shape — **and a structural admission invariant
that makes decoding the new policy safe**. The three G1 review rounds
established that a scattered "reject at validator X" approach is
whack-a-mole: bark has no single input/output admission chokepoint, so each
round surfaced another path that bypassed the last patch (direct arkoor →
Lightning-pay → delegated round participation → round outputs). The fix is
not another patch site but an invariant enforced fail-closed at each
subsystem's *construction* mechanism:

> **A `channel-funding` VTXO may be created or spent only through an
> explicitly authorized construction.** Enforced at the point each
> mechanism is shared by all its callers, defaulting to refusal.

The mechanisms, and where the invariant lands:
- **ArkoorBuilder** (`lib/src/arkoor/mod.rs`) — the constructor gains a
  required `ChannelAuthorization` argument (`{ None, UpgradeOutput,
  DowngradeInput }`) and refuses to build a `channel-funding` *destination*
  unless it is `UpgradeOutput`, or spend a `channel-funding` *input* unless
  it is `DowngradeInput`. Because *every* arkoor caller — direct arkoor,
  Lightning-pay, Lightning-receive-claim, Lightning-revocation — constructs
  through this one builder, this single required-parameter change closes the
  whole arkoor perimeter at the type level: a future new caller cannot
  compile without consciously choosing an authorization, and only two values
  authorize anything. MR-1 passes `None` at every existing call site.
- **Round tree construction** (`server/src/round/mod.rs`) — refuse a
  `channel-funding` round *output* (leaf) and a `channel-funding` round
  *input* (forfeit), enforced in the shared `validate_payment_amounts`
  reached by *both* the self-signed and delegated participation paths.
  Unconditional in stage 1: round issuance/forfeit of a channel VTXO exists
  only under the refresh extension, which is out of scope, so there is no
  stage-1 authorization to grant.
- **Offboard input validation** (`server/src/offboards.rs`) — refuse a
  `channel-funding` input (a channel VTXO reaches an on-chain exit only via
  downgrade-then-offboard, never a direct offboard).
- **Not gated here, deliberately**: the bridge spends the channel VTXO by
  MuSig2 *key path* (`lib/src/channel.rs`, §3), not through any of these
  builders — it is the presigned unilateral path and must stay always-valid.

The honest inertness claim is therefore: **previously valid non-channel
flows are byte/transaction-identical after this MR; every channel-shaped
construction (a `channel-funding` output or input, on any path) is refused
before any DB write, output creation, or spent-state mutation.** The two
authorized entry points arrive later — the upgrade passes `UpgradeOutput`
(MR-2, captaind server), the downgrade passes `DowngradeInput` after
verifying the sanctioned split (MR-4, close). Neither exists yet, so MR-1
refuses universally; the downgrade adding its opt-in is not a carve-out in a
shared refusal but simply supplying a token the builder was always designed
to require. `supports_channels` stays `false`.

Not fully inert, and the note no longer claims otherwise: adding a public
`VtxoPolicy` variant and public proto/struct fields is a source-level API
change (breaks exhaustive matches / struct literals downstream), and
exposing `supports_channels: false` through bark-json changes observable
JSON (protobuf stays byte-identical — `false` is omitted). These are
recorded as API/changelog items, not behavior changes.

Non-goals (later MRs): upgrade/downgrade *authorization* and admission
checks, registration gating, watches, any captaind/bark runtime, the
ark-info advertisement *logic*, the attestation `channel_id` binding
(recorded first-release profile deviation).

## 2. The policy and the admission invariant

### 2a. Policy type (`lib/src/vtxo/policy/mod.rs`, `lib/src/vtxo/mod.rs`)

- `ChannelFundingVtxoPolicy { user_pubkey: PublicKey }` following
  `PubkeyVtxoPolicy`; new `VtxoPolicy::ChannelFunding` variant (a
  *destination* policy guaranteeing `user_pubkey()`), mirrored into
  `ServerVtxoPolicy`'s decode per the mirror note at `vtxo/mod.rs:1084`.
- `VtxoPolicyKind::ChannelFunding`; Display/FromStr `"channel-funding"`.
- Tag `VTXO_POLICY_CHANNEL_FUNDING = 0x08` — next free byte and the
  spec-pinned byte. **Tag-race handling**: an upstream `0x08` allocation
  mid-review REOPENS G1 (spec + matrix + vectors + compatibility fixtures
  all move together), not a silent renumber; raise a tag reservation in
  the draft-MR dialogue.
- **Taproot — DOMAIN-SEPARATED from board funding (decided 2026-08-03; see
  §2d).** Internal key = the untweaked aggregate
  `musig::combine_keys([user_pubkey, server_pubkey])`; the taproot keeps the
  **stock** server-expiry leaf `timelock-sign(expiry, S)` (an unchanged
  `TimelockSign` clause) and has **no user exit leaf** — and adds one
  unspendable domain-marker leaf. A bare `cosign_taproot(musig(A,S), S,
  expiry)` — `taproot(musig(A,S), {timelock-sign(expiry, S)})` — is
  byte-identical to a board funding output; a distinct VTXO type with
  distinct spending rules should not reuse another type's exact output
  construction (§2d). **Mechanism (decided 2026-08-03):** a **second,
  unspendable constant domain-marker tapleaf** →
  `taproot(musig(A,S), {timelock-sign(expiry,S), <marker>})`. The extra leaf
  changes the merkle root → output key, separating it from board's, while
  leaving the *expiry leaf itself* a stock `TimelockSign` clause. This is
  the only mechanism that touches **no shared machinery**: the key-path is
  still a single x-only taproot tweak (`tweaked_key_agg` unchanged), and the
  marker is never a spendable clause — it appears only as a sibling hash in
  the control block, which `vtxo.output_taproot().control_block(..)` computes
  automatically, so the watchman/exit signers and the closed `VtxoClause`
  enum are **untouched** (the whole diff is local to the channel policy's
  `taproot()`/`clauses()`). Cost: +32 witness bytes (the sibling hash) in the
  server's script-path expiry sweep, only on the rare
  actualized-then-abandoned recourse path (the key-path bridge spend is
  unaffected).
  **Rejected alternatives, both because they touch shared machinery:**
  *(i) mutate the expiry leaf* with a `<tag> OP_DROP` prefix — cheaper
  (~+6 wu) and one-leaf, BUT the altered script is no longer a stock
  `TimelockSign` clause, forcing a new/modified variant into the closed
  `VtxoClause` enum and every exhaustive match on it (`watchman/signer.rs`,
  the client exit signer, `policy.clauses()`); *(ii) internal-key ec-tweak*
  — zero on-chain cost, but needs two-tweak signing variants across the
  shared musig helpers, construction, and verification. There is no
  cheaper-than-32 option that avoids touching either the clause enum or the
  musig helpers; the marker leaf is the unique touch-nothing-shared choice.
  Pinned by a byte-**inequality** test vs a board funding output of the same
  keys/expiry (they must differ — the inverse of the old PV-2 claim), plus
  the §2d domain-separation test.

### 2b. The admission invariant (the one behavior this MR adds)

Enforced fail-closed at each subsystem's construction mechanism (§1):

- **ArkoorBuilder** (`lib/src/arkoor/mod.rs`): the constructor(s)
  (`new`/`new_with_checkpoint`/`new_isolate_dust` ~1044-1137 and the
  `ArkoorPackageBuilder`) take a required `ChannelAuthorization` and refuse
  a `channel-funding` destination — normal or isolated, since the builder
  copies supplied policies into both (`arkoor/mod.rs:427,544`) — unless it
  is `UpgradeOutput`, and refuse a `channel-funding` input unless it is
  `DowngradeInput`. This single type-level change covers every arkoor
  caller at once: direct arkoor (`cosign_oor`), Lightning-pay,
  Lightning-receive-claim, and Lightning-revocation all construct here
  (verified: `server/src/arkoor.rs`, `server/src/ln/mod.rs:135,794`,
  revocation). The builder does NOT enforce `is_arkoor_compatible` today
  (`arkoor/mod.rs:1132,1340`) — the required parameter is what forces the
  decision at every call site. MR-1 passes `None` everywhere.
  (G1 history: three rounds of "reject at validator X" each missed a
  sibling path — Lightning-pay bypassing a `cosign_oor`-only check, then the
  delegated round path — which is why the invariant moves into the builder
  itself rather than its callers.)
- **Round** (`server/src/round/mod.rs`): refuse a `channel-funding` output
  (leaf) and input (forfeit) in the shared `validate_payment_amounts`,
  reached by BOTH self-signed and delegated participation (the delegated
  path at ~1836 does only policy-agnostic `check_spendable` and was the
  round-2 bypass). Unconditional in stage 1 — no round authorization exists
  until the refresh extension.
- **Offboard** (`server/src/offboards.rs`): refuse a `channel-funding`
  input.
- **Client** (`bark/src/arkoor.rs`): the same builder parameter applies —
  the client's own downgrade (MR-4) is the one place it will pass
  `DowngradeInput`; MR-1 passes `None`, so the client cannot build a
  channel-shaped arkoor either.

Every refusal is before any DB write, output creation, or spent-state
mutation — and, for receive-claim, before its preimage is durably settled
(`ln/mod.rs:797`).

### 2c. Forced-match inventory (compiler-enforced; each arm is the safe one)

Verified against the tree (G1-F4):
- `VtxoPolicyKind::Display` (`policy/mod.rs:170`).
- The six `VtxoPolicy` methods at `policy/mod.rs:720`: `policy_type`,
  `is_arkoor_compatible` (→ `false`), `arkoor_pubkey` (→ `None`),
  `user_pubkey` (→ the field), `taproot`, `clauses`.
- Encoding (`vtxo/mod.rs:1025`) — the `0x08 || 33-byte key` arm.
- The historical bark migration `m0021_fix_lightning_movements.rs:49` —
  safe arm `false`.
- Round-output construction/telemetry, watchman policy — watchman takes the
  **expiry-claim** arm (spec-correct terminal behavior EX-1; unreachable
  in-tree until a later MR creates such a VTXO, which is why the gate in
  §2b is what actually keeps it unreachable now).
- `FromStr` and the server decoder mirror are wildcard/tag matches (not
  compiler-forced) but semantically required — covered by tests.

### 2d. Domain separation from board funding (the second money-safety property)

The (b) invariant covers the paths that run through a *builder* (arkoor,
round, offboard). Cosignatures over a taproot output, though, are also
produced by flows that don't touch those builders, so type separation
cannot rest on builder gating alone: it must rest on the **output
construction** itself. A bare `cosign_taproot(musig(A,S), S, expiry)` —
`taproot(musig(A,S), {timelock-sign(expiry, S)})` — is the board funding
output's exact construction. `channel-funding` is a distinct VTXO type with
distinct spending rules (spent only via the bridge at
`nSequence=exit_delta`, or the sanctioned downgrade), so reusing another
type's exact output key is a footgun: a cosignature valid for one output
type would be valid for the other. Standard domain-separation hygiene for a
new type is to give it its own, non-colliding construction.

**Requirement**: the `channel-funding` output key MUST differ from a board
funding output's for the same `(A, S, expiry)`. This is a structural,
stateless property, local to the channel-funding construction and
independent of any server-side flow's behavior. Mechanism per §2a (the
domain-marker tapleaf). The channel's own legitimate spends
stay intact: the bridge/downgrade key-path use the channel taproot's tweak,
and the server expiry sweep uses the (still-present) expiry leaf.

The domain marker MUST be a **constant** (e.g. a tagged hash of a fixed
`"ark channel funding"` string), not bound to `channel_id`/`bridge_txid`:
those derive from the funding outpoint, which derives from the
channel-funding scriptPubKey — binding them would be circular.

## 3. The bridge (`lib/src/channel.rs`; precedent `lib/src/board.rs`)

Pure functions of `(channel-VTXO outpoint + amount + taproot spend info,
two BOLT-3 funding pubkeys, pinned_exit_delta)`:

- `bridge_tx(..) -> Transaction`: `nVersion=3`, `nLockTime=0`, one input
  (channel VTXO output, `Sequence::from_height(pinned_exit_delta)`, empty
  witness), out0 = `.to_p2wsh()` of
  `lightning::ln::chan_utils::make_funding_redeemscript(k1, k2)` (canonical
  BOLT-3, key-sorted; lib already deps `lightning` 0.2.4) with value = the
  full VTXO amount (0-fee), out1 = `bitcoin_ext::fee` P2A anchor (zero
  value). BR-1..6, PV-11. (G1 confirmed all of these correct.)
- `bridge_sighash(..) -> TapSighash`: BIP-341 key-path, `SIGHASH_DEFAULT`,
  `Prevouts::All(&[channel_vtxo_txout])`. BR-15/17.
- MuSig2 helpers over `lib/src/musig.rs`, using
  `tweaked_key_agg([A,S], spend_info.tap_tweak())` for the **signing**
  aggregate: nonce pair, partial-sign, partial-verify, and finalize →
  64-byte schnorr key-spend witness.
- Determinism (BR-9): construction is witness-independent, so both sides
  compute the same `bridge_txid` before any signature — the value MR-3
  later derives the permanent `channel_id` and funding outpoint
  (`bridge_txid:0`) from. Documented as a contract; MR-1 adds no
  `channel_id` helper (opaque bytes at this layer).

## 4. The wire (grouped carriers per G1-F3)

Loose `Option` fields let package transformations (`with_vtxo`,
`convert_vtxo`, `set_vtxos` in `lib/src/arkoor/{mod,package}.rs`) silently
drop half a pair. Use grouped carriers instead:

- proto: `ArkoorCosignRequest` += `optional bytes channel_id = 7;`
  `optional bytes bridge_pub_nonce = 8;`; `ArkoorCosignResponse` +=
  `optional bytes bridge_pub_nonce = 3;` `optional bytes
  bridge_partial_sig = 4;`; `ArkInfo` += `bool supports_channels = 21;`
  (bump the `last index` comment). Tags verified free.
- lib carriers: `Option<BridgeCosignRequest { channel_id: [u8;32],
  pub_nonce }>` on the request, `Option<BridgeCosignResponse { pub_nonce,
  partial_sig }>` on the response — one `Option` each, so a half-present
  pair is unrepresentable internally.
- `convert.rs` MUST reject a half-present or malformed protobuf pair
  (not silently `None` it), and the group MUST survive every package
  transformation — tested protobuf → internal → `set_vtxos`/`convert_vtxo`
  → protobuf.
- Attestation (OP-28, reworded per G1-F3-clean): the attestation message
  hashes only input id, output count, per-output amount, and per-output
  policy bytes (`lib/src/attestations.rs:388`) — it provably cannot absorb
  the new request fields. `channel_id` rides deliberately unbound (recorded
  profile deviation); the amount is committed separately and the policy
  bytes commit the channel-funding `user_pubkey`; a tampered `channel_id`
  fails at bridge partial-signature verification (MR-3).

## 5. Tests and the construction-primitive scoping (G1-F5)

These matrix IDs are dischargeable HERE only as **construction/schema
primitives**; their runtime obligations stay owned by the captaind MR
(MR-2, server admission) and the bark MR (MR-3, client), and the plan's §7
table is labeled accordingly:
- BR-3/BR-8 (source/storage/never-reread-live), BR-4 (negotiated-amount
  equality), BR-7 (commitment-input half), BR-14 (key exchange/registry),
  BR-16 (commitment signing), BR-15/BR-17 (once-at-open / session
  freshness / atomicity / retry), OP-23 (identifier lookup), OP-24
  (at-most-one-part admission) — MR-1 pins only the tx/schema shape they
  build on.

Tests (upstream style; unit in-file, vectors in
`lib/src/test_util/vectors.rs`):

| Pin | Test |
|---|---|
| exact `0x08 \|\| 33-byte-key` encoding + decode round-trip + kind strings | policy unit + vector |
| unknown-tag rejection: a **`0xff`** blob (not `0x09` — the likely next allocation) | decode-failure unit |
| taproot shape, no exit leaf, key-role separation (PV-1..4) | scriptpubkey hex vector, fixed keys/expiry |
| **domain separation from board (§2d)**: channel-funding output key ≠ board funding output key for the same `(A,S,expiry)` | byte-**inequality** test (the inverse of the old PV-2 equality); assert a key-path signature over one output key does not validate against the other (distinct sighashes) |
| bridge shape + `verify_tx` (BR-1..6, PV-11) | field asserts + consensus `verify` of the funding-input spend |
| determinism (BR-9) | two constructions byte-identical; pinned txid vector |
| sighash + MuSig2 round-trip (BR-15/17) | sign both halves, aggregate, assert **one 64-byte DEFAULT witness**, schnorr-verify vs the tweaked output key; **reject a corrupted partial**; **funding-key-order independence** |
| **builder invariant — destination** (the new behavior): a `ChannelFunding` destination is refused with `ChannelAuthorization::None` on EACH arkoor caller — direct arkoor, Lightning-pay, Lightning-receive-claim (before its preimage settle), Lightning-revocation — **before any DB/output/spent-state mutation** | server unit/integration, one per caller |
| **builder invariant — input**: a `ChannelFunding` input is refused on each of those callers with `None` — explicitly including the **Lightning-pay input case** (the round-1 bypass) | server unit/integration |
| **round refusal — both sub-paths**: a `ChannelFunding` round output (leaf) and input (forfeit) refused via shared `validate_payment_amounts`, tested on BOTH self-signed AND **delegated** participation (the round-2 bypass) | server unit |
| **offboard refusal**: a `ChannelFunding` offboard input refused | server unit |
| proto optionality + pair preservation (PV-10, OP-23..24 shape) | convert round-trips ±channel fields, and through `set_vtxos`/`convert_vtxo`/`with_vtxo` |
| decoder-compat for `0x08` (PV-9) | PV-9 (a pre-channel decoder rejects unknown tags) is a property of the BASELINE `decode_vtxo_policy` wildcard-error arm (`vtxo/mod.rs:1088`), **structurally verified against `ea33bbf4`** — it cannot be a post-MR-1 test, since MR-1 adds the `0x08` *accept* arm. The MR-1 test is the separate builder-invariant refusal above (a decodable policy still cannot be minted/spent); plus unknown proto fields skipped (prost 0.14 skips-not-preserves) |

Plus whole-workspace `cargo check --all --tests --examples` green, the
opener's release-contract suite untouched and green.

## 6. Commit plan (each builds workspace-wide)

Decoding `0x08` and the admission invariant MUST land in the **same
commit**: the moment the policy is decodable, every construction mechanism
must already refuse it, or the intermediate tree has the very bypass this
MR closes. The `ChannelAuthorization` builder parameter is compiler-forcing
— it touches every arkoor call site atomically — which is the point (a
caller cannot exist without choosing an authorization); the forced-match
policy arms (§2c) land in the same commit for the same reason.

1. `lib, server: the channel-funding VTXO policy and its admission
   invariant` — policy type + **domain-separated taproot (§2a/§2d — NOT
   `cosign_taproot`)** + encoding + the full forced-match inventory (incl.
   m0021) + the `ChannelAuthorization` builder parameter (all callers pass
   `None`) + the round `validate_payment_amounts` refusal (both sub-paths) +
   the offboard input refusal + policy/vector tests + the domain-separation
   inequality + key-nonvalidity tests (§2d) + the before-mutation refusal
   tests per arkoor caller (incl. the Lightning-pay input case) and both
   round sub-paths and offboard.
2. `lib: the presigned bridge transaction and its cosign helpers` —
   `channel.rs` + bridge/sighash/musig + `verify_tx` and the sign tests.
3. `server-rpc, bark: optional channel fields on the arkoor cosign
   exchange; supports_channels in ark info` — proto + grouped carriers +
   convert (half-present rejection) + package-preservation tests + server
   `ArkInfo` construction (still `false`) + bark-json field + its
   field-completeness test. Carries the API/changelog note.

## 7. Open items for review

1. Module name `lib/src/channel.rs` (board.rs precedent argues top-level).
2. Whether the gate should instead defer to the captaind MR (MR-2) —
   decided NO: decoding `0x08` without the gate is unsafe, so the gate
   ships in the same *commit* that makes the policy decodable (commit 1).
3. `optional` proto3 presence vs bare fields — `optional` for explicit
   presence (matches `max_vtxo_amount`).
