# MR-7 — surface + lifecycle e2e completion (design note, rev4, pre-G1)

Per the stage-1 plan (§5, renumbered; originally "surface + hardening"):
REST `/channels`, bark-json DTOs, CLI subcommands, barkd integration,
the consolidated adversarial sweep via `ArkRpcProxy`, and operator
docs. **Rev2 (2026-08-27)** added the payments surface MR-6 created and
absorbed the e2e debts its arc recorded. **Rev3** corrects rev2 against
the shipped wallet API per G1 round 1 (12 findings): action-oriented
open, keysend excluded from the surface, stranded is NOT re-payable,
the expiry lifecycle is deadline-close→reopen (funding is deliberately
non-refreshable), blocked HTLC claims surfaced, the c3 debug seams
enumerated, the reorg vector matched to shipped lease semantics. **Rev4** absorbs
G1 round 2 (6 findings): the binaries are NOT channel-enabled today
(feature propagation + wallet-open wiring + driver lifecycle become
explicit c1/c2 scope), the open action gets a real pollable resource,
seam (b) becomes a composite persistence barrier, pre-release
close/exit arcs join the matrix, seam (f) exports the commitment only
(the claim relays itself), and the HTTP error table is pinned. The
bar: after this MR, every lifecycle a user can hit has end-to-end
coverage. Code waits for Greg's go.

## What already exists (verified 2026-08-27 at `a491cea7f`)

- **Wallet-lib API**, including payments: `open_channel` (amount only —
  the wallet discovers the server; returns a `WalletActionId`, the
  record appearing as the action advances), `close_channel`,
  `start_channel_exit`, `cancel_channel_exit`,
  `channel_opens_in_progress`, `sync_pending_channel_{opens,closes}`,
  `maintain_channel_deadlines`, `maintain_downgrade_watches`,
  `maintain_channel_payments`, `pay_channel_invoice` (with an amount
  override for amountless invoices), `channel_keysend` (requires a
  `KeysendRoute` — recipient floor + optional intra-ark hop; no route
  discovery exists), `channel_bolt11_invoice` (optional amount +
  description; the floor is DERIVED, not caller-supplied);
  `list_channel_payments`; the persister exposes `list_channel_records`
  (state machine `Cosigned … Closed`), the payment journal (`pending`,
  `sent`, `failed`, `claiming`, `received`, `stranded` — stranded and
  sent/received/claiming rows are NON-replaceable; only `failed` may be
  re-paid in place), and the durable `BlockedHtlcClaim`
  operator-attention marker.
- **Daemon integration done in the LIBRARY**: barkd drives sync, the
  ungated chain-safety duties, the deadline rungs, the close driver
  and the payments sweep; barkd serves bark-rest, so the `/channels`
  router lands there. BUT the production binaries are NOT
  channel-enabled today (G1 r2 f1): bark-cli and bark-rest disable
  default features without enabling `channels`; the shared wallet
  opener never sets `OpenWalletArgs.channels`; the one-shot CLI starts
  its daemon only for `watch`, while the channel payment verbs refuse
  a stopped node. Enabling all of this is explicit c1/c2 scope.
- **Server admin surface**: `ChannelCollectInvoice` (the collect leg)
  exists as a captaind admin RPC; e2e drives it via
  `ChannelAdminClient`.
- **DTO toeholds**: `ArkInfo` channel fields in bark-json (cli + web),
  the movements CHANNELS kind (one movement per payment, figures at
  finish).
- **Missing entirely**: any CLI noun, any REST route, any channel or
  payment list/status DTO, a PUBLIC channel-status accessor composing
  the record with the LDK balance view (today crate-private), the
  proxy-based adversarial sweep, the force-close e2e staging and its
  debug seams, operator docs beyond `doc/channel-payments.md`.

## Proposed commits (each independently green, MR-5/6 rhythm)

