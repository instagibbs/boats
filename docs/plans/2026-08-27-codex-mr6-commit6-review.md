# MR-6 commit 6 — the payments e2e: codex review record

Commit: `592f08add` "the payments e2e — and the bug only a real payment
could find" (bark-stage1, ark8-channels-stage1-close). Reviewer: codex.
Verdict: **PASS at round 4**. Battery green throughout (units /
bark-channels / SDK channels 24 incl. the 5 new payment vectors /
server channel 22).

## The vectors

1. **collect_pay_and_close_composition** — the collect leg end to end
   (journal sent + proof preimage, the sweep's movement mirror, the
   repeat-pay refusal), then the cooperative close carrying the moved
   balance through the downgrade (370k/30k split).
2. **pay_between_clients_via_server** — A→captaind→B through the
   forward: sender sent / receiver received at the exact invoice
   amount, polled PaymentForwarded telemetry, and a keysend riding the
   hop parsed from B's own invoice hint. The sender retries with fresh
   invoices while the fresh receiver's freshness lease arms (by
   design).
3. **keysend_at_floor_claims** — the deterministic half of the floor:
   an at-floor keysend claims (router padding only ADDS blocks). The
   BOUNDARY (F admits / F−1 refuses / absence+overflow fail closed)
   moved to a unit test on the seam's now-PURE measurement
   (measure_claim_floor): LDK pads finals with a random 0–120 block
   shadow offset, so no public-surface send can deterministically
   undercut a floor — the round-1 e2e "F−1 fails" pass was a
   stale-lease artifact codex caught.
4. **collect_invoice_is_single_claim** — two payers, one hash; the
   second settles failed.
5. **crash_cut_send_strands** — the exact crash shape (a pending row
   inserted into the stopped wallet's sqlite), stranding at startup,
   listing visibility, non-replacement, restart idempotence.

## The product bug the matrix found (and its two-stage fix)

**The synthesized SCID split-brain.** Client-to-client payments were
IMPOSSIBLE: both sides derive the virtual funding's synthetic position
"deterministically from the bridge txid" — with different formulas
(client: 4 txid bytes mod band; server: 3 raw bytes). Different
positions ⇒ different funding scids ⇒ each side's private
channel_update names a scid the peer never fed ⇒ forwarding_info never
forms ⇒ no invoice hints. Round 1's formula alignment was rejected by
codex round 2 (same-height base collisions re-split through node-local
probes; birthday math makes that a certainty at scale) — the shipped
fix is COORDINATION:

- the SERVER allocates the index globally at cosign
  (allocate_channel_scid_index — idempotent per channel, band-walked,
  tx_index globally UNIQUE + band CHECK; V62 reshapes channel_scid to
  (channel_id PK, tx_index UNIQUE), anchor keying dropped);
- the index rides the bridge cosign RESPONSE (new required group
  field, band-validated on receipt), is pinned on the client record
  (m0054), and the client's local derivation is deleted;
- only the INDEX is allocated — the height is the CURRENT anchor's,
  derived at feed time on both sides and never the persisted pair's
  (round 3: a crash-then-reorg would re-feed a stale height while the
  server derives the new anchor's); the stored pair is re-recordable
  bookkeeping, the index immovable.

Also recorded: the router-saturation hypothesis for an early
RouteNotFound was TESTED AND REFUTED (exact-liquidity/infinite-capacity
hops are exempt from the saturation shift; single-path routing retries
saturation-free) — no knob shipped. The 2s test tick exists because the
claim seam's freshness lease would otherwise refuse test receives for
most of the default 60s interval.

## Deferred vectors (recorded, justified)

- Hostile multi-part sends: unrepresentable in this harness (one
  channel per client, max_path_count pinned to 1 — no topology
  produces a second part; the seam's one-entry check is total).
- The even-TLV refusal and the wedged-peer HTLC-rung firing: need
  seams the production surface deliberately does not expose (a custom
  sender; a peer withholding the removal handshake) — pinned by review
  + LDK citations in the commit-5 record.

## Round history

r1 (2 P1): the F−1 vector was nondeterministic (shadow padding; its
pass was a stale-lease artifact) → pure-measurement unit boundary +
at-floor e2e; the saturation "fix" was a no-op → removed, recorded as
refuted. r2 (2 P1): node-local collision probing re-splits at scale →
the coordinated allocation; telemetry race → poll. r3 (1 P1): persisted
height re-splits after crash+reorg → height always the current
anchor's. r4: PASS.
