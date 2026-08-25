# Codex G1 review: MR-6 payments design note (6 rounds to PASS)

Note: `docs/plans/2026-08-25-mr6-payments-design.md` (rev6 = the PASS
state; earlier revisions are folded in-place as "G1 round N" sections).
Reviewer ran read-only against bark-stage1 with pinned LDK 0.2.4
(`~/.cargo/registry/.../lightning-0.2.4`); every load-bearing premise
was verified with line-level citations, recorded below because they are
the implementation's anchors.

## Round trail

- **R1: FAIL, 7 P1 / 5 P2** — output-rekey strands outputs (LDK
  balances carry no outpoint); no payment journal while payment events
  are discarded; A→captaind→B impossible (push_msat 0, push rejected);
  stock LDK enforces the OUTGOING channel's delta, not F_in;
  CLTV_CLAIM_BUFFER covers only inbound-with-preimage; the HTLC bump
  handler batches (CPFP stake model unusable verbatim); forwarding caps
  negotiated at open (no retrofit). P2: Filter-based watch feed;
  embedded invoice/pay/keysend APIs; MPP per-part expiry invisible;
  msat movements; Theirs gating.
- **R2: FAIL, 5 P1** — claim-time floor seam demanded; uniform-F
  rollout invariant (LDK prev_config grace); barrier must await MANAGER
  durability; journal needs a command outbox + PaymentForwarded
  handling; cap capability must be a concrete predicate.
- **R3: FAIL, 2 P1 / 5 P2 — premises VERIFIED** (see below) —
  must_drive_onchain (failed-but-uncommitted-removal HTLCs must not
  trigger the rung); H_cap/dust/kill-switch completion; MPP
  undisableable; capability evidence persisted at OpenChannelRequest;
  orphaned-descriptor terminality; operator-guide gate; command-side
  crash cuts.
- **R4: FAIL, 1 P1** — capability gate must cover ACCEPTANCE, not just
  forwarding (legacy inbound caps live for direct/incomplete-MPP HTLCs
  pre-claim); zero unknown/nonconforming receiving channels before
  enablement, close-and-reopen enforced.
- **R5: FAIL, 1 P2** — complete vs incomplete MPP distinguished:
  three-tick expiry covers only INCOMPLETE sets; complete multipart
  claimables must be FAILED BACKWARDS explicitly; sends pin
  max_path_count = 1.
- **R6: PASS** — "the fold is sound"; P3 wording residual only.

## Verified stock-LDK premises (0.2.4, line-cited by the reviewer)

- **Claim seam**: keysend onion decoding stores the preimage in
  ChannelManager routing state only (`ln/onion_payment.rs:369`); the
  monitor receives a preimage only while CLAIMING
  (`ln/channel.rs:7375`); monitor-triggered inbound force-close
  requires a stored preimage (`chain/channelmonitor.rs:6068`). ⇒
  fail-backwards at PaymentClaimable never creates a claim obligation.
- **Invoice ordering**: `min_final_cltv_expiry_delta` rejects at
  `ln/channelmanager.rs:7928-7936`, BEFORE `check_total_value!` and
  `PaymentClaimable`.
- **Config grace**: forwarding checks fall back to `prev_config` on
  failure (`ln/channel.rs:10845`) ⇒ F* immutable-while-channels-exist.
- **Forwarding delta**: incoming expiry is validated against the
  SELECTED OUTGOING channel's `cltv_expiry_delta`
  (`ln/channelmanager.rs:4923`, `ln/channel.rs:10830`).
- **Irrevocable commitment**: final-hop acceptance logic runs after the
  HTLC is committed (`ln/channel.rs:10871`).
- **Manager durability**: manager persistence is a separate async write
  awaited by the background processor
  (`lightning-background-processor-0.2.3/src/lib.rs:1079`); bark's own
  `bark-channels/src/driver.rs:11` documents handler-completion ≠
  manager-durable.
- **PaymentForwarded**: no payment identifier; LDK documents
  application-side tracking beyond pending-PaymentId dedupe
  (`ln/channelmanager.rs:5428`, `events/mod.rs:1319`).
- **MPP**: `basic_mpp` always advertised
  (`ln/channelmanager.rs:15752`); parts aggregate before
  `PaymentClaimable` (`:7865`); INCOMPLETE sets expire after three
  timer ticks (`:8364`) while COMPLETE ones are retained
  (`:8376`, `payment_tests.rs:442`); one `receiving_channel_ids` entry
  per part. Router default allows up to ten paths
  (`routing/router.rs:794`) ⇒ pin `max_path_count = 1`.
- **Failed-HTLC lingering**: `fail_htlc_backwards` removes claimability
  but the HTLC stays in both commitments until the removal handshake
  (`ln/channelmanager.rs:8457`) ⇒ `must_drive_onchain` excludes them.
- **Caps**: `max_htlc_value_in_flight` is an aggregate inbound knob
  (`util/config.rs:66-77`), no outbound-total knob;
  `max_dust_htlc_exposure` is the aggregate dust bound
  (`util/config.rs:569`); negotiated count/dust are visible only in
  `OpenChannelRequest` (`events/mod.rs:1606`), not in ChannelDetails.
- **Balances carry no outpoint**: `chain/channelmonitor.rs:886`
  (ClaimableAwaitingConfirmations = amount/height/source).
- **Defense-in-depth floor arithmetic**: reconstruct the single HTLC's
  expiry as `claim_deadline + HTLC_FAIL_BACK_BUFFER`, checked, fail
  closed on `None`; boundary tests at exactly `F` vs `F − 1`.

## Open P3 (non-blocking)

Stale "all MPP times out" phrasing in one historical round section of
the note; the live sections say complete = failed backwards,
incomplete = tick expiry.