**c1 — bark-json channel DTOs + REST `/channels` (+ the wallet wiring
the surface needs).**
Per-resource axum router in bark-rest matching the exits/boards pattern
(utoipa docs, `bark_json::web` types):
- **Binary enablement (G1 r2 f1)**: propagate the `channels` feature
  into bark-rest (and bark-cli in c2); wire `OpenWalletArgs.channels`
  through the shared opener (dual-wiring the concrete SQLite store
  into `persister` and `channels`; filestore behavior defined —
  channels REQUIRE the sqlite store, a filestore wallet refuses the
  channel verbs with the typed error).
- New PUBLIC wallet accessor `channel_statuses()`: the persisted record
  joined with the LDK view when live (balances, usability) and the
  recorded close otherwise, plus any `BlockedHtlcClaim` marker and the
  originating open-action id — one DTO serves list, get, CLI and the
  tests.
- `GET /channels` — list of that DTO: channel id, record state string,
  funding value, balances-when-known, backing VTXO id + expiry height,
  blocked-claim marker.
- `GET /channels/{channel_id}` — one record + close-phase detail
  (retained-split presence, response settlement, exit state).
- `POST /channels/open` — amount only (the wallet discovers the
  server; there is no caller node address). ACTION-ORIENTED with a
  REAL pollable resource (G1 r2 f2): `202` +
  `Location: /channels/opens/{action_id}`; `GET /channels/opens/{id}`
  reports progress (the pre-record `Establishing` park included),
  eventual channel/backing ids, and TERMINAL failure (a pre-record
  cosign refusal removes the checkpoint — the action resource is what
  makes that observable; polling the channel list alone cannot be).
- `POST /channels/{channel_id}/close`, `/exit`, `/exit/cancel` — as
  before (decision 1).
- **Payments:**
  - `POST /channels/payments` — body `{invoice, amount_msat?}` (the
    override is REQUIRED for amountless invoices, refused on amounted
    ones — mirroring the wallet API). Returns the journal row (payment
    id, state). KEYSEND IS EXCLUDED from the surface (decision 5): the
    wallet API needs a recipient-produced route descriptor and no
    route discovery exists; it stays wallet-API/test-only in stage 1.
  - `GET /channels/payments` — the journal verbatim, states included
    (`stranded` is user-visible BY DESIGN; see decision 7 for its
    resolution semantics).
  - `POST /channels/invoice` — `{amount_msat?, description?}` →
    bolt11 with the routing hint; the CLTV floor is derived from the
    admitted channels; refused when NO channel is claim-admissible.
- **Error contract pinned** (G1 r2 f6 — today every wallet error is an
  untyped anyhow that bark-rest turns into 500): malformed input → 400,
  unknown channel/action → 404, invalid state for the verb and
  policy/floor refusals → 422, node not running / not synced → 503,
  internal → 500. Mechanism: downcastable wallet refusal categories (a
  small typed error enum on the channel wallet APIs) so handlers never
  parse error strings.
- openapi.json + bark-rest-client regeneration (the G0-review artifact
  set).

**c2 — CLI `bark channel …`.**
Noun `channel`: `open`, `close`, `list`, `exit`, `cancel-exit`,
`pay <invoice> [--amount]`, `invoice [amount] [--description]`,
`payments` (journal listing incl. stranded and blocked-claim
visibility). Thin: parse, call the wallet API, print the same
bark-json types REST uses. No `keysend` (decision 5). One-shot
lifecycle (G1 r2 f1): the channel verbs start and AWAIT the channel
driver for the command's duration (today only `watch` starts the
daemon and the payment verbs refuse a stopped node); `open` MAY poll
the action to present a synchronous UX. The c2 test is an EXECUTABLE
open → pay → invoice smoke, not JSON-shape-only.

