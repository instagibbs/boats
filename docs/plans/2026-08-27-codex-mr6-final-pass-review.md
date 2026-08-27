# Codex review record — MR-6 WHOLE-MR FINAL PASS (+ the final-fixes commit)

**Scope:** the entire payments MR as one unit, `git diff f13336286..HEAD`
on `ark8-channels-stage1-payments` (bark-stage1; its own bookmark on top
of `ark8-channels-stage1-close`, which stays at its pushed tip
`f13336286`). Six commits converged
individually (c1 feed/drain barrier, c2 watch feed, c3 HTLC claim trust
design, c4 payments state/policy, c5 enablement, c6 payments e2e); this
pass hunted CROSS-COMMIT seams the per-commit reviews, each scoped to
its own diff, could not see.

**Verdict: PASS in 7 rounds.** The findings were developed as one
working final-fixes commit (golden tree `54daa5871`, the shape the PASS
was issued against), then FOLDED into their home commits at Greg's call
for clean history — final stack c1 `33dc9f332` / c2 `ab7652969` / c3
`8efd74533` / c4 `38eb3785f` / c5 `2659855aa` / c6 (tip) `a491cea7f`,
tip tree byte-identical to the golden tree. The fold attribution:
profile-coherence + defaults → c4; the whole server claim-funding
machinery → c5; the SCID allocation move + V62 carry-over → c6. Every
FAIL round's findings were fixed (none refuted, none deferred). One
design decision was escalated to Greg mid-arc (see WD-3).

## Round 1 — the four cross-commit findings (all fixed as c7's core)

