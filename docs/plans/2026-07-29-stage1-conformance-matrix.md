# ARK #8 channels — stage-1 conformance matrix

**Purpose**: the normative checklist a stage-1 implementation (upgrade/downgrade
channel-VTXO lifecycle on stock LDK, no teleport, no Ark channel type / HTLC
success CSV) must satisfy, extracted from the spec, plus the places where
stage-1 text is entangled with excluded mechanisms (input to the spec
restructure) and the requirements that are impossible or meaningless without
them (candidates for explicit stage-1 profile relaxation).

**Source**: spec working tree at bookmark `phase4_deletions` (`9f1c7fd`), where
open = upgrade and close = downgrade are the only open/close mechanisms.
Files: `08-channels.md` (repo root), `05-arkoor.md`, `02-vtxo.md`,
`00-overview.md`, `docs/channel-user-stories.md`.

**Excluded from stage 1** (for reference): channel refresh (`08-channels.md`
§"Refresh" ~702–908, §"The teleport protocol" ~909–1109, `leaf_vtxo_cosign`
~1679–1715, round-participation response cap ~1800–1814) and the Ark channel
type / HTLC-success-CSV machinery (§"The Ark channel type" ~238–449).

**Disambiguation used throughout**: the spec uses "refresh" for two things.
(1) **Channel refresh** = round+teleport re-pointing of a channel's backing
VTXO — **excluded**. (2) **Plain VTXO refresh** = ordinary ARK #4 round
refresh of a `pubkey` VTXO that is not (yet, or any longer) a channel's
backing — base Ark, in scope, already implemented upstream.

Tag legend: **MUST**/**SHOULD**/**MAY** = literal RFC-2119 keyword in the
cited text. **MUST\*** = same obligation stated declaratively/structurally
without the keyword. **fact** = load-bearing statement with no obligation of
its own.

---

## PART 1 — STAGE-1 REQUIREMENTS

### 0. Inherited ARK #5 baseline `[IB]` *(added at G0 — codex F8)*

Generic arkoor obligations that BOTH channel entry points (the upgrade
variant and the downgrade split) sit on top of and MUST NOT bypass. They
exist upstream for the generic cosign path; stage-1 conformance owns proving
the NEW entry points route through them (structural check + tests), since an
explicit channel path could otherwise sidestep the generic validators.

- **IB-1 MUST** — The base ARK #5 attestation is verified before any signing. — `05-arkoor.md:141-170`
- **IB-2 MUST** — The input exists, is registered/Spendable, is not banned, and is not already spent or reserved. — `05-arkoor.md:172-191`
- **IB-3 MUST** — An input already **exiting** (its exit chain observed in mempool or chain) is refused at cosign — distinct from the late-*registration* refusals RG-7/RG-12. — `05-arkoor.md:193-200`
- **IB-4 MUST** — Checkpoint-mode parity/uniformity per server policy across the package. — `05-arkoor.md:203-222`
- **IB-5 MUST** — Balance conservation and dust rules hold over the destinations. — `05-arkoor.md:30-48`
- **IB-6 MUST** — Package atomicity over distinct inputs: all parts cosigned or none. — `05-arkoor.md:224-244`
- **IB-7 MUST** — Full VTXO/genesis-chain validation against the confirmed anchor at registration. — `05-arkoor.md:291-300`

### 1. Policy / VTXO shape `[PV]`

- **PV-1 MUST\*** — A channel VTXO's output uses policy `channel-funding` (type byte `0x08`), fields: `user_pubkey` (33 bytes) only. — `02-vtxo.md:149-156`; `08-channels.md:164-166`
- **PV-2 MUST\*** — Internal key = `musig(user_pubkey, server_pubkey)` (cosign taproot `(musig(A,S), S, expiry_height)`, the same construction as a board funding output); this is both the bridge's spend path and the path used to forfeit/refresh/cooperatively-spend the VTXO. — `02-vtxo.md:158-171`; `08-channels.md:167-174`
- **PV-3 MUST\*** — The taproot carries exactly **one** leaf, `timelock-sign(expiry_height, S)`; the `pubkey` policy's `delayed-sign(exit_delta, A)` unilateral-exit leaf is **absent** from `channel-funding` — there is no user VTXO-level exit leaf at all. — `02-vtxo.md:168-178`; `08-channels.md:176-186`
- **PV-4 MUST\*** — `S` is the VTXO's `server_pubkey` field (not restated in the policy); `A` is the policy's `user_pubkey` and is the ordinary cooperative user key — **not** a BOLT-3 funding key. — `02-vtxo.md:159-163`; `08-channels.md:85-86,160-163`
- **PV-5 MUST** — Everything else about a channel VTXO (genesis chain, encoding, decode-time bounds, validation steps 1–5) is exactly ARK #2's generic VTXO machinery, unchanged. — `08-channels.md:188-189`
- **PV-6 MUST** — `channel-funding` is **not** arkoor-spendable at all except through the one sanctioned downgrade split (verified against a recorded close outcome); every other arkoor spend of a `channel-funding` input MUST be rejected. — `02-vtxo.md:106-113`; `05-arkoor.md:181-191`
- **PV-7 MUST\*** — A `channel-funding` VTXO (`0x08`) is rejected by any decoder that predates it (unknown policy-type bytes are rejected) — channel VTXOs are only meaningful between channel-aware parties. — `08-channels.md:1839-1841`
- **PV-8 MUST** — A client MUST NOT attempt a channel open against a server that does not advertise `supports_channels` in ark-info. — `08-channels.md:1836-1838`; `00-overview.md:110`
- **PV-9 MUST\*** — Against a channel-unaware server, an open (upgrade) MUST fail safely: the `channel-funding` destination policy itself is rejected as an unknown policy type, before anything is marked spent (it cannot silently downgrade to a non-channel VTXO). — `08-channels.md:1828-1832`
- **PV-10 MUST** — The `channel_id` + `bridge_pub_nonce` fields added to `arkoor_cosign_request` MUST be optional/ignorable: a peer that does not implement channels MUST see unchanged board/arkoor/offboard flows for non-channel VTXOs. — `08-channels.md:1819-1823`
- **PV-11 MUST** — All exit-chain-family transactions (bridge included) use `nVersion=3` (TRUC, BIP-431) and carry a P2A `OP_1 <0x4e73>` fee-bumping output; a decoder MUST accept VTXO encoding versions 1–2 and reject others. — `00-overview.md:205-208`; `02-vtxo.md:375-376`