**c3 — lifecycle e2e completion (the heavy staging + its seams).**
The machinery-landed-proof-didn't list from MR-5/6 plus the
force-close choreography no prior suite staged. **Prerequisite
sub-scope — deterministic seams (G1 r1 f9, r2 f3/f5):** acknowledged
debug gates for (a) the upstream fulfill (wedging a forward at the
irrevocable point), (b) the stranded window — a COMPOSITE barrier, not
a caller-side gate (the journal write, manager persistence and the
PaymentSent/PaymentFailed handling race independently through the
driver): the seam holds BOTH the driver's manager persist AND the
terminal payment events after the journal row lands, the test then
HARD-crashes and asserts the pre-restart manager blob lacks the
payment id before expecting `stranded`; (c) client post-claim-binding
and (d) server post-collect-binding (crash between CAS and
claim_funds); (e) a server-side cooperative-shutdown trigger
(peer-initiated close); and (f) an admin-only
force-close/export-COMMITMENT seam — captaind's broadcaster
deliberately captures commitments without relaying, so the harness
receives the captured commitment and broadcasts it itself. The HTLC
claim needs NO export: captaind's claim path relays to bitcoind on its
own — the harness just mines it and feeds the client watch. All seams
are debug/test-gated, none reachable in production configuration.
Vectors:
- **Force-close with HTLCs in flight, both claim directions**: wedge
  the upstream fulfill (seam a), force-close, client unrolls to the
  commitment, then: the SERVER funds and relays its HTLC-success claim
  (rounds-wallet funding, durable input locks) and it confirms; a
  captaind restart mid-claim re-arms the V63 locks and the claim still
  lands; a concurrent round never spends a claim-locked coin.
- **The backwards scrape (stage-1 addendum item c)**: seam f exports
  the captaind commitment, the harness broadcasts and mines it,
  captaind's own claim path relays its preimage claim to bitcoind, the
  harness mines that → the client watch feeds it → LDK scrapes the
  preimage → our upstream/inbound HTLC is claimed off it.
- **`Theirs` sweeps**: the captaind commitment (seam f) reaches the
  chain; the client's StaticPayment sweep collects its balance.
- **Blocked HTLC claim**: insufficient claim funding → the durable
  marker is visible on the status surface → top-up → the claim
  confirms and the marker clears.
- **The forced-exit rungs**: `htlc_deadline` (HTLC approaching the
  floor with the peer wedged → automatic exit, cause recorded) and
  `peer_close` (unilateral close observed → exit) — plus the negative
  control: a PEER-INITIATED COOPERATIVE close (seam e) is adopted and
  completes WITHOUT tripping `peer_close`, from Ready and from
  mid-exit.
- **Pre-release arcs (G1 r2 f4, split per r3 f1 — `Cosigned` is
  shutdown-legal but NOT payment-capable or forced-exit-scoped in the
  shipped code)**: at `Cosigned` (deterministic hold before
  registration), peer-cooperative adoption and a non-cooperative peer
  close — the close routes through registration/bookkeeping completion
  into the existing Registered-side handling, never a direct exit; at
  `Registered` (hold before confirmation feeding), the same closes
  PLUS the HTLC-deadline forced exit (Registered is
  payment-capable and rung-scoped). Both assert the in-flight open
  action resolves cleanly rather than wedging. No seeded `Cosigned`
  HTLCs and no early confirmation feeding — neither is a shipped
  lifecycle. **This vector carries a shipped-code CONVERGENCE FIX
  (G1 r4 f1 + r5 f1, a real user-reachable wedge)**: cooperative
  completion before release moves the record to
  `outcome_ready`/`downgraded` — states the open action's
  Registering/Feeding phases neither accept nor terminate on — and
  the deeper interleave is worse: deadline maintenance can EXIT an
  `OutcomeReady` channel at the hard line before registration is
  possible, after which the server permanently refuses release while
  the open action consults local terminal states only AFTER that
  doomed RPC, parking every post-cosign rejection forever. Fix (r5's
  second alternative): the open action consults local close/exit-owned
  terminal states BEFORE attempting registration and reconciles them
  terminally — input/change bookkeeping, movement and checkpoint
  resolved, `Advance::Done` — without requiring the registration RPC;
  where registration DID land, the existing exiting/closed-style
  reconcile applies. The c3 vector proves both: cooperative completion
  at `Cosigned`, and a hard-line exit winning before registration
  (registration held through hard-line confirmation), each ending with
  the open action cleanly resolved.
