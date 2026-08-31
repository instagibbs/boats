# Channel revocation/justice gap — initial assessment

Status: DRAFT, SUPERSEDED for the threat model by the full audit
`2026-08-28-channel-security-audit.md` (§"Refined threat model"). This
memo is the first assessment; read it for the justice detail, but the
audit is authoritative on WHEN the revoked commitment reaches the chain.

## The threat (spec-sanctioned, not hypothetical)

`08-channels.md:288` describes a retained-bridge server force-closing by
broadcasting **bridge + commitment**. IMPORTANT REFINEMENT (from the
audit): in STAGE-1 the server stores no bridge witness and cannot
broadcast the bridge — the retained-bridge early force-close is a
STAGE-2 (CSV-gated) capability, absent here. So a revoked SERVER
commitment reaches the chain only when the CLIENT is exiting (the client
puts the bridge on-chain) AND a malicious server RACES a retained
**revoked** commitment onto that bridge, winning the funding-output
spend. Either way the harm is the same — the client is rewound past
payments moved to it, unpunished — but the precondition is a client exit
plus an adversarial server race, not an idle-channel server force-close.

The counter is standard LN justice: when a party broadcasts a revoked
state, the counterparty uses the revealed per-commitment-secret to claim
the cheater's `to_local` via the revocation branch. The spec keeps this
branch — `08-channels.md:1300`: "The revocation branch is unchanged and
immediate — justice gets no response window, deliberately."

## What is wired (client side)

- **Detection: WIRED (traced, corrected).** `sync_channel_watch`
  (`channels/watch.rs`) runs on EVERY daemon tick before the other
  channel duties (`daemon/mod.rs:354`). It reads every LDK
  Filter-registered watched output (`watched_outputs()` — the funding
  outpoint and commitment outputs of a live channel) and scans the
  chain for their spenders (`chain.confirmed_spenders(outpoints)`,
  watch.rs:275), feeding matches to LDK. So a counterparty commitment
  landing on an idle Ready channel IS detected proactively → LDK fires
  `Event::ChannelClosed` → `observe_channel_close` → the `peer_close`
  rung starts the exit. (An earlier draft of this memo wrongly called
  idle detection unwired; the only-in-exit `txs_spending_inputs` scan
  at mod.rs:2358 is a SECOND scan during the exit, not the sole one.)
- **Response to the LATEST counterparty commitment: WIRED.**
  `obtain_commitment` → `Theirs` → the `StaticPayment` sweep recovers
  the client's balance (covered by the c3 `theirs_commitment_sweeps`
  vector).
- **Foreign-commitment handling**: `obtain_commitment` (mod.rs:2327)
  selects a confirmed foreign spend of the funding and returns
  `ObtainedCommitment::Theirs`; the exit then builds **StaticPayment**
  sweeps — the client's OWN balance (`to_remote`) off that commitment.

## It is a REGRESSION, with a reference implementation

The legacy `bark-lightning` crate (the impl the spec was partly derived
from) HANDLES this. `ArkBroadcaster::broadcast_transactions`
(`bark-lightning/src/broadcaster.rs:555`):

```
if let Some(keys) = self.is_ark_channel_commitment(tx) {
    self.queue_ark_exit(keys, tx.clone());   // a commitment: sequence the exit
} else {
    let _ = self.inner.broadcast(tx);        // ANYTHING ELSE: relay normally
}
```

So LDK's justice penalty (which spends the counterparty's commitment,
NOT the bridge → not an Ark commitment) is RELAYED. Its signer wires
the justice signatures (`bark-lightning/src/signer.rs:219,238` —
`sign_justice_revoked_output` / `sign_justice_revoked_htlc` delegating
to LDK's `EcdsaChannelSigner`). Justice works end to end in the legacy.

Stage-1 REGRESSED this: the client `QueueBroadcaster`
(`bark/src/channels/broadcaster.rs`) captures EVERY LDK broadcast and
relays NOTHING ("Nothing is ever handed to a network relay from here").
The justice signing is still present (LDK's signer), but the penalty is
captured and dropped. Note the asymmetry: the SERVER-side
`CaptureBroadcaster` got the relay-non-candidate behavior back in c3
(the HTLC-claim fix), but the CLIENT-side `QueueBroadcaster` did not —
so the fix pattern already exists in-tree, applied to the wrong side.

### The reference: `ark-channels-bridge-2026-06-18` (bark repo)

The branch we are basing off of has the full justice machinery — a
dedicated exit driver that stage-1 has NO equivalent of:

- **`bark/src/lightning/exit_driver.rs`** — `ChannelExitScratch` +
  `LdkChannelExitDriver`:
  - `detected_bridge_spends` — detects a counterparty commitment
    spending the bridge funding (the server force-close);
  - `pending_ldk_confirmations` — **FLUSHES the justice / HTLC TXs from
    LDK's in-memory broadcaster** (the penalties LDK builds), relays
    them, and feeds their confirmations back to the chain monitor so
    LDK emits the follow-up `SpendableOutputs`;
  - `fed_output_spends` — feeds the CHEATER'S second-stage spends (its
    HTLC-success/timeout tx on a revoked commitment) to the monitor.
- **`bark-lightning/src/broadcaster.rs`** — a TYPED broadcaster:
  captures the cooperative-close tx (virtual funding, doomed to relay)
  and RELAYS everything else.
- **`bark-lightning/src/signer.rs`** — justice signing
  (`sign_justice_revoked_output` / `_htlc`, delegated to LDK).
- **`bark-lightning/tests/channel_justice.rs`** (on the sibling
  `bark-lightning-channels` branch, tip cf10a0349) — the e2e vector:
  "server detects revoked commitment TX and sweeps via justice";
  captures the commitment at state N, advances state (N revoked), feeds
  the OLD commitment to the monitor, and asserts a justice TX is
  BROADCAST via `BroadcasterInterface` spending the revoked `to_local`.
  Its own comment: "LDK builds the justice TX internally (using
  sign_justice_revoked_output) and broadcasts it directly — NOT via
  BumpTransaction events" — i.e. it arrives at the broadcaster, which
  MUST relay it.

