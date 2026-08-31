# ARK #8 stage 1 — security remediation: what shipped

The ratified dispositions in `2026-08-28-channel-security-handoff.md` are
BUILT. Every fix is folded into its introducing commit, adversarially
reviewed, and mutation-validated. This is the record of what changed and
what the process caught.

Stack: `ark8-channels-stage1`, tip `5c245e272` (19 commits, unpushed).
Fold homes below are that stack's hashes.

## Root cause A — the client relays LDK's self-funded claims

GAP 1 (justice unwired), GAP 4 (HTLC claim off a counterparty
commitment), ex-GAP 8, GAP 9 (coop-close never reselects a foreign
winner).

The client's `QueueBroadcaster` captured every LDK broadcast and relayed
nothing, so a justice penalty on a revoked commitment and an HTLC claim
on a counterparty's commitment were both built, queued, and dropped. The
broadcaster keeps its capture-only posture (its fail-closed durability is
the point); a new daemon duty `relay_channel_broadcasts` drains the
queued NON-close-candidate rows and relays them. The split is the
funding-outpoint join: a close candidate leaves through the exit
machinery, everything else is LDK's own claim and is lost if unsent.

`ExitChannelCooperativeClosingState` now re-asks `obtain_commitment` and
adopts a confirmed foreign winner instead of rebroadcasting a doomed
closing forever. `ChannelSweepKind::Justice` — defined, never constructed
— is deleted: justice recovery rides the monitor's opaque revoked-output
balance, which terminal accounting already refuses to terminalize past.

Homes: relay duty ⇒ `c3e9c5399` (HTLC claim funding); Justice deletion ⇒
`57c6dfb0d` (the channel exit); GAP 9 reselect ⇒ `10c11a546` (the
cooperative close); seams and vectors ⇒ `1be002bdf` (lifecycle e2e).