1. **P1: the server dropped its own HTLC claims.** `HTLCResolution`
   bump events fell through a catch-all and the close-capture
   broadcaster never relays. Scenario: captaind forwards A→B, pays B,
   A wedges the fulfill; LDK force-closes, the client unrolls, the
   commitment confirms — and every on-chain success claim was ignored
   until A's timeout path took back money the server had already paid
   downstream. Fix: `server/src/channels/claims.rs` — LDK's stock
   `BumpTransactionEventHandler` over LDK's own `Wallet` adapter,
   sourced from the ROUNDS wallet, captured then RELAYED for real
   (claims spend an already-actualized commitment; WD-16 untouched —
   `ChannelClose` CPFP events stay deliberately unhandled: the
   commitment rides the CLIENT's exit CPFP).
2. **P1: SCID exhaustion after the sacred commit.** The V62 allocation
   ran after the admission commit; exhaustion left a committed-cosign
   channel that could never receive its index. Fix: allocation moved
   BEFORE the commit block — exhaustion is a clean pre-commit refusal
   (a refused open leaves one dead index in a 16.7M band).
3. **P1: V62 destroyed existing SCID allocations.** Fix: rename +
   `INSERT SELECT` + drop preserves rows.
4. **P2: contradictory shipped defaults admitted no channel**
   (worst-case floor 100+144+18 = 262 > profile 144, and validation
   only checked profile > slack). Fix: startup computes the worst-case
   floor from the ACTUAL configured triple (max exit depth + exit
   delta + claim slack) and refuses a profile below it; default
   profile 288; both timing knobs got serde defaults and the config
   template documents all channel keys (round 2 no. 5 folded here).

## Rounds 2–7 — hardening the server claim path

The new claims module absorbed six more rounds. Chronology of what
each round's findings changed:

- **r2 (5 findings):** the signing crux — `finish_tx` requires a
  FINALIZED psbt, but LDK adds the HTLC witnesses only AFTER
  `sign_psbt` returns (bump_transaction/mod.rs:1137→1139), so no claim
  could ever be built → new `PersistedWallet::sign_owned_inputs`
  (sign wallet inputs, ignore the finalized flag, extract unchecked —
  the implied feerate of a deliberately-unfinalized psbt is
  meaningless). Claim inputs invisible to concurrent rounds → guards
  through the SAME `locked_outputs` registry round funding consults.
  +5 WU satisfaction-weight correction (witness count byte + empty
  scriptSig length byte). Immature coinbase screened from the source.
- **r3 (3 findings), all in the guard design:** same-claim rebuilds
  screened out their OWN locked coin (one-coin wallet = claim starved
  forever) → the source's per-invocation `exempt` set; guard
  replacement released a superseded-but-mempool build's coins →
  guards ACCUMULATE per claim, released together; pruning only ran
  inside the claim path → its own 10-minute tick.
- **r4 (2 findings):** silently SKIPPING a concurrently-spent wallet
  input → strict classification (every non-HTLC input must be a
  lockable unspent wallet coin or the whole relay aborts); guards
  don't survive restart → first fixed with a `gettxspendingprevout`
  mempool reconcile, which Greg REJECTED on principle (WD-3) —
  replaced by committing accepted builds to the wallet (WD-4).
- **r5 (2 findings):** commit-to-wallet alone is NOT enough across a
  restart — BDK canonicalizes to the NEWEST build, exposing a
  superseded (but still minable) build's exclusive coins as unspent →
  the accumulated guard set became DURABLE (WD-5). The N-block
  eviction-unwind lag was confirmed conservative (safe direction) and
  accepted.
- **r6 (1 finding):** the durability gate only fired when the
  in-memory set GREW — a failed upsert could be bypassed by a
  same-coins rebuild → the FULL set is upserted before EVERY relay.
- **r7: PASS.** Empty-set builds confirmed unreachable-or-harmless
  (anchor HTLC outputs pay no fee, so builds always add fee inputs;
  a hypothetical zero-wallet-input build has nothing to protect).

## Key decisions for review (WD)

- **WD-1 — the server claims its own HTLCs, with LDK's machinery,
  funded by the rounds wallet.** Mirrors the client's c3 trust design:
  the monitor is the durable claim state (re-emits per block), so the
  funder persists no claim state of its own and every failure is
  transient by construction. Relay failures are DEBUG-level: a claim
  spending a not-yet-unrolled commitment is the normal early shape.
- **WD-2 — close-side asymmetry stands.** `ChannelClose` bump events
  are ignored (the commitment rides the client's exit CPFP); ONLY
  `HTLCResolution` is funded and relayed — the single place the server
  must put its own sats on chain or lose forwarded value.
- **WD-3 — Greg's constraint, mid-arc: NEVER inspect the mempool for
  safe operation.** The r4 restart fix was first implemented as a
  `gettxspendingprevout` reconcile; Greg stopped it ("we should
  *never* require looking at the mempool for safe operation") and it
  was deleted before ever being committed to a reviewed round.
- **WD-4 — accepted claim builds are committed to the rounds wallet**
  (`commit_tx` + persist), exactly like rounds/offboards/send record
  their in-flight spends. The wallet's EXISTING mempool-eviction sync
  releases the coins when a build is replaced or dropped (a claim
  spends the foreign HTLC input, so it is never a "fully owned" tx
  exempt from eviction). Never-accepted builds commit nothing — they
  cannot conflict with anything after a restart.
- **WD-5 — the accumulated claim-input locks are DURABLE**
  (`channel_claim_input_locks`, V63: claim_id PK, outpoints TEXT[],
  refreshed). Rationale: commit-to-wallet cannot protect a SUPERSEDED
  build's exclusive coins (BDK canonicalizes to the newest build) while
  that build may still be minable from peer mempools. Ordering
  discipline: the FULL set is upserted before EVERY relay (failure
  aborts the relay); liveness is re-upserted up front per bump event
  (nonfatal — heals failed/pruned rows); startup re-arms all rows
  within the horizon into the lock registry (fatal on failure — it
  runs before anything can select coins, so an untakeable lock is a
  startup-order bug); a 10-minute tick prunes in-memory + DB entries
  past the 6-hour staleness horizon (a live claim re-emits per block;
  stale = resolved).
- **WD-6 — guards accumulate, never replace.** A superseded build may
  still sit in a mempool spending its coins; releasing on replacement
  would hand a round a coin whose conflicting spend is already
  relayed. Everything releases together at staleness (or earlier via
  wallet eviction for accepted builds).
- **WD-7 — abort-don't-skip.** Any claim-tx input that is neither an
  event HTLC outpoint nor an already-guarded carryover MUST be a
  lockable unspent wallet coin, or the whole relay aborts (the monitor
  retries next block). Relaying past a concurrently-spent input would
  race the concurrent spend for the coin.
- **WD-8 — the same-claim exempt set.** LDK reuses a claim's own coins
  across RBF rebuilds; the source exposes the handled claim's locked
  outpoints (and only for the duration of one `handle_event`; events
  are serialized in the driver) so a one-coin wallet cannot starve its
  own claim. Cross-claim reuse stays barred by LDK's internal
  per-claim UTXO locks.
- **WD-9 — `sign_owned_inputs` on `PersistedWallet`**: the wallet
  signs only its inputs and does not require finalization — the LDK
  handler adds HTLC witnesses after. Extraction is
  fee-rate-unchecked because empty HTLC witnesses skew the implied
  feerate of the intermediate psbt; the feerate discipline is LDK's.
- **WD-10 — change = peeked (not revealed) internal address at index
  0.** LDK builds and discards candidate transactions; revealing per
  attempt would gap the keychain for nothing. The accepted reuse is
  between the server and itself. Conceded as privacy/accounting-only
  in review.
- **WD-11 — default `htlc_floor_profile` 288** (was 144), with startup
  coherence validation against the configured worst-case floor, and
  serde defaults so pre-existing configs keep deserializing.
- **WD-12 — accepted residuals:** the 6-hour staleness horizon (a
  resolved claim's coins stay locked up to 6h past its last event; a
  6h+ block drought could release a live claim's locks — self-healing
  on the next re-emit); the N-block eviction unwind for N superseded
  builds (conservative direction — coins stay unspendable too long,
  never spendable too early); wall-clock DB staleness vs monotonic
  in-memory staleness (same horizon, seconds of skew, harmless).

## Battery

Every working-commit amend during the rounds ran the full battery
(`cargo build --workspace` + workspace `--lib` units + bark-channels +
bark-sdk channels + server channel suites) — ALL GREEN each time,
including on the golden tree `54daa5871`. After the fold: per-commit
batteries on the rewritten c4 `38eb3785f` and c5 `2659855aa` (the only
new intermediate trees; the tip is byte-identical to the golden tree).