### What stage-1 dropped (the regression, precisely)

- The client `QueueBroadcaster` captures every LDK broadcast and relays
  none — so LDK's justice/HTLC TXs are dropped at the seam the
  reference relays them from.
- There is no exit-driver flush of LDK's in-memory broadcaster, no
  feed-back of justice/HTLC confirmations, and no feed of the cheater's
  second-stage spends.
- `ChannelSweepKind::Justice` exists but is never constructed.

The server side is PARTIALLY back: c3's `CaptureBroadcaster` relays
non-candidate LDK broadcasts, so a server punishing a client's revoked
commitment MIGHT work (untested). The client side is fully regressed.

## The gap (client side) — a theft vector

The `Theirs` path recovers only the client's `to_remote` from whatever
commitment landed. If that commitment is REVOKED, `to_remote` is the
OLD (smaller) client balance and the server keeps the OLD (larger)
`to_local` — the client silently loses every payment that moved balance
its way after the revoked state. There is NO punishment:

1. `ChannelSweepKind::Justice` exists (`exit/models/states.rs:162`) but
   is **never constructed** anywhere — the descriptor→kind mapping
   (mod.rs:2542) only ever yields `ToLocal` / `StaticPayment`. No
   marker, no TODO; silently unwired.
2. LDK's ChannelMonitor DOES detect a revoked broadcast (it holds the
   secrets) and produces the penalty transaction, handing it to the
   BroadcasterInterface — the client's `QueueBroadcaster`, which
   **captures and never relays** ("Nothing is ever handed to a network
   relay from here", broadcaster.rs). So even LDK's own penalty never
   reaches the network.
3. The client's event handlers cover `ChannelClose` (capture) and
   `HTLCResolution` (its own HTLC claims, relayed via its chain path) —
   there is NO justice/penalty handler.

Net: as shipped, a retained-bridge server can broadcast a revoked
commitment and the client will not punish it. This is money-safety, not
cosmetics.

## Adjacent surfaces to scope in the review

- **HTLC-on-revoked-commitment (second stage)**: the revoked commitment
  may carry HTLC outputs; their revocation branches need the same
  justice treatment. Unwired for the same reason.
- **Server-side justice (client cheats)**: a malicious client force-
  closing on its own revoked commitment via its exit. The server DOES
  feed its LDK monitor the chain (`transactions_confirmed`, mod.rs:766)
  and the c3 broadcaster fix RELAYS non-candidate LDK broadcasts — so
  the server's penalty MIGHT reach chain. Wired-but-untested; needs a
  seam to force LDK to broadcast a revoked holder commitment.
- **Detection proactivity** (above): idle-client watch of the funding.
- **Scope-transition forfeit** (`08-channels.md:2158`): a separate,
  Ark-specific defense — the old-scope VTXO is forfeited so an
  old-SCOPE force-close can't profit "even at a revoked state." Distinct
  from per-state justice; verify it is wired + covered, and how it
  interacts with per-state justice.
- **Watch-tower / liveness assumption**: justice requires the victim to
  be watching within the window. The spec gives justice "no response
  window" (immediate), but the CLIENT must still see the broadcast and
  act before the cheater's `to_local` CSV matures — what is that CSV,
  and what liveness does the client's watch assume?

## Questions for the requirements review

1. Is per-state justice IN SCOPE for stage 1, or was it deliberately
   deferred (and if so, what compensating control makes a retained-
   bridge server safe — e.g. the server MUST NOT retain the bridge)?
2. If in scope: remediation shape — relay LDK's penalty for the justice
   case (narrow the QueueBroadcaster no-relay rule), or reconstruct the
   justice claim in the exit machinery (build the `Justice` sweep from
   the monitor's balance/descriptors, like the other sweeps)?
3. The detection-proactivity and CSV-liveness questions above.
4. The full blast radius: is this the only unwired branch, or are there
   sibling states (HTLC second-stage, scope-forfeit) equally exposed?

## Test homes (once requirements are set)

`testing/tests/bark_sdk/lifecycle.rs` — the c3 suite already has the
dev-seam force-close, the exit→commitment→sweep drive, and the `Theirs`
sweep. A revoked-commitment justice vector slots in, but needs a seam
to make LDK broadcast a revoked (old) holder commitment (LDK won't do
that voluntarily). The server-side justice vector needs the symmetric
seam on the captaind node.
