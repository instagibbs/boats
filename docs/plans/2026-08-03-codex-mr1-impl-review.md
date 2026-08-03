# Codex review: MR-1 implementation series (2026-08-03)

**Subject**: the 3-commit protocol-surface series in bark-stage1
(`ea33bbf4..fd3b87a9` at review time; `df2225c0` / `838419f5` / `c01c9b0e`
after the fixes below were folded in — bookmark
`ark8-channels-stage1-protocol`).
**Reviewer**: codex (read-only, high reasoning), prompted against the MR-1
design note §2–§6 and the normative spec (02-vtxo.md, 08-channels.md).
**Verdict**: REWORK — no Critical findings; 3 Important, 1 Minor.
**Disposition**: all four findings verified substantively correct and fixed
same-day; fixes folded into their home commits. Residuals recorded below.

## Findings and disposition

### F1 (Important) — stored delegated-round rows consumed without policy revalidation — CONFIRMED, FIXED

Delegated participations are policy-checked only at registration
(`register_delegated_round_participation` → `validate_payment_amounts`).
When a round later consumes the STORED rows, `process_delegated_participation`
re-checks amounts/capacity/spendability/exit state but never policy — and the
rows are decoded from the database (where `0x08` now decodes). No current
writer can store a channel row, but the invariant must hold at the
construction boundary, not rest on trusting every past writer — the exact
whack-a-mole class the G1 rounds killed.

**Fix**: extracted `validate_round_input_policy` / `validate_round_output_policy`
(the exhaustive matches now live there), called from `validate_payment_amounts`
(both RPC paths, behavior-identical) AND from `process_delegated_participation`
right after the stored rows' inputs are fetched, before
`register_delegated_participation`.

**Residual**: the consumption-point call cannot be exercised by a test —
no writer can store a channel row (the RPC validates first), and the round
state machine is not constructible in unit tests. Covered by the shared
validators' tests + the three-line call site.

### F2 (Important) — bark wallet admits channel VTXOs as generic spendable balance — CONFIRMED, FIXED

Mailbox ingestion (`process_raw_vtxos`) accepted any chain-valid VTXO and
stored it `Spendable`; seed recovery matched by `user_pubkey()` (which a
channel-funding policy has) and stored it spendable; generic selection then
counts/locks it and flows fail only at build time — a malicious server or
mixed-version peer could wedge balance accounting.

**Fix**, three layers:
- `Wallet::store_vtxos` (the single wallet-side persistence funnel — all
  wrappers and recovery go through it) hard-refuses a channel-funding VTXO
  with an exhaustive match: it is unrepresentable as a generic wallet VTXO.
- Mailbox `process_raw_vtxos` filters it into the invalid list (per-VTXO
  tolerance: one bad VTXO no longer aborts the receive batch).
- Recovery `resolve_owned_vtxos` reports it failed (not recoverable as
  ordinary balance) instead of admitting it.

**Residual**: no wallet-level test — constructing a `Wallet` in unit tests
is not supported, and no e2e path can hand the wallet a channel VTXO until
a mint path exists (a malicious-server harness is MR-5 territory). The
store gate is an exhaustive match at the single funnel.

### F3 (Important) — caller-level tests missing (helper-level tests don't pin the production call sites) — PARTIALLY FIXED, REST DOCUMENTED

Correct observation: the round/offboard tests call the shared helpers
directly and would survive removal of the production calls.

**Fix**: new e2e test `channel_funding_destination_refused_at_cosign_endpoint`
(testing/tests/server/mod.rs) drives a channel-shaped package — with an
HONEST client attestation over the channel destination, via the builder's
own `UpgradeOutput` — into a running captaind's `request_arkoor_cosign`,
and pins (a) the refusal (message names channel-funding), (b) no
spent-state mutation (the same input immediately completes an ordinary
send). This exercises the exact shared `validate_cosign_request` →
`from_cosign_request` entry the Lightning pay / receive-claim / revocation
endpoints also construct through.

**Not delivered, with reasons**:
- Round + delegated ENDPOINT tests: round attestations bind the output
  list and are verified BEFORE `validate_payment_amounts`, so a mangled
  submission dies at attestation verification — the policy gate is
  unreachable from the wire with a foreign proxy and un-forgeable with
  honest clients. Helper-level tests (both request shapes) + F1's
  consumption-point gate are the achievable coverage.
- Input-side endpoint tests (arkoor / offboard / round): impossible by
  construction — no channel-funding VTXO can exist server-side to spend,
  precisely because of the gates under test.
- LN-pay/receive/revocation endpoint variants: same shared entry as the
  delivered endpoint test; per-flow setup cost not justified in this MR.
  Flow-level adversarial coverage belongs to the surface/adversarial MR.

### F4 (Minor) — "deterministic" server cosign wording — CONFIRMED, FIXED

`musig::deterministic_partial_sign` samples fresh randomness; the name
refers to the first-signer-with-counterparty-nonces flow, not
reproducibility. Reworded the `server_cosign_bridge` doc, the roundtrip
test comment, and the commit message ("signs first against the user's
nonce with a freshly sampled nonce of its own"). Bridge tx/txid
determinism (witness-independence) is unaffected and still claimed.

## Verified clean (codex, summarized)

Exact 0x08 encoding + baseline rejection + decode/gate landing in one
commit; marker constant independently recomputed, script bytes and leaf
version exact; taproot internal key/two-depth-1-leaves/no-exit-leaf/65-byte
control block; channel≠board output keys with literal (non-recomputed)
vectors; board construction fixed to Pubkey; all arkoor callers pass None
with mutually-exclusive authorizations; self-signed + delegated
registration refuse before durable writes; offboard refusal before
session persistence; hArk paths downstream of round admission (no separate
bypass); bridge version/locktime/BIP-68 sequence/BOLT-3 sorted
P2WSH/zero-fee + P2A/Prevouts::All/SIGHASH_DEFAULT/tweak use/partial +
final verification/witness-independent txid; proto tags free and
presence-aware, half-present pairs reject, fields survive every package
transformation incl. the LightningPayHtlcCosignRequest path;
supports_channels tag 21 default-compatible and advertised false through
bark-json/OpenAPI; `git diff --check` clean, tab style consistent.

## Post-fix verification

- workspace nextest: 484 passed (483 + the new e2e compiled into the
  server suite; unit set 483/483 at the final tip)
- e2e vs real captaind/bitcoind-v29/postgres: arkoor/offboard/round/channel
  slice 33/34 + the new endpoint test PASS; the 2 known-environmental
  failures both die spawning lightningd (docker image absent locally)
- clippy -p ark-lib: clean; per-commit `cargo check --all --tests`: green
  at each of the three rewritten commits