### 2. Bridge transaction `[BR]`

- **BR-1 MUST\*** — The bridge is a single presigned transaction: `nVersion=3` (TRUC), `nLockTime=0`, 0-fee, structurally an ARK #6-style exit-chain transaction. — `08-channels.md:193-194`
- **BR-2 MUST\*** — Exactly one input: the channel VTXO output, spent via key path `musig(A,S)`, with BIP-68 relative timelock `nSequence = pinned_exit_delta`. — `08-channels.md:196-198`
- **BR-3 MUST\*** — `pinned_exit_delta` is fixed once, at open, to the input VTXO's decoded `exit_delta` at that moment; it is stored with the channel and is **not** re-read from live ark-info thereafter. — `08-channels.md:198-200,613-618`
- **BR-4 MUST\*** — Output 0 (funding) is a stock segwit-v0 P2WSH 2-of-2 of the channel's two BOLT-3 funding pubkeys, value = channel VTXO value (0-fee); the open cosign MUST verify this value equals the channel's negotiated funding amount. — `08-channels.md:201-204`; restated `08-channels.md:1750-1758`
- **BR-5 MUST\*** — Output 0 carries **no** Ark sweep script (plain BOLT-3 P2WSH only) — the server's expiry recourse sits one hop up on the VTXO output. — `08-channels.md:204-207`
- **BR-6 MUST\*** — Output 1 (anchor) is P2A `OP_1 <0x4e73>`, 0 value, CPFP-bumped at broadcast, exactly as ARK #6 exit transactions. — `08-channels.md:208-209`
- **BR-7 MUST\*** — The unilateral-exit delay lives **only** on the bridge's `nSequence` — no script on the VTXO output, no CSV on the commitment input. — `08-channels.md:211-214`
- **BR-8 MUST** — The delay MUST equal the channel's pinned `exit_delta` — fixed at open from the **input VTXO's decoded `exit_delta`** (itself descended from ark-info `vtxo_exit_delta` at issuance; never re-read live) and reused unchanged for the life of the channel. — `08-channels.md:218-221,613-618` *(the "including by every refresh bridge" clause is channel-refresh; see E-2)*
- **BR-9 MUST** — Both parties MUST construct the identical bridge (same funding output, P2A anchor, `nSequence`) or the funding outpoints differ and the channel cannot operate; the cosign reconstructs it canonically. — `08-channels.md:223-225`
- **BR-10 MUST\*** — The bridge is the last transaction of the channel VTXO's exit chain: unilateral exit = genesis chain root-first, then bridge, then commitment. — `08-channels.md:225-229`
- **BR-11 MUST\*** — The client MUST retain the fully signed bridge — its unilateral-exit path depends on it. — `08-channels.md:229-230`
- **BR-12 MAY** — The server MAY (need not) also retain the bridge; its expiry-sweep recourse needs neither bridge nor commitment. — `08-channels.md:230-234`
- **BR-13 MAY** — A server that does retain the bridge MAY force-close before expiry by broadcasting bridge+commitment (optimization, not required). — `08-channels.md:234-236`
- **BR-14 MUST\*** — The BOLT-3 funding pubkeys are ordinary Lightning keys exchanged in `open_channel`/`accept_channel`, distinct from `A`/`S`; no key pinning is required, and the keys appear in no policy/ark-info field/cosign request — the server holds both, keyed by `channel_id`. — `08-channels.md:88-95`
- **BR-15 MUST** — The bridge's `musig(A,S)` key-path cosignature is collected **once**, in the open cosign, as a BIP-327 MuSig2 partial signature over the BIP-341 key-path (taproot-tweaked) sighash, aggregated per ARK #0's KeySort — **not** per-commitment. — `08-channels.md:688-693`
- **BR-16 MUST\*** — The commitment spends the bridge's funding output through the stock BOLT-3 2-of-2 (ECDSA), with no Ark involvement per update. — `08-channels.md:693-694`
- **BR-17 MUST** — All signatures are BIP-340 Schnorr; all sighashes are BIP-341 with `SIGHASH_DEFAULT` and all prevouts provided; the server's cosign partial is produced first-signer one-shot — a fresh nonce for every **fresh signing session**, never reused across sessions. A byte-identical duplicate request is replayed-or-rejected per ARK #5's operation-identity rule, not given a new session; the bridge slot is atomic with the transfer slots. — `00-overview.md:212-221`; `05-arkoor.md:203-244`; `08-channels.md:688-693,1781-1787`. *Stage-1 posture (G0): upstream re-signs byte-identical duplicates with fresh nonces — cryptographically safe for the server (a fresh-nonce re-sign leaks nothing; the replay-or-reject MUST is defense-in-depth for a client that completes twice). Stage 1 keeps upstream parity and documents this as a known conformance nit rather than building store-and-replay.*
- **BR-18 MUST** — On the upgrade cosign, the server reconstructs the bridge exactly: input = the reconstructed transfer's `channel-funding` output (keyspend `musig(A,S)`, `nSequence=pinned_exit_delta`), out0 = P2WSH 2-of-2 of the funding keys looked up by `channel_id`, out1 = P2A, 0-fee. — `08-channels.md:1731-1736`

### 3. Upgrade / Open `[OP]`

**Origin and destination shape**
- **OP-1 MUST\*** — The channel VTXO comes to exist exactly one way: an **upgrade**, an out-of-round ARK #5 self-spend of a `pubkey` VTXO the user already holds (boarded, round-issued, or arkoor-received). — `08-channels.md:452-461`
- **OP-2 MUST** — A package effecting an upgrade MUST carry exactly one `channel-funding` destination, and it MUST be a **normal** destination (never isolated). — `08-channels.md:520-521`
- **OP-3 SHOULD** — For a single-part package whose destination set is exactly the one `channel-funding` output, the server SHOULD accept `use_checkpoint=false` (a pass-through hop, one exit level shallower); every other shape follows the server's ordinary checkpoint policy. — `08-channels.md:524-530,1776-1779`
- **OP-4 MUST** — The `channel-funding` destination's `user_pubkey` MUST equal the input VTXO's user key (self-spend binding). — `08-channels.md:531-533`
- **OP-5 MUST** — Its amount MUST equal the channel's negotiated funding amount (verified against the bridge's funding-output value at cosign). — `08-channels.md:534-536`
- **OP-6 MUST\*** — The new VTXO inherits the input's `expiry_height`, `server_pubkey`, `exit_delta`, `anchor_point`; the upgrade itself resets none of these. — `08-channels.md:536-539`
- **OP-7 fact** — An upgrade carries no separate fee. — `08-channels.md:539-541`

