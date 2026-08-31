# Embedded stock LDK vs ldk-node/ldk-server — decision record

**DECIDED 2026-09-01 (Greg): keep the embedded-stock-LDK architecture and
keep the coverage.** The hand-rolled node plumbing is a modest, now
well-pinned cost ("maybe 1/5 of the code, attributable to a couple bugs —
not an outrageous cost to carry outside of ldk-node"), and the e2e suite
would be wanted under any node runner because it pins system outcomes in
an LDK usage mode upstream never tests. The only follow-up left open is
optional and cheap: file the missing ldk-node seams upstream as
Ark-independent issues so a future migration stays an option, not a
project.

Question examined: what minimal LDK-side changes would make direct
"LDK server" (ldk-node / ldk-server) usage possible for captaind, and how
much code/test cleanup would that buy — in particular, would the stage-1
test investment have been "free" given LDK server?

Sources: bark-stage1 @ `2b9b52704` (moving target — Greg was committing
test vectors live during the research; counts are that morning's
snapshot), rust-lightning fork @ `0a0a489d6` (`ark-ldk-bridge-modern`),
local ldk-server clone @ `3e9e537`, `rust-lightning/sdd/
ldk-upstream-requirements.md` (2026-07-13 audit), and three independent
test-classification passes that converged.

## 1. Where stage 1 actually stands

Stage 1 runs on **stock crates.io `lightning 0.2.4`** — no fork, no git
pin. `bark-channels` (1,154 lines) assembles a stock node around three
owned seams, and its own crate doc names them: manual ("virtual")
funding, application-fed confirmations at real chain heights, and a
broadcast seam that captures into the embedder's pipeline rather than
relay. The reimplementation is self-aware: `bark-channels/src/driver.rs`
says "mirrors `lightning-background-processor`" six times,
`bark/src/channels/mod.rs:1257` says "the driver runs no LDK
`OutputSweeper`; the exit machinery sweeps from the persisted rows," and
the server's sweep worker is literally named `ChannelsOutputSweeper`.

Every LDK primitive the design needs is a public rust-lightning API
(`unsafe_manual_funding_transaction_generated`, `chain::Confirm`,
`BroadcasterInterface`, manual accept, `anchor_zero_fee_commitments`).
The blocker is that **ldk-node does not forward them** — which is the
2026-07-29 rule-out (verified then against ldk-node 0.7.0): no
manual-funding passthrough, five closed preset chain sources (no manual
`Confirm` feed), no broadcaster seam, no manual/0conf accept or zero-fee
config, no raw `ChannelManager` access; only `KVStore` is pluggable.
One gap has since closed upstream: ldk-server `36df71e` exposes
zero-fee-commitment config, and stage-1's designated type
(`anchor_zero_fee_commitments` + `scid_privacy`) is pure stock bits.
ldk-server itself is a thin (~1.7k-line) gRPC wrapper over ldk-node's
`Builder` — it inherits whatever ldk-node exposes, plus proto surface.

## 2. The minimal ldk-node changes (if we ever wanted it)

Four builder seams; each individually small, each just forwarding a
trait rust-lightning already exposes (precedent: custom `KVStore` via
`build_with_store` got in exactly this way). Together they amount to an
"embedder mode," which inverts ldk-node's turn-key philosophy — that
negotiation, not the code, is the real cost.

1. **Externally-funded channels** — forward
   `unsafe_manual_funding_transaction_generated` (funder-only; the
   acceptor gates `channel_ready` by withholding the confirmation feed,
   no trusted-accept variant needed), plus a manual inbound-acceptance
   hook so Ark admission can gate the accept.
2. **Application chain-feed seam** — a manual `Confirm` source
   (confirm the bridge at its real anchor height with the synthetic
   tx_index for SCID minting; withdraw on reorg) coexisting with a real
   chain source. The deepest cut against ldk-node's model; no
   workaround exists.
3. **Broadcast interception** — injectable `BroadcasterInterface` or a
   capture-vs-relay policy hook with durable capture (content-matched
   close-candidate capture, poison-latch crash consistency).
4. **Event-ordering and output hooks** — the durable-ack barrier
   (persistence sequenced against event handling; what
   `feed_barrier.rs` pins and why the driver mirrors rather than uses
   `lightning-background-processor`), plus raw `SpendableOutputs` or a
   sweep-destination override (funds must land in the rounds wallet,
   not ldk-node's internal BDK wallet).

## 3. What migrating would buy — the accounting

**Code.** Server-reachable glue is ~9,470 lines. Deletable under
ldk-node-with-seams: the `bark-channels` assembly + driver (~860), the
server persist/db-executor/node/config plumbing + monitor CRUD
(~1,000), `server/src/channels/sweeps.rs` + the V64 lifecycle → LDK's
`OutputSweeper` (~340), plus event-loop shares — call it 2.5–3k, with a
similar-shaped share on the client. The Ark core (~3.6k: admission,
downgrade groups, parent-exit watch, SCID policy, claim locks,
stalled-recovery, the capture policy itself) stays under any runner.

**Tests.** 118 e2e vectors, ~12.5k LOC: 24 PLUMBING (~3,150) /
52 ARK (~5,650) / 35 MIXED (~3,700), plus the 19 `bark-channels`
release-contract/feed-barrier pins (~1,900) and a 1,735-line test-double
harness. Three independent classification passes converged on this
shape (sensitivity: ~7–8 boundary tests swing PLUMBING↔MIXED).

**Bugs.** The only two product bugs the whole coverage arc surfaced —
the client justice-relay regression and the server capture-drop of
LDK's internal HTLC claims — were both at the broadcast seam, i.e. the
component ldk-node would own. Both are now mutation-pinned.

## 4. Why we are NOT migrating

1. **The coverage was never going to be free.** The expensive vectors
   assert *system outcomes* — value provenance into the rounds wallet,
   justice inside the delay window, sweeps surviving crash+RBF+reorg —
   not whose code implements the sweeper. Ark's usage mode
   (never-confirming virtual funding, synthetic SCID positions,
   captured closes) is exactly what upstream's own suite never
   exercises, so "battle-tested for free" is weakest precisely where
   Ark lives. Under ldk-node these tests become *more* important, not
   redundant: regressions from version bumps or seam mis-wiring produce
   the same money-loss failures, and you lose the ability to read the
   implementation to convince yourself. Only the cheap
   internal-mechanics pins (persistence round-trip, event-pump
   equivalence, the feed-barrier doubles) would genuinely retire — and
   those are not the 200–900s vectors driving suite cost.
2. **The discovery cost is already paid.** The hard part was learning
   the requirements (justice must ride the relay path, close candidates
   need content matching, persistence must barrier against event
   handling). What remains is ~2.5–3k lines of stable assembly whose
   known failure modes are pinned.
3. **Control over a money-safety seam.** Capture-vs-relay with the
   poison latch and one-postgres-DB atomicity between Ark tables and
   LDK blobs is a stronger invariant than a custom KVStore behind
   ldk-node's store handle can give (same-DB yes, same-transaction no).
   Migrating trades a verified invariant for an API contract.
4. **Stage 2 makes ldk-node strictly worse for a while.** The CSV
   channel type and teleport need the rust-lightning fork until the
   6-PR decomposition in `sdd/ldk-upstream-requirements.md` lands;
   ldk-node pins upstream releases, so stage-2 development on ldk-node
   means forking both layers.
5. **The steady-state carry cost is the tests, not the code — and the
   pins are self-justifying.** The 19 release-contract/feed-barrier
   vectors are what let us take crates.io `lightning` bumps safely
   without a fork; that contract exists under any architecture.

## 5. Kept open

- **File the seam gaps upstream** (manual `Confirm` feed, injectable
  broadcaster, manual-funding passthrough, acceptance hook) as
  Ark-independent ldk-node issues — each is "expose what rust-lightning
  already exposes." If they land on upstream's timeline, migration can
  be re-priced later. Optional; no deadline.
- **Re-visit trigger:** if e2e suite wall-clock becomes the binding
  pain, note that the plumbing vectors are the long ones — ldk-node
  would buy relief there, not in LOC. Second trigger: stage-2's
  rust-lightning PRs landing upstream removes objection #4.
- The stage-2 upstreaming arc (the fork's 6-PR decomposition, teleport
  framed as the generic quiescent funding re-point primitive) is
  unaffected by this decision and remains the higher-leverage upstream
  conversation.