- **Close under pending payments**: cooperative close with a journaled
  in-flight send and a claimable receive; nothing double-books, the
  close completes, the journal resolves.
- **Payments crash/restart matrix**: the stranded window via seam b's
  composite barrier + hard crash (asserting the persisted manager
  lacks the payment id, then `stranded` on restart; resolved by a LATE
  authoritative PaymentSent and, separately, by PaymentFailed —
  stranded is never re-paid, see decision 7); failed → SAME-invoice
  retry → pending → sent (the in-place replacement the journal
  permits, currently untested); crash at the open's `Establishing`
  park and the close's `Negotiating` phase (the earliest user-visible
  states, uncovered); captaind crash between collect-claiming and
  received (seam d); and the CLIENT claim-binding cell (seam c, G1 r3
  f2): hard crash between the binding CAS and `claim_funds`, restart,
  LDK replays the claimable event → the SAME claim id is re-admitted
  through the CAS and the journal transitions to `received` exactly
  once.
- **The lease/reorg race, as shipped**: a claim racing a watch
  re-registration (the generation bump) fails BACKWARDS — permanently,
  by design — and the PAYER's retry with a fresh attempt succeeds
  after the next quiescent pass. (No re-admission of the failed HTLC
  is promised; that is not the shipped semantics.)
- **Deadline-close → reopen composition**: the expiry treadmill as it
  actually works — funding is deliberately non-refreshable; near the
  deadline the maintenance rung closes (or exits), and the user
  reopens from the settled funds.
- **Sub-dust real close side** and the **chain-overrules-tombstone
  rescue** (MR-5 residuals unlocked by payments).
- **Quarantine**: nonconforming caps (seeded row) → forwarding off +
  offender claims refused, everything else functional.
- Explicitly OUT: hostile-anchor-child (rides the upstream
  fee-discipline submission); MPP (single-part by design).

**c4 — the consolidated adversarial sweep (`ArkRpcProxy`).**
Pruned per G1 r1 f11 (several rev1 cells landed in the MR-5/6 suites:
downgrade cosign→registration restart, partial downgrade uploads,
post-registration replay, identical committed-cosign retry — the sweep
re-asserts those only as PROXY-LAYER regressions, labeled as such):
- **RG-4** at the live surface: byte-identical re-upload idempotent;
  partial upload leaves the operation unregistered (upgrade AND
  downgrade group), through the proxy.
- **IB bypass attempts** at the live gRPC surface (tampered
  attestation, spent/banned input, exiting input, unbalanced
  destinations, package-atomicity break).
- **WD-15 remaining cells**: mid-watch-response crash, server
  admission-commit→response crash (the client retry must be
  idempotent — pairs with RG-4), and proxy-loss variants (response
  dropped on the wire after the server committed).
- **Sacred replay with the grown cosign surface**: a committed-cosign
  retry under a CHANGED config skips the fresh-only screens AND
  returns the identical SCID index (request fields 9/10, response
  field 5), asserted at the proxy.

**c5 — operator docs.**
Stage-1 profile statement, forwarding posture (§3.7: ON when channels
enabled, capped and kill-switched via `channel_forwarding_enabled`,
intra-ark by construction), `vtxo_exit_delta` guidance (WD-1), the
**deadline lifecycle as shipped** (funding is non-refreshable; the
rungs close or exit near the deadline; the user reopens — no
"refresh keeps it alive" language), the payments operator page
(collect-leg residual statement extending `doc/channel-payments.md`,
timing-knob coherence profile ≥ depth + delta + slack, quarantine
semantics), and the payment-state UX: `stranded` means reconcile with
the recipient and obtain a FRESH invoice (never re-pay the same one);
`failed` may be retried on the same invoice; `BlockedHtlcClaim` means
top up the wallet.