**Channel setup (Lightning half; type negotiation replaced per profile — see E-1/I-3)**
- **OP-8 MUST\*** — The parties MUST run Lightning channel establishment over the bridge's output (`bridge_txid:0`) as funding outpoint: exchange the two BOLT-3 funding pubkeys (`open_channel`/`accept_channel`) and exchange initial-commitment signatures (`FundingCreated`/`FundingSigned`), a stock ECDSA 2-of-2. — `08-channels.md:483-487,554-563`
- **OP-9 fact** — The commitment is held off-chain and never broadcast unless force-closed; the Lightning stack treats the funding output as confirmed (virtual funding — OP-18..22). — `08-channels.md:487-490`

**Ordering, safety gate, and admission**
- **OP-10 MUST** — Establishment runs **before** the cosign (arkoor txids are witness-independent); **registration is the point of no return** — the user MUST NOT register the signed transfer until it holds (a) the fully cosigned exit story of the channel VTXO, (b) the cosigned bridge, and (c) a valid initial commitment. — `08-channels.md:543-551`
- **OP-11 MUST** — At every step the user MUST hold at least one of two complete exit stories (the input's own, or the channel's); (ii) MUST be verified complete before registration surrenders (i). — `08-channels.md:577-584`
- **OP-12 MUST** — The client MUST NOT complete or register unless **every** partial signature verifies, the bridge's included. — `08-channels.md:585-589`
- **OP-13 fact** — A post-cosign abort leaves the input marked spent on the server, with unilateral exit of the input as recovery; an abort at or before the "initial commitment held" gate costs nothing. — `08-channels.md:589-591`
- **OP-14 MUST** — Each party enforces admission from its own view: the client MUST refuse to build, and the server MUST refuse to cosign, an upgrade violating Depth (DA-6/DA-7), Runway, or Pinned-parameters. — `08-channels.md:593-596`
- **OP-15 MUST** — **Runway**: the remaining runway (`expiry_height − real chain tip`) MUST exceed the party's computed complete CLTV floor. — `08-channels.md:603-612` *(floor formula is excluded-derived — see E-8, I-4/I-5 for the stage-1 substitute)*
- **OP-16 MUST** — **Pinned parameters**: `pinned_exit_delta` = the input VTXO's decoded `exit_delta`; a server unwilling to operate at that value MUST refuse to cosign (remedy: plain-VTXO refresh of the input first, then upgrade). `channel_max_vtxo_exit_depth` is pinned from the published `max_vtxo_exit_depth` as of the open. — `08-channels.md:613-620`
- **OP-17 fact** — An upgrade adds no arkoor double-sign trust of its own — only the holder can request spends of its own input. — `08-channels.md:669-671`

**Virtual funding / chain-view consistency**
- **OP-18 MUST\*** — Virtual funding = treating the funding output as confirmed once the VTXO's chain anchor confirms; presented as confirmed at the anchor's **actual** best-chain height. — `08-channels.md:106-115`
- **OP-19 MUST** — An implementation MUST NOT advance its best-chain view beyond the real best chain to manufacture virtual depth. — `08-channels.md:115-116`
- **OP-20 MUST** — Every height-dependent decision MUST observe **one** consistent, real-chain history, including reorgs and unconfirmations in order. — `08-channels.md:117-121`
- **OP-21 MUST\*** — An anchor disconnection withdraws the virtual confirmation, suspending normal channel operation until a valid anchor is re-established. — `08-channels.md:121-123`
- **OP-22 MUST** — A restart MUST NOT weaken any of this: a node MUST NOT accept or forward HTLCs on a channel until it again holds a coherent chain view and funding status for it, and MUST leave the channel fail-closed if it cannot recover them. — `08-channels.md:123-126`

**Messages — `arkoor_cosign_request` channel (upgrade) variant**
- **OP-23 MUST** — `channel_id` (32B) on the part is the channel's **permanent** BOLT-2 id (funding outpoint assigned before this cosign); it MUST match the identifier under which the server stored the channel's funding keys. The part also carries `bridge_pub_nonce` (the user's MuSig2 public nonce for the bridge key-path sighash); the response is extended with the server's bridge `pub_nonce` + `partial_sig`. — `08-channels.md:1726-1739`
- **OP-24 MUST** — A package MUST carry **at most one** part with the channel fields (exactly one `channel-funding` destination). — `08-channels.md:1720-1723`
- **OP-25 MUST** — When `channel_id` is present, the named channel MUST be one the server is opening (funding keys exchanged, not yet operating, no prior funding registered, `ChannelReady` never sent) and its establishment run against exactly the funding output this cosign produces: (i) the channel's assigned funding outpoint MUST equal the reconstructed transfer's `bridge_txid:0`, and (ii) the reconstructed bridge's output 0 MUST equal the channel's negotiated funding output (value = `open_channel.funding_satoshis`; script = P2WSH 2-of-2 of the stored funding keys). If `channel_id` names no such channel, or either equality fails, the server MUST NOT cosign **any** part of the package. — `08-channels.md:1746-1766`
- **OP-26 MUST** — The server MUST NOT cosign a `channel-funding` destination except together with its bridge in the same exchange (no-bridgeless-VTXO rule); a request with a `channel-funding` destination and no `channel_id` MUST be rejected. — `08-channels.md:1767-1772`
- **OP-27 MUST** — On any retry, the user MUST discard and regenerate its secret nonces for **every** slot, the bridge's included; `channel_id` joins the part's ARK #5 operation identity. — `08-channels.md:1781-1787`
- **OP-28 MUST** — The attestation is the unchanged ARK #5 attestation (destination policy bytes already commit `channel-funding`'s `user_pubkey`/amount); a tampered `channel_id` yields a bridge over the wrong funding keys, caught at partial-signature verification. — `08-channels.md:1738-1742`
- **OP-29 MUST\*** — The attestation's operation-identity hash binds `channel_id` as its own distinct extension field. — `05-arkoor.md:157-170` *(PARKED for stage 1 by decision 2026-07-29: upstream's attestation is used as-is; `channel_id` rides unattested with `use_checkpoint` precedent; low-severity MITM griefing accepted, candidate upstream issue.)*

