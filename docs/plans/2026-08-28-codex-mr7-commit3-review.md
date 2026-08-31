# Codex review record — MR-7 c3: the lifecycle e2e suite

**Commit:** `a1efa93fe` "testing: complete the channel lifecycle e2e
suite" (amended through the rounds), on c2 `cf423cb8a`, branch
`ark8-channels-stage1-payments`.
**Verdict: PASS in 2 rounds.** The full lifecycle suite green (25/25
incl. one barkd name-collision test); whole battery green — 634 unit
tests, bark-channels, bark-sdk channels 19/19, payments 5/5, server
channel 22/22, CLI channel smoke.

## What shipped

`testing/tests/bark_sdk/lifecycle.rs` — twenty-four vectors proving
the arcs the machinery landed without an end-to-end test:

- **The force-close arc** (`force_close_server_claims_and_payer_scrapes`):
  a wedged forward forces the channel on-chain; the server funds and
  RELAYS its HTLC-success claim, survives a captaind restart re-arming
  the durable V63 claim-input locks, runs a CONCURRENT ROUND against
  those locks (unconfirmed claim, rounds-wallet-funded round), and the
  offline payer scrapes the preimage off the chain before its exit
  recovers the balance.
- **`Theirs` sweep**: the exported captaind commitment is mined the
  moment the bridge confirms; the exit resolves through the foreign
  commitment and the StaticPayment sweep collects.
- **Blocked HTLC claim**: an unfundable timeout claim surfaces the
  durable marker → top-up → the claim confirms and the marker clears.
- **Forced-exit rungs**: `htlc_deadline` (inbound-staged: a
  claimed-but-unresolvable fulfill against a stopped captaind — the
  cause recorded, the exit non-cancelable) and the deadline treadmill
  (the cooperative window closes, never exits; refreshed funds reopen).
- **Pre-release arcs**: peer cooperative + force closes at `cosigned`
  and `registered`, the hard line taking `registered` and
  `outcome_ready` records — the open action converges through the
  close/exit-owned terminal states, BOTH convergence proofs (the
  coop-at-cosigned and the hard-line-exit-before-registration case the
  shipped convergence fix exists for).
- **Payment crash matrices**: committed-send resume (monitor
  resurrection, stays PENDING → settles), receive claim-binding crash
  (fails BACKWARDS — no boot lease — payer refunded, fresh invoice
  recovers), collect-binding crash (server re-admits), the stranded
  window (journal-then-cut, never re-payable), failed→same-invoice
  retry, open `Establishing` / close `Negotiating` park resumes.
- **Close under pending payments**, the **lease race** (reorg → fail
  backwards → payer retry), a **real sub-dust close side**,
  **chain-overrules-tombstone**, **quarantine** (postgres-seeded
  nonconforming caps → forwarding off + offender claims refused).

**The one product fix** the staging surfaced
(`server/src/channels/broadcaster.rs`): the capture broadcaster now
RELAYS non-close-candidate broadcasts to bitcoind. LDK claims HTLCs on
COUNTERPARTY commitments through its internal onchain path (fee from
the claimed output, no HTLCResolution event) — those were captured and
dropped, so the server never claimed on-chain. Close candidates
(funding-outpoint spends) stay captured.

**Seams.** Client, all `#[cfg(feature = "test-util")]`: open
establish/registration/feed holds, close negotiating hold, send-issue
hold, post-claim-binding hold, the claim-PASS hold (mailbox parks
cleanly — holding EVENT DELIVERY starves the chain feed), event
delivery hold, a payer CLTV-budget cap (kills LDK's randomized shadow
offset), and probes (pending-HTLC expiries, claim lease). Server,
config-gated (`dev_seams` + `test_crash_after_collect_binding`,
refused at startup on mainnet): admin force-close exporting the
CAPTURED commitment (fed by the otherwise-unused `ChannelClose` bump
event), admin cooperative close, the post-collect-binding abort.
`ChannelStatus` gains `pending_htlcs` (the deterministic observable).

**Harness:** captaind reserves its RPC ports ONCE — a restart returns
on the same addresses (the locks were process-lifetime anyway;
re-picking each start leaked them and stranded every configured
client). The seamed server pins `htlc_floor_profile = 100` over the
harness worst-case floor of 80.

## The rounds

- **r1 (FAIL, 5×P2 + 2×P3):**
  - **P2 seams ungated** — `test_hold_events`,
    `test_hold_claim_binding`, `test_hold_manager_persist` shipped in
    production builds → the first two are now `#[cfg(test-util)]` end
    to end (fields, construction, driver event-hold wrapper with a
    pass-through non-test build, handler threading, check sites); the
    third is DELETED (dead, no caller — also r1 f7). NodeManagerStore
    is a plain pass-through again.
  - **P2 dev seams production-reachable** — `dev_seams` /
    `test_crash_after_collect_binding` are plain deserializable config
    → the channels subsystem now REFUSES to boot on `Network::Bitcoin`
    with either set (an invariant beside the other boot guards; a
    server test-only feature was judged heavier than one refusal for
    two knobs — noted for ratification).
  - **P2 open convergence not proven** — `drive_open_to_done` reads
    checkpoint disappearance, which a FAILED action also produces → a
    new `assert_open_successful` polls the pollable status and requires
    `Successful`, run in every happy open and every convergence arc
    (the previously-unused `action_id`s now consumed).
  - **P2 mid-exit coop close too loose** — accepted any outcome/state
    → now asserts `close_cooperative` AND drives the exit to
    `ChannelClaimed` through the captured cooperative closing txid.
  - **P2 concurrent-round race weak** (partial): codex's ordering read
    was wrong (the round precedes the confirmation loop; no block mined
    since the restart), but the comment now states the
    no-block-mined invariant and that the coin-exclusion itself is a
    unit-level assertion, the e2e being the live interference run.
  - **P3 broadcaster doc stale** ("capture, never relay") → rewritten
    to the capture-the-close-shapes / relay-the-claims contract,
    pointing at the candidate-capture database tests.
- **r2: PASS**, no residual findings.

## Key decisions for review (Greg)

1. **Dev seams refused on mainnet at startup, not compile-gated.** The
   server crate has no test-only feature; adding one for two knobs
   would fork the config struct, deserialization, and admin wiring.
   The mainnet boot refusal removes the exposure in one place. Flag if
   you want the heavier feature gate instead.
2. **An open converging through a close/exit-owned terminal finishes
   its movement `Successful`, not `Failed`** — the channel existed and
   its value is accounted (`assert_open_successful` asserts exactly
   this). `Failed` is reserved for opens that never produced a channel.
3. **The broadcaster relay is the one product change in a
   test-completion commit** — it is a real money-safety fix (dropped
   on-chain claims) that only a full force-close e2e could surface;
   folded here rather than parked because the vector that needs it
   ships here. Byte-identical to the capture/relay split the database
   tests already cover.
4. **The `htlc_deadline` rung is staged INBOUND** — an outbound HTLC's
   absolute expiry rides LDK's randomized shadow-CLTV offset and is
   not a deterministic mining target; the inbound obligation's floor
   is pinned.

barkd suite unaffected (the 2 CLN recovery vectors still fail at
lightningd spawn — CLN not installed, environmental).