Codex (xhigh): 8 findings, all fixed. Two were fund-loss: a stamped-then-
evicted claim was suppressed forever (a re-capture now re-arms it), and
the relay was gated on the wallet sync (a wallet outage must not hold
justice off the network while a cheater's delay matures).

## Root cause B — the server sweeps its own outputs

GAP 2. The server had NO `Event::SpendableOutputs` handler at all: every
channel resolving on-chain matured outputs it owned — above all its
`to_remote` on a client's commitment, which happens on ANY ordinary
client exit — and nothing swept them. Deterministic abandonment.

Shipped: the event arm (persist-before-ack, replay on failure), V64
`channel_spendable_output` and its DB layer, and `sweeps.rs` — one
attempt per output built through LDK's own spender and persisted before
broadcast, retired only at `DEEPLY_CONFIRMED`, replaced by RBF when fees
outrun it (superseded txids retained, since any may be the one that
confirms), and individually-uneconomical outputs swept TOGETHER in one
fee-sharing transaction. A 60s duty drives it.

Home: product ⇒ `8e722ba4e` (the claim seam — where the server's on-chain
claim machinery was born and whose `ClaimFunder` the sweeper reuses);
vector ⇒ `1be002bdf`.

Codex: 6 findings, all fixed — one-confirmation retirement was
reorg-unsafe, a near-dust descriptor could persist a ZERO-OUTPUT
transaction as its only attempt, build-once had no fee-bump path,
per-output economics stranded batch-profitable outputs, the test asserted
aggregate balance rather than provenance, and `>=` re-abandoned every
tick.

## Root cause C — the server resolves its own contract on-chain

GAP 3 / WD-16. A client can actualize the bridge and vanish; nothing then
settles the channel (the server cannot cooperatively close, and the
client's exit owns the unroll). The server's balance and HTLCs were
locked forever, recoverable only through a mainnet-refused dev seam.

**The rule, after two wrong versions:** when LDK has force-closed a
channel — an HTLC deadline, the stalled-channel policy, an operator, or a
protocol fault — the commitment is PUBLISHED as soon as the funding
output is canonically on-chain. Nothing else. Funded by an anchor CPFP
from the rounds wallet and submitted as a package.

Two gates were tried and removed:

* Gating on `confirmation_fed_at` was UNSOUND — that latch records
  feeding LDK a *reconstructed* bridge at the backing's anchor, is true
  while the real bridge is unbroadcast, is per-process, and survives a
  reorg. Publishing on it would submit a commitment whose input does not
  exist, locking wallet coins behind a package that can never confirm.
  Replaced by a chain query (`gettxout`, no mempool, fail closed).
* Gating on the peer being disconnected was WORSE. Connectivity says
  nothing about an adversary's intent, and LDK removes the channel from
  `list_channels()` the instant it force-closes — so the retry loop never
  walks it again and a commitment skipped once is stranded permanently.
  Greg's framing settles it: the server is resolving its own contract, so
  it should not be guessing what the counterparty will do.

Also: the stalled-channel policy (peer unreachable ≥ `stalled_close_
after_blocks`, default 1008 ≈ a week, balance above a floor, durable
block-height liveness so a restart neither forgets nor invents absence),
and the admin RPC graduated from dev seam to operator trigger reporting
`funding_on_chain`.

Home: all of C ⇒ `1be002bdf` — that commit introduced the server
force-close surface (`dev_seams`, the commitment stash, the admin RPC)
that C corrects, so it is the earliest commit where C's behavior can
exist.

Codex: 6 findings, all fixed, including the unsound gate above.

## R3 — knowing you can afford your own exit

Not the durable earmark the handoff sketched. Greg: the answer is
re-derivable, so storing it is a cache that goes stale exactly when it
matters. What shipped instead:

* the open-time reserve is priced at `fast` (the rate an exit really
  pays — `default_exit_fee_rate`, "exits are time-critical") instead of
  `regular`, and measured against CONFIRMED coins, because a CPFP child
  cannot spend unconfirmed change;
* each channel's status carries `exit_funding` — estimated exit cost and
  whether the wallet covers it — COMPUTED ON READ.

Manual-sync needed no work: channels + `daemon_manual_sync` is already
refused at startup with a loud error, not silently disabled.

Home: `1be002bdf` (it depends on `balance_confirmed`, added there).

## Accepted and documented

R1 (HTLC fee-race at the CLTV boundary, bounded per channel, CSV fix is
stage-2) and R2 (expiry fee-race — losing it costs the WHOLE channel
VTXO, mitigated operationally by the `F` exit lead) are written up in
`doc/channel-payments.md`, together with the `exit_funding` surface and
an operator note this work produced: a rounds wallet consolidated into
one UTXO can have round funding blocked while a close package is
unconfirmed, because the anchor child locks the coin it spends.

C1: `08-channels.md` corrected at both sites — the designated type is
`zero_fee_commitments` with `option_scid_alias` and NOT
`static_remote_key`, which the zero-fee type does not carry.

Punted as agreed: GAP 5 (reorg coverage), dust, P2A/Esplora.

## What the process caught

FIVE illicit test passes, each found by mutation-testing rather than
review:

1. GAP 4's "the HTLC output was spent" — satisfied by the SERVER's
   timeout claim, i.e. by the client LOSING the payment.
2. B's "the server's balance grew" — satisfied by incidental income.
3. B's provenance gap (codex): growth is not proof the money came from
   the sweep.
4. C's first vector passing in 64s — the client's own commitment had
   already landed, so it re-tested B.
5. C's housekeeping mutation passing — the client daemon (2s tick) was
   still running, which ALSO explained why housekeeping looked redundant.

THREE real product bugs, found by tests rather than by reading: the close
package's commitment input being locked as a wallet coin (every relay
failed), the unsound `confirmation_fed_at` gate, and `submitpackage`
success being read from the RPC call rather than `package_msg`.

TWO regressions the FULL battery caught after per-fix testing had passed:
a mis-implemented codex recommendation (refusing a pre-bridge operator
force-close broke the sanctioned `peer_close` path) and the over-eager
relay starving round funding.

Standing practice this establishes: assert RECOVERED ON-CHAIN VALUE
against values derived from the actual transactions — never a hardcoded
sat floor — tie it to an outpoint only the intended party can spend, and
mutation-test every fix by disabling it and requiring the vector to fail.

## Verification

19-commit cascade: production build, crate lib tests, bark-channels and
server-channel e2e at every commit; full battery (workspace units + the
whole bark-sdk suite) at the fold homes and the tip. Units 635/635,
bark-channels 23/23, server-channel 22/22, bark-sdk 77/83 — the 5
failures are the known CLN/bip321 environmental ones (lightningd cannot
start here), plus one confirmed load flake that passes in isolation.