### 4. Registration & point-of-no-return `[RG]`

**Generic ARK #5 registration**
- **RG-1 MUST** — After completing signatures, the sender MUST upload the signed virtual transactions (at minimum the checkpoint) to the server. — `05-arkoor.md:252-255`
- **RG-2 fact** — An unregistered VTXO is not accepted as input to a later transfer, and the server can only defend a recipient's exit by broadcasting the checkpoint if it holds that signed transaction. — `05-arkoor.md:256-263`
- **RG-3 MUST** — For a flow that treats registration as its **point of no return** (both upgrade and downgrade), "registration" MUST mean the **complete** set of levels the duty depends on, not merely the checkpoint. The server MUST NOT treat the operation as registered, arm its duty, or release a channel until it durably holds all of them. — `05-arkoor.md:266-278`
- **RG-4 MUST** — The server MUST apply such a registration as a single all-or-nothing transition over the required levels, and MUST accept a byte-identical re-upload idempotently; a partial upload leaves the operation unregistered. — `05-arkoor.md:279-285`
- **RG-5 MUST** — The registered set MUST survive a crash: registration is recorded before the duty is reported armed. — `05-arkoor.md:286-289`

**Upgrade-specific PONR**
- **RG-6 MUST** — The server MUST NOT send `ChannelReady` before it durably holds the fully signed transactions of the new level(s) — ARK #5 registration completing is the trigger; until then the channel MUST NOT operate. — `08-channels.md:636-639`
- **RG-7 MUST** — Late-registration refusal: a registration arriving after the input's exit-chain final transaction has confirmed (watch resolved with nothing broadcast) MUST be refused rather than completed. — `08-channels.md:640-646`
- **RG-8 fact** — Registration arms the server's parent-exit defense: the registered transfer spends the input by key path at `nSequence=0`, mineable the block after the input output confirms — `exit_delta − 1` blocks ahead of the leaf's claim. — `08-channels.md:622-633`; `00-overview.md:115-132`

**Downgrade-specific PONR**
- **RG-9 MUST** — "Registration" for the downgrade means the **complete split** (every arkoor level of both `pubkey(A)` and `pubkey(S)`); the server MUST NOT treat a downgrade as registered, arm its response, or consider settlement final until it durably holds the whole signed chain. — `08-channels.md:1447-1452`
- **RG-10 MUST\*** — Before registration is eligible to have been sent, a failed exchange leaves a closed channel and an unresolved VTXO: an implementation supports retrying the cosign under ARK #5's operation-identity rule, or falling back to the unilateral close path before the force-close deadline (stated declaratively in the cited text). — `08-channels.md:1454-1459`
- **RG-11 MUST** — From the moment registration is eligible to have been sent, and across any restart, the user MUST NOT broadcast the old scope — recovery is by replaying the idempotent registration, not the fallback. — `08-channels.md:1460-1465`
- **RG-12 MUST** — The server MUST refuse the split's registration once the old chain's final exit transaction has confirmed unregistered — even though the bridge's `exit_delta` has not yet elapsed. — `08-channels.md:1519-1524,1531-1534`
- **RG-13 MUST NOT** — Past the downgrade's point of no return, the old scope (commitment/closing transaction) MUST NOT be broadcast. — `08-channels.md:1253-1254`