## User-lifecycle coverage matrix

| Lifecycle | Coverage |
|---|---|
| board → open → ready; reopen ladder | existing (MR-3/5 suites) |
| open/close crash at the EARLIEST phases (Establishing park, Negotiating) | **c3** |
| pay / receive / forward / collect | existing (MR-6 c6 vectors) |
| pay: amountless-invoice override; failed → same-invoice retry | **c1/c3** |
| stranded: late PaymentSent / PaymentFailed resolution (no re-pay) | **c3** |
| cooperative close (client-initiated); close→reopen composition | existing (MR-5/6 suites) |
| cooperative close, PEER-initiated (incl. mid-exit; no peer_close trip) | **c3** |
| pre-release arcs: peer close at Cosigned; peer close + HTLC-deadline exit at Registered | **c3** |
| receive crash at the claim binding (CAS→claim_funds, exactly-once) | **c3** |
| close under pending payments | **c3** |
| user exit + cancel; reorg across the bridge | existing (MR-3 suite) |
| forced exit: htlc_deadline, peer_close | **c3** |
| force-close w/ HTLCs: server claim (+restart re-arm), backwards scrape, Theirs sweep | **c3** |
| blocked HTLC claim: visible → top-up → clears | **c1 surface + c3** |
| claim vs lease/reorg race (payer-retry semantics) | **c3** |
| deadline-close → reopen (the real expiry treadmill) | **c3** |
| quarantine (nonconforming caps) | **c3** |
| byzantine client vs the live server surface; sacred replay w/ SCID | **c4** |
| REST/CLI drive every surfaced verb; typed error contract | **c1/c2** tests |

## Design decisions to ratify (Greg)

1. **Channel exits under `/channels`, not `/exits`** (as rev1).
2. **Action-oriented open** (REVISED from rev1's synchronous open,
   which the wallet API cannot serve): `POST /channels/open` returns
   the action id; ids and readiness are observable via GET. The CLI
   MAY poll to present a synchronous UX.
3. **State exposure: record/journal state strings verbatim**,
   extended to payment states and the blocked-claim marker.
4. **No pagination/filtering on lists** (stage 1: few channels).
5. **Keysend is NOT surfaced in stage 1** (REVISED): the wallet API
   requires a recipient-produced route descriptor (recipient floor +
   intra-ark hop) and returns no payment handle; without route
   discovery the verb is not serviceable. Wallet-API/test-only.
6. **Payments surface lives under the `channel` noun**, not folded
   into the existing lightning-send path (post-stage-1 UX question).
7. **Stranded is terminal-until-authoritative** (NEW, matching the
   journal): never re-payable; resolved only by a late authoritative
   PaymentSent/PaymentFailed, else the user reconciles out-of-band and
   pays a FRESH invoice. The surface and docs say exactly this.
8. **The collect leg stays admin-only** (as rev2).
9. **Debug seams ship test-gated** (NEW): the six c3 seams (incl. the
   admin force-close/export-commitment seam) are compiled or
   config-gated out of production paths.

## Conformance mapping

RG-4 → c4; IB-1..7 (live-surface) → c4; WD-15 remaining cells → c4;
the addendum's on-chain-HTLC-resolution proof (item c) + Theirs → c3;
WD-1 guidance → c5; §3.7 posture → c5; the G0-review artifact
requirement (openapi + generated client) → c1.

## Test plan

c1: REST e2e in the bark-rest style (open → poll the ACTION resource
to ready, incl. a terminal-failure vector → pay (incl. amountless
override) → invoice (incl. no-admissible-channel refusal) → close →
observe downgraded/closed; exit → cancel refusal past the bar; the
typed error table per verb, status codes asserted). c2: an EXECUTABLE
CLI open → pay → invoice smoke (channel-enabled binary, driver
lifecycle exercised), JSON parsing into the DTOs. c3 and c4 are
themselves the test commits. c5: docs only, link-checked.
