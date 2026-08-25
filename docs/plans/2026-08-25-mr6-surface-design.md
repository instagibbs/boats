# MR-6 — surface + hardening (scoping/design note, pre-G1)

Per the stage-1 plan (§5, renumbered): REST `/channels`, bark-json DTOs,
CLI subcommands, barkd integration, the consolidated adversarial sweep
via `ArkRpcProxy`, and operator docs. This note scopes it against what
the arc has already built and proposes the commit cut. Code waits for
Greg's go.

## What already exists (verified 2026-08-25 at `cac12a382`)

- **Wallet-lib API complete**: `open_channel`, `close_channel`,
  `start_channel_exit`, `cancel_channel_exit`,
  `channel_opens_in_progress`, `sync_pending_channel_{opens,closes}`,
  `maintain_channel_deadlines`, `maintain_downgrade_watches`; the
  persister exposes `list_channel_records` with the full record state
  machine (`Cosigned … Closed`).
- **Daemon integration done**: the barkd daemon loop drives sync, the
  ungated chain-safety duties, deadline rungs and the close driver.
  barkd serves bark-rest, so a `/channels` router lands in barkd with no
  new binary work.
- **DTO toeholds**: `ArkInfo` channel fields in bark-json (cli + web),
  the movements subsystem's CHANNELS kind, `uneconomic_txs` etc.
- **Missing entirely**: any CLI noun, any REST route, any channel
  list/status DTO, the proxy-based adversarial sweep, operator docs.

## Proposed commits (each independently green, MR-4/5 rhythm)

**c1 — bark-json channel DTOs + REST `/channels`.**
Per-resource axum router in bark-rest matching the exits/boards
pattern (utoipa docs, `bark_json::web` request/response types,
synchronous handlers over the wallet API):
- `GET /channels` — list: channel id, state (the record state string),
  funding value, balances when known (from the LDK view for live
  channels; from the recorded close for closing/closed), backing VTXO
  id + expiry height.
- `GET /channels/{channel_id}` — one record, plus close-phase detail
  (retained-split presence, response settlement, exit state) for
  operators debugging a close.
- `POST /channels/open` — amount + node address; returns the channel id
  and the funding VTXO id. Synchronous like `board-amount` (the wallet
  API is synchronous through cosign; readiness is then observable via
  GET).
- `POST /channels/{channel_id}/close` — initiates the cooperative
  close; state advances via the daemon; progress observable via GET.
- `POST /channels/{channel_id}/exit` and `/exit/cancel` — the unilateral
  path, mirroring the wallet API. (Decision below: these live under
  `/channels`, not `/exits` — the exit is channel-scoped and its cancel
  has channel-specific refusal rules.)
- openapi.json + bark-rest-client regeneration (the plan's G0 review
  flagged exactly this artifact set for this MR).

**c2 — CLI `bark channel …`.**
Noun `channel` (plan §9 naming): `open`, `close`, `list`, `exit`,
`cancel-exit`, JSON output via the same bark-json types the REST layer
uses. Thin: parse, call the wallet API, print.

**c3 — the consolidated adversarial sweep (`ArkRpcProxy`).**
e2e vectors the per-MR suites deliberately deferred, driven through the
proxy so the SERVER's behavior is asserted against a byzantine client:
- **RG-4**: byte-identical re-upload of a registration is idempotent;
  a partial upload leaves the operation unregistered (replay both for
  the upgrade transfer and the downgrade split group).
- **IB bypass attempts**: cosign requests violating IB-1/2/3/5/6
  shapes at the proxy layer (tampered attestation, spent/banned input,
  exiting input, unbalanced destinations, package-atomicity break) —
  most have unit/vector coverage; the sweep asserts the LIVE gRPC
  surface refuses them, not just the library.
- **WD-15 crash matrix, remaining cells**: MR-4/5 covered
  crash-at-PONR and exit resume. Remaining client cells: crash between
  cosign and registration upload (upgrade AND downgrade), crash between
  close capture and record flip, crash mid-watch-response (the
  associate-before pattern's window). Remaining server cells: crash
  between admission commit and response (the client's retry must be
  idempotent — pairs with RG-4). Each cell = kill at a debug seam,
  restart, assert the terminal state matches the no-crash run.
- Explicitly OUT (recorded residuals): sub-dust end-to-end and the
  chain-overrules-tombstone rescue e2e (payments-slot); the
  hostile-anchor-child e2e rides with the upstream submission of the
  fee-discipline commit.

**c4 — operator docs.**
Stage-1 profile statement (what the embedded node does and refuses),
forwarding posture (plan §3.7: forwarding ON when channels are enabled, capped and kill-switched, intra-ark by construction),
`vtxo_exit_delta` guidance (WD-1: comfortably larger than worst-case
respond-and-confirm), and the expiry-treadmill UX note (refresh keeps
the channel alive; the deadline rungs close it when the operator does
not).

## Design decisions to ratify (Greg)

1. **Channel exits under `/channels`, not `/exits`** (proposed): the
   generic exits surface stays VTXO-keyed; a channel exit is
   channel-id-keyed with channel-specific cancel rules. The generic
   `/exits/status` continues to show the underlying exit VTXOs.
2. **Synchronous open over job-style**: `open_channel` returns after
   cosign+registration (seconds); LDK readiness is a state the GET
   reports. Matches the CLI's UX and the boards pattern.
3. **State exposure**: the record state string verbatim
   (`negotiating`, `registration_pending`, `downgraded`, …) rather than
   a simplified public enum — operators debugging a close need the real
   phase; the strings are already stable identifiers in the DB.
4. **No pagination/filtering on the list** (stage 1: few channels).

## Conformance mapping

RG-4 → c3; IB-1..7 (live-surface) → c3; WD-15 remaining cells → c3;
WD-1 guidance → c4; §3.7 posture → c4; the plan's G0-review artifact
requirement (openapi + generated client) → c1.

## Test plan

c1: REST e2e in the existing bark-rest test style (open → list → close
→ observe downgraded/closed; exit → cancel refusal past the bar).
c2: CLI smoke in the bark suite (JSON output parses into the DTOs).
c3 is itself the test commit. c4: docs only, link-checked.