**Cross-flow reservation**
- **RG-14 MUST** — **Spent-state is a single atomic reservation**: every server-mediated consumer of a VTXO — offboard (ARK #7), arkoor cosign including the downgrade split (ARK #5), and round participation (ARK #4) — MUST check-and-set **one** atomic per-VTXO spent-or-reserved state; independent per-flow locks are non-conforming. — `08-channels.md:1408-1416`
- **RG-15 MUST** — Under that reservation, the split's spent-mark MUST be written before signing and is never unwound. — `08-channels.md:1421-1424`
- **RG-16 fact** — Registration via the ordinary ARK #5 registration endpoint is the irrevocable point of no return for both upgrade (Sequence step 7) and downgrade (Sequence step 5). — `08-channels.md:569-571,1553-1554`

### 5. Downgrade / Close `[DC]`

**The close (BOLT-2 half)**
- **DC-1 fact** — The close is the ordinary BOLT-2 flow: either peer MAY initiate `shutdown`; once empty of HTLCs, closing negotiation produces a fully signed closing transaction. `stfu` plays no part. — `08-channels.md:1125-1140`
- **DC-2 MUST** — The user MUST complete the close (hold a fully signed closing transaction) **before** the split's cosign request — exit-before-forfeit discipline. — `08-channels.md:1179-1183`
- **DC-3 MUST NOT** — The server MUST NOT cosign a split of a channel VTXO whose channel has not completed the close. — `08-channels.md:1184-1186`
- **DC-4 MUST** — The server MUST retain the close outcome (final balances, keyed to the exact backing VTXO) until the VTXO is resolved (split or expiry-swept). — `08-channels.md:1186-1190`
- **DC-5 MUST** — The close outcome MUST survive anything after close completes (crash, restart, however routed) until the backing VTXO is resolved; the user additionally retains the fully signed closing transaction from the moment it exists. — `08-channels.md:1191-1197`
- **DC-6 MUST** — Every driver of the settlement MUST see the same recorded outcome — beginning the Ark leg MUST NOT depend on having personally observed the close complete. — `08-channels.md:1197-1199`
- **DC-7 MUST** — A party that cannot record the outcome MUST NOT report the close complete or begin the Ark leg, and MUST keep the channel fail-closed. — `08-channels.md:1200-1202`
- **DC-8 MUST NOT** — A completed cooperative close is not a force-close: automatic recovery MUST NOT react to one by starting the unilateral exit or discarding the close outcome. — `08-channels.md:1203-1208`
- **DC-9 MUST** — Until the split is registered, the client MUST hold the closed channel/backing scope to the **same** force-close deadline as an operating channel; if no settlement is potentially final by that deadline, it MUST begin the unilateral fallback in time to confirm the bridge before expiry. — `08-channels.md:1209-1215`
- **DC-10 fact** — The BOLT-2 closing fee never enters the settlement (verified against pre-fee balances); the user SHOULD negotiate it low. — `08-channels.md:1216-1221`
- **DC-11 MUST** — The unilateral fallback MUST be able to confirm at current feerates: closing tx + fee-paying child spending the user's shutdown output, or the latest commitment via its anchor/package path. — `08-channels.md:1222-1227`
- **DC-12 SHOULD** — Fee negotiation made unfailable by policy: the non-funder (server) SHOULD accept any fee at or above its relay floor; the funder SHOULD prefer conceding upward; `option_simple_close` preferred once available. — `08-channels.md:1228-1243`
- **DC-13 fact** — The close is a one-way door; before the split's cosign request has been sent, a settlement failure leaves the channel closed but loses nothing. — `08-channels.md:1245-1250`
- **DC-14 MUST** — The close does not extend expiry: while no settlement is completing, the client MUST begin fallback in time to confirm the bridge before expiry. — `08-channels.md:1250-1252`

**The split (ARK #5 shape)** — a plain ARK #5 request, **no new fields** (`08-channels.md:1789-1798`)
- **DC-15 MUST** — Single-part package; input = the closed channel's backing VTXO. — `08-channels.md:1291-1294`
- **DC-16 MUST** — Every destination MUST carry the `pubkey` policy with `user_pubkey` = `A` or `S`, and per-key totals MUST equal the close-fixed final balances (pre-fee). — `08-channels.md:1297-1303`
- **DC-17 MUST** — Sub-satoshi remainder: per-side floor; the odd satoshi goes to the user's `pubkey(A)` total (destinations sum to `V` exactly); the server MUST verify on the same floor-plus-remainder basis. A side that floors to zero gets no output. — `08-channels.md:1303-1316`
- **DC-18 fact** — The rule is over per-key **totals**, so ARK #5 dust isolation composes with it. — `08-channels.md:1316-1329`
- **DC-19 MUST** — The split's conflict-winning transaction (checkpoint, or arkoor tx when `use_checkpoint=false`) MUST be standard: beyond the single P2A anchor, no output below `P2TR_DUST` — verified by the server on the reconstructed conflict-winning transaction directly. — `08-channels.md:1330-1337`
- **DC-20 MUST NOT** — ARK #5's permissive "MAY mix without isolation" does **not** apply to a downgrade. — `08-channels.md:1338-1339`
- **DC-21 MUST NOT** — No fragmenting a side into several same-key destinations to force multiple sub-dust outputs into the conflict-winning transaction (the isolation-lender fragment is the sole exception). — `08-channels.md:1340-1344`
- **DC-22 MUST** — When one side's balance `d < P2TR_DUST`, the split MUST isolate it (lender-fragment shape); the checkpoint `[V−D, D, P2A]` is standard iff `V ≥ 2·P2TR_DUST` (660 sats). — `08-channels.md:1346-1354`
- **DC-23 MUST** — A downgrade of a channel below 660 sats total with a sub-dust side has no standard conflict-winning transaction; the server MUST refuse it. Normative floor. — `08-channels.md:1354-1360`
- **DC-24 fact** — No forfeit in a downgrade: the split spending the channel VTXO by key path at `nSequence=0`, ahead of the retired bridge's `exit_delta`, plays the forfeit's role. — `08-channels.md:1361-1367`
- **DC-25 fact** — No fee. — `08-channels.md:1368-1370`
- **DC-26 fact** — The new `pubkey` VTXOs inherit `expiry_height`, `exit_delta`, anchor, at one (no-checkpoint) or two (checkpointed) exit levels deeper; refreshed via ordinary plain-VTXO ARK #4 refresh. — `08-channels.md:1371-1374`
- **DC-27 MUST** — Depth headroom MUST be checked **before initiating the close** (one-way door; DA-9). — `08-channels.md:1379-1383`

**Admission**
- **DC-28 MUST NOT** — The server MUST NOT cosign any arkoor spend of a `channel-funding` input except the sanctioned split, verified against the close outcome recorded at `ChannelClosed`, bound to exactly this backing VTXO. — `08-channels.md:1396-1401,1793-1797`
- **DC-29 MUST** — Each party enforces from its own view: refuse on no recorded completed close, destination key outside `{A,S}`, per-key total mismatch, or a non-standard conflict-winning transaction. — `08-channels.md:1401-1406`
- **DC-30 MUST** — RG-14's atomic spent-state reservation applies identically to the split. — `08-channels.md:1408-1416`

**Sequence**
- **DC-31 MUST** — (1) BOLT `shutdown` → closing negotiation → record close outcome; (2) build the split [LAST FULLY-FREE ABORT]; (3) `arkoor_cosign_request` (single part) — server verifies against recorded close outcome, marks input spent, returns partials; (4) verify all partials + complete; (5) register [**POINT OF NO RETURN**]; (6) both retain split transactions and watch the old chain. — `08-channels.md:1538-1558`
- **DC-32 fact** — Open = upgrade, close = downgrade (+ generic ARK #7 offboard when on-chain exit is wanted). — `08-channels.md:1566-1571`
- **DC-33 fact** — A downgrade never surrenders the VTXO uncompensated — the split's outputs restate the user's balance in the same signed package that spends it. — `08-channels.md:1852-1856`

### 6. Watch & response duties `[WD]`

- **WD-1 MUST** — A server MUST publish `vtxo_exit_delta` comfortably larger than its worst-case respond-and-confirm time. — `00-overview.md:130-132`

**Upgrade: the parent-exit response** (armed at registration; RG-6/RG-7)
- **WD-2 MUST** — The server MUST retain the new level(s)' signed transactions and MUST watch for any prefix of the input's exit chain confirming, until the input VTXO's output is conclusively spent on-chain (by the retained transfer), or a confirmed expiry sweep of an ancestor forecloses its creation. — `08-channels.md:647-654`
- **WD-3 MUST** — On seeing the input's chain confirm, the server MUST broadcast the retained transaction(s), fee-bumped via the P2A anchor, ahead of the input's `exit_delta` window. — `08-channels.md:660-661`
- **WD-4 fact** — The response actualizes only part of the channel's exit chain; bridge and commitment are unaffected; the server's expiry-sweep recourse is preserved. — `08-channels.md:664-667`
- **WD-5 fact** — This is the forfeit-watch pattern applied to an open, load-bearing for the channel balance. — `08-channels.md:662-663,1857-1866`

**Downgrade: the split response** (symmetric — held by **both** parties)
- **WD-6 MUST** — The server MUST retain the fully signed split transactions from registration, and MUST watch for any prefix of the channel VTXO's exit chain confirming, for as long as the retired bridge remains confirmable. Onward movement of the split outputs does **not** end the duty. — `08-channels.md:1475-1482`
- **WD-7 MUST** — On seeing the chain confirm, the server MUST broadcast the retained checkpoint (or arkoor) transaction, fee-bumped via its P2A anchor, ahead of the bridge's `exit_delta` window. — `08-channels.md:1482-1484`
- **WD-8 MUST** — The user MUST keep the same watch for its own share — the server's expiry-sweep leaf takes the **whole** output, so a server that actualized the old chain and sat out the clock would sweep both balances. — `08-channels.md:1485-1489`
- **WD-9 fact** — The user's response is the same broadcast (its split transactions are its new VTXOs' genesis levels); the duty is scoped to the user's own share and ends when no split output remains its own. — `08-channels.md:1489-1498`
- **WD-10 fact** — The response actualizes only part of the new VTXOs' genesis chains; the server's expiry-sweep recourse over the actualized outputs is preserved. — `08-channels.md:1499-1502`
- **WD-11 MUST\*** — **Where the race is decided**: the contested event is confirmation of the exit chain's **final** transaction (the one creating the channel VTXO's output and starting the bridge's `exit_delta` clock) — a confirmed prefix sharpens the watch but decides nothing and MUST NOT be treated as the fallback in progress. — `08-channels.md:1504-1515`
- **WD-12 fact** — Registered first: the armed response wins by construction (`nSequence=0`, `exit_delta − 1` blocks ahead of the bridge). — `08-channels.md:1515-1519`
- **WD-13 MUST** — Confirmed first, split unregistered: the settlement is the unilateral fallback already in progress, and the server MUST refuse the split's registration from then on (RG-12). — `08-channels.md:1519-1531`
- **WD-14 fact** — This refusal is the downgrade's counterpart of the upgrade's late-registration refusal (RG-7). — `08-channels.md:1531-1534`
- **WD-15 fact** — Neither side's crash creates a window for the other: safety-critical state is stated as observable properties that MUST survive a crash; a party that cannot recover fails **closed**. — `docs/channel-user-stories.md:182-186`; `08-channels.md:634,1473`
- **WD-16 fact** — The server never unrolls a tree on its own initiative; its only self-initiated on-chain act is the post-expiry sweep. — `docs/channel-user-stories.md:149-152`; `08-channels.md:1605-1614`

### 7. Unilateral exit `[UE]`

- **UE-1 MUST\*** — Step 1: exit the VTXO (ARK #6) — broadcast the genesis chain root-first until the `channel-funding` output confirms (TRUC v3, P2A/CPFP, each level after its parent). — `08-channels.md:1581-1584`
- **UE-2 MUST\*** — Step 2: broadcast the bridge — valid only `pinned_exit_delta` blocks after the VTXO output confirms; itself TRUC v3, P2A/CPFP. — `08-channels.md:1585-1590`
- **UE-3 MUST\*** — Step 3: force-close — broadcast the latest commitment (no extra delay of its own). After a completed cooperative close whose settlement never resolved, spend the funding output with the **closing transaction** instead (no `to_self_delay`, no HTLCs). — `08-channels.md:1591-1597`
- **UE-4 fact** — Step 4: claim commitment outputs per stock BOLT-3 rules. — `08-channels.md:1598-1600`
- **UE-5 fact** — No VTXO-level `delayed-sign` claim exists: the channel VTXO's value is claimed only through the bridge and then the commitment. — `08-channels.md:1602-1604`
- **UE-6 MUST\*** — The user needs spendable on-chain funds to CPFP the full exit chain (genesis levels, bridge, commitment, HTLC second-stage) within the force-close deadline. — `08-channels.md:136-142`
- **UE-7 fact** — Genesis chain + bridge + commitment (or closing tx) = complete unilateral recovery without server cooperation. — `08-channels.md:1845-1850`

### 8. Expiry `[EX]`

- **EX-1 MUST\*** — The `timelock-sign(expiry_height, S)` leaf lets the server sweep the channel VTXO output directly after `expiry_height` — its recourse when the user actualized the output on-chain but left the channel unresolved. — `08-channels.md:1607-1612`
- **EX-2 MUST** — The user MUST get the bridge confirmed **before** `expiry_height` (hard deadline). Before the bridge confirms, the **whole** channel VTXO is exposed to the sweep. — `08-channels.md:1617-1623`
- **EX-3 MUST** — The user MUST begin a force-close at least `channel_max_vtxo_exit_depth + pinned_exit_delta` blocks — plus confirmation and fee-bumping slack — before `expiry_height`; the deadline is the **bridge's** confirmation, not the commitment's. — `08-channels.md:1625-1635`
- **EX-4 MUST** — The user MUST stop offering new HTLCs and resolve the channel ahead of expiry on **two thresholds** (G0 amendment, codex F4): an earlier **cooperative lead** at which a downgrade is initiated — close negotiation and HTLC drain are unbounded in time, so the cooperative path gets its own separately-derived head start — and the hard **force-close margin** of EX-3, at which the node MUST force-close unless a complete split registration is already potentially final (DC-9). Waiting for a downgrade at the hard margin can consume the bridge-confirmation budget and lose the whole VTXO. — `08-channels.md:1209-1215,1245-1254,1645-1652` *(the spec's "larger of (a)/(b)" term (b) is CSV-derived — see I-4 for the stage-1 floor)*
- **EX-5 MUST** — Absent a responsive peer near the deadline, the node MUST force-close unilaterally. — `08-channels.md:1642-1645`
- **EX-6 fact** — The deadline is the user's own discipline: the server's expiry-sweep leaf **gains** the whole channel on a missed deadline, so the user cannot rely on the server to stop forwarding near expiry. — `08-channels.md:1660-1666`
- **EX-7 MUST\*** — The sweep takes the **whole** channel (single `musig(A,S)` output, expiry leaf is the server's alone); a channel carried past expiry loses the user's **entire** balance. — `08-channels.md:1883-1890`
- **EX-8 MUST** — The user's only protection is the deadline: force-close (actualizing its balance into the commitment's `to_local`) well before expiry, with margin to confirm the bridge first. — `08-channels.md:1890-1897`

### 9. Depth accounting `[DA]`

- **DA-1 def** — Exit depth = number of genesis items in a VTXO's chain. — `02-vtxo.md:28-29`
- **DA-2 MUST** — A server publishes `max_vtxo_exit_depth` and MUST refuse to cosign a transition spending an input whose exit depth has already reached it. — `02-vtxo.md:29-35`; `05-arkoor.md:10-12,198`
- **DA-3 SHOULD** — Clients SHOULD refresh (plain-VTXO) well before reaching the depth bound. — `02-vtxo.md:35-36`
- **DA-4 MUST** — Recipient of an arkoor-created VTXO SHOULD verify depth room; MUST refresh or exit before `expiry_height`. — `05-arkoor.md:326-329`
- **DA-5 SHOULD** — Servers SHOULD bound destinations per input (reference default 4). — `05-arkoor.md:338-343`
- **DA-6 MUST** — **Upgrade admission — Depth**: the resulting channel VTXO's exit depth (input's + 1, or + 2 through a checkpoint) MUST be at most the `channel_max_vtxo_exit_depth` being pinned at open. This **tightens** plain ARK #5 (which bounds only the input). — `08-channels.md:597-601`
- **DA-7 MUST** — `channel_max_vtxo_exit_depth` is pinned from the published `max_vtxo_exit_depth` as of the open; not re-read for an existing channel. — `08-channels.md:613,619-620`
- **DA-8 MUST\*** — **Downgrade eligibility**: a no-checkpoint split needs channel-VTXO depth ≤ `max_vtxo_exit_depth − 1`; a checkpointed split needs depth ≤ `max_vtxo_exit_depth − 2`. — `08-channels.md:1375-1379`
- **DA-9 MUST** — Depth headroom checked **before initiating the close**. — `08-channels.md:1379-1384` *(the refresh-then-downgrade remedy is excluded — see I-9)*
- **DA-10 fact** — Split outputs may legitimately land at `max_vtxo_exit_depth`; such a VTXO cannot be arkoor-spent again until a plain round refresh resets its depth — ordinary deep-VTXO discipline. — `08-channels.md:1384-1391`

---

## PART 2 — ENTANGLEMENT POINTS (inputs to the spec restructure)

Stage-1-included text whose wording derives from, cross-references, or is
justified by an excluded mechanism. Quotes verbatim.

- **E-1** — "Channel setup" (in-scope) is textually defined via the Ark channel type (excluded): "The parties MUST negotiate the Ark channel type…" (`08-channels.md:482-483`); "…the parties MUST negotiate a dedicated Ark channel type" (`:240-241`); "The server MUST refuse to open a channel of any other type…" (`:250-252`, direct conflict — see I-3).
- **E-2** — BR-8's justification spans the excluded refresh: "…reused unchanged for the life of the channel, including by every refresh bridge (see 'The Ark channel type')…" (`:218-221`).
- **E-3** — `pinned_exit_delta` pinning is *defined* inside the excluded "exit_delta is pinned at open" subsection (`:399-403`) but *used* by in-scope Bridge/Open text (BR-3, OP-16). The stage-1 extract must re-home the definition.
- **E-4** — "Expiry is a deadline" offers "refresh or close" as the two escapes; only close survives stage 1 (`:129-131,134-135`).
- **E-5** — Open-by-upgrade's "no reset" note points at channel-refresh: "the upgrade resets nothing — the next channel refresh does" (`:536-539`).
- **E-6** — The upgrade's "no added trust" claim relies on channel-refresh eventually closing the arkoor double-sign window (`:671-673` + `02-vtxo.md:106-110`) — see I-1.
- **E-7** — Operation's "Normative (Ark side)" bullet list opens with the excluded channel-type bullet alongside in-scope bridge bullets (`:684-693`) — the list must be split.
- **E-8** — Upgrade admission Depth/Runway bullets cite the excluded CLTV-budget formula (`:597-609`; `cltv_safety_margin` and the doubled `pinned_exit_delta` defined only in `:317-393`) — see I-4/I-5.
- **E-9** — Pinned-parameters bullet mixes in-scope (bridge `nSequence`) and excluded (HTLC success-path CSV) clauses in one sentence (`:615-617`).
- **E-10** — The force-close deadline's "larger of (a)/(b)" term (b) is explicitly the success-path CSV's second `pinned_exit_delta` (`:1650-1655`) — the sharpest entanglement; see I-4.
- **E-11** — The atomic spent-state reservation names round participation (the refresh vehicle) as a co-equal flow (`:1410-1416`); stage-1's reservation abstraction should already accommodate a future refresh participant.
- **E-12** — Downgrade's over-depth remedy is "refresh-then-downgrade" (`:1379-1383`) — see I-9.
- **E-13** — Security notes restate "refresh well before expiry" as the user's protection (`:1893-1897`).
- **E-14** — "a channel touches the generic flows at exactly three seams — upgrade, refresh, downgrade" (`:1566-1571`); stage 1 implements 2 of 3.
- **E-15** — Compatibility bundles the two refresh-only messages (`leaf_vtxo_cosign` channel fields, round-participation response cap) with the one stage-1 message (`:1819-1828`); stage-1's actual new wire surface is only `channel_id` + `bridge_pub_nonce` on `arkoor_cosign_request`.
- **E-16** — Keysend-rejection / CLTV-floor-advertisement rules (`:364-374`) live wholly inside the excluded section — but their stage-1 treatment is NOT a clean drop; see the corrected I-6.
- **E-17** *(added at G0)* — UE-4's claim rules are stated in the spec through the Ark channel type's HTLC scripts (`08-channels.md:1598-1600`); under stage 1 the claims are genuinely stock BOLT-3 — the restructure re-points the reference at plain BOLT-3.

---

## PART 3 — IMPOSSIBLE OR MEANINGLESS WITHOUT THE EXCLUDED MECHANISMS
### (stage-1 profile relaxations to make explicit)

- **I-1 [HEADLINE]** — An upgrade from a third-party arkoor-received (double-sign-trust) VTXO can never shed that trust for the channel's life under stage 1: `channel-funding` is round-issuable *only* through the excluded channel-refresh flow (`02-vtxo.md:106-108`), and the spec's own trust note (`08-channels.md:671-673`) relies on "the next channel refresh". Escape = downgrade → plain refresh → re-upgrade (full close/reopen). **Resolution**: client-side default policy (refresh third-party arkoor receipts before upgrading) + explicit documentation; the exposure is the upgrader's own, so no server admission rule is required.
- **I-2** — "Refresh or close before expiry" loses one disjunct: a stage-1 channel cannot outlive its VTXO's expiry without a full downgrade → round-refresh → re-upgrade cycle. Needs proactive scheduling and documentation; long-lived in-place channels are explicitly not a stage-1 goal.
- **I-3** — "The server MUST refuse to open a channel of any other type" (`08-channels.md:250-251`) is inapplicable as written: stage 1 runs a stock LDK channel type. **Resolution**: profile designates the stage-1 channel type (stock `zero_fee_commitments`) and re-states the rule against it; `ark_channel` feature-bit negotiation drops entirely.
- **I-4 (rewritten at G0 — codex F1)** — The excluded formula's *second* `pinned_exit_delta` (the success-path CSV on the confirmed commitment) drops; **everything else survives**. The stage-1 **unified floor** is

  `F = channel_max_vtxo_exit_depth + pinned_exit_delta + cltv_claim_slack`

  — every relative timelock on the path plus the unroll **distance** to the actual commitment transaction: up to `D` *sequential* genesis confirmations (unrolling lag — each level confirms before the next broadcasts), the bridge's `d` BIP-68 CSV, and then bridge, commitment, and HTLC second-stage confirmations plus fee-bump lag inside `cltv_claim_slack` (a config with a conservative documented default). Checked arithmetic (I-8) and u16-representability wherever `F` feeds an LDK `cltv_expiry_delta`-typed field. The stop-forwarding/force-close deadline uses `F` on the two thresholds of EX-4.
- **I-5** — The upgrade admission Runway floor is the same excluded formula. **Resolution**: runway floor = the same unified floor `F`.
- **I-6 (rewritten at G0 — codex F1, Critical)** — The keysend/CLTV-floor rules do **NOT** cleanly drop. Rationale: LDK was told the funding is *already confirmed*, so its ordinary CLTV buffers budget **none** of the Ark actualization prefix — yet any on-chain HTLC claim on a stage-1 channel must first cross that whole prefix (`D` unroll + `d` bridge CSV + confirmations). An HTLC admitted below the floor can expire before any claim is even *possible* — a deterministic loss, distinct from the deferred success-vs-timeout race. The floor `F` MUST therefore be enforced at every per-HTLC CLTV decision:
  (a) invoices we issue: `min_final_cltv_expiry_delta ≥ F`;
  (b) every received HTLC: reject if `cltv_expiry − real_chain_height < F` — this per-HTLC check also handles **keysend** (no blanket reject needed: the spec's blanket rule existed only because the CSV-era floor could not be advertised per-payment);
  (c) the server's forwarding `cltv_expiry_delta ≥ F` on Ark-backed hops;
  (d) per-HTLC force-close scheduling: a channel holding an in-flight HTLC whose remaining budget approaches `F` force-closes now.
- **I-7** — The bridge/success-CSV cross-derivation invariant and its restore-time force-close guard (`:417-419`, `:310-315`) are vacuous; only the bridge-`nSequence` half (BR-3/BR-8) survives.
- **I-8** — The `≤ 65535` checked-arithmetic bound is a property of the excluded formula; stage-1's simplified floor needs its own (looser) overflow check, kept in checked arithmetic.
- **I-9** — "Too deep to split ⇒ refresh-then-downgrade" has **no substitute**: a stage-1 channel whose backing depth exceeds the downgrade bound has no cooperative close path, forever. **Resolution (mandatory design decision)**: reserve downgrade headroom at open-time admission — require the resulting channel-VTXO depth to leave room for a worst-case (checkpointed) split, i.e. `resulting_depth ≤ pinned channel_max_vtxo_exit_depth − 2` — and prefer `use_checkpoint=false` upgrades (OP-3) to minimize depth consumed at open. **G0 amendment (codex F6)**: the reservation guarantees nothing if the server's LIVE `max_vtxo_exit_depth` later *decreases* — the generic split eligibility check (DA-8) uses the then-current bound, and the pinned value cannot override ARK #2. The server MUST refuse a configuration decrease that would strand any live channel past downgrade eligibility (or require those channels closed first); enforce checked `max_vtxo_exit_depth ≥ 2`; the client retains its pre-close live-bound re-check; tested across a restart with a lowered bound.
- **I-10** *(added at G0 — codex F3)* — OP-21's suspend-until-reestablished is **unachievable on stock LDK**: withdrawing the funding confirmation force-closes an active channel rather than suspending it. **Stage-1 disposition**: the chain-feed adapter withdraws the virtual funding confirmation only on a genuine (deep) reorg of the anchor — anchors are typically deeply confirmed at open — and the resulting stock force-close is accepted as the fail-closed direction the spec itself prefers (WD-15). Proven in the spike. Related stock-LDK constraint, same origin: the confirmed funding position synthesizes a real SCID from `(height, tx_index, vout)` and LDK asserts on collisions — the fed transaction index MUST be allocated deterministically per channel (derived from the bridge txid, so both peers agree without coordination), persisted, and collision-handled.
