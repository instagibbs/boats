# Codex review record — MR-7 c4: the ArkRpcProxy adversarial sweep

**Commit:** `28c85fff2` "testing: the channel gRPC adversarial sweep
through a proxy" (amended through the rounds), on c3 `a1efa93fe`,
branch `ark8-channels-stage1-payments`.
**Verdict: PASS in 3 rounds**, no residual findings. All 4 vectors
green in the default drive mode AND under `BARK_DOUBLE_DRIVE_ACTIONS=1`;
the bark-sdk channels module (19/19) green after the shared-helper
refactor.

## What shipped

`testing/tests/bark_sdk/channel_proxy.rs` — 4 vectors driving the
channel open/close through an interposing `ArkRpcProxy`:

- **RG-4 idempotent re-upload + re-cosign SCID stability**: a
  register-duplicating proxy (byte-identical re-upload accepted) and a
  cosign-duplicating proxy (the `bridge_scid_tx_index` STABLE across
  the duplicate — the server-global allocation is per-channel-fixed;
  the re-cosign is re-signed under a FRESH nonce, per `raced_bridge`).
- **WD-15 lost registration response**: the register response is
  dropped AFTER the server committed; the client's byte-identical
  retry completes the open. The downgrade group on close likewise
  survives a lost response → downgraded.
- **IB-1 tampered attestation**: a corrupted ARK #5 attestation is
  refused; the cosign refusal is terminal → the open resolves FAILED,
  no channel reaches ready.

Every mutating proxy is armed only AFTER board+maintenance, matches its
target registration by CONTENT (a channel-funding VTXO for the open;
the exact retained leaf bytes for the close), uses a `drive_factor()`
loss/tamper budget (both reentrant calls of a step behave identically),
and records that it actually fired — each vector asserts it.

Routing: `channels_client_through` overrides the wallet's configured
`server_address` to the proxy URL; the board rides the direct captaind.

## The rounds

- **r1 (FAIL, 1×P1 + 3×P2):** the tamper proxy diverged under
  double-drive (first call tampered, reentrant call not) → tamper the
  first `drive_factor()` cosign calls; the loss/corruption proxies were
  armed during setup so board+maintenance consumed their budgets → arm
  after setup + track the action; the tamper/SCID asserts accepted a
  no-op → record and assert the mutation.
- **r2 (FAIL, 1×P1 + 3×P2):** the P1 double-drive fix was incomplete on
  the loss proxy (lose-once diverged the reentrant calls) → a
  `drive_factor()` budget; **finding #2 was decisive** — flipping a
  vtxo's last byte changes its outpoint→id, so the server rejects at
  LOOKUP not signature validation, and the ark API doesn't cleanly
  expose witness mutation through a proxy → the corruption vector was
  REMOVED (RG-4 atomicity is proven directly in the server suite,
  `downgrade_split_admission_and_group_registration`); the loss race
  (a setup registration consuming the budget) → CONTENT MATCHING
  (channel-funding VTXO / exact leaf bytes), never call counting.
- **r3: PASS**, no findings.

## Key decisions for review (Greg)

1. **RG-4 group atomicity is NOT re-staged through the proxy** — it is
   covered directly in the server suite (which constructs malformed
   levels precisely); a proxy can only mangle bytes, and a byte flip
   changes the vtxo id (a lookup miss, not a validation failure), so a
   proxy re-stage adds no faithful coverage. Stated in the module
   header + commit.
2. **The re-cosign proxy asserts SCID stability, not a byte-identical
   response** — the shipped rule (`raced_bridge`) re-signs a duplicate
   cosign under a fresh nonce; idempotency lives at registration. The
   proxy asserts the load-bearing invariant (the virtual-funding SCID
   is per-channel-stable), recording that it compared a PRESENT SCID.
