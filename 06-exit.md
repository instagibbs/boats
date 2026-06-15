# ARK #6: Emergency Exit

The emergency exit lets a user move a VTXO's funds on-chain with no
cooperation from the server or anyone else. It is the safety property that
makes every other part of the protocol trust-minimized: any VTXO that passes
ARK #2 validation can be exited by broadcasting its genesis chain.

An exit is strictly more expensive and slower than cooperative alternatives
(refresh, ARK #4; offboard, ARK #7). Users SHOULD use it only when the server is unavailable
or adversarial, or when forced (e.g. a withheld unlock preimage, ARK #4).

## Preconditions

A VTXO is exitable when:

* it passes full validation (ARK #2) — in particular, every genesis
  transition carries a valid signature, and any `hash-locked-cosigned`
  transition carries its preimage;
* its amount is at least `P2TR_DUST` (a smaller output cannot be relayed);
* its chain anchor transaction is confirmed (or can be made to confirm —
  for a board, the user controls broadcasting the funding transaction);
* the current height plus the exit confirmation time leaves room before
  `expiry_height` — once the expiry leaves become spendable the server can
  race the exit for any not-yet-broadcast portion of the chain. Users MUST
  begin exits sufficiently before expiry.

A VTXO from an unfinished round (missing leaf cosignature or preimage)
cannot complete its exit; see ARK #4 for the swap-failure resolution, which
relies on broadcasting the *completable* prefix of the chain.

## Procedure

The exit transactions are exactly the transactions derived from the genesis
chain (ARK #2): each is `nVersion = 3` (TRUC) with a single pre-signed input
and a P2A fee anchor.

1. **Registration.** The user registers the VTXO for exit and persists the
   full transaction chain. From this point the wallet MUST NOT spend the
   VTXO off-chain.
2. **Broadcast, root first.** Transactions MUST be broadcast in chain order
   (each spends an output of the previous). Because the transactions pay no
   fee themselves, each broadcast generally requires a **CPFP** child
   spending the transaction's P2A anchor (plus wallet inputs) paying for the
   package. Under TRUC/v3 relay rules, a v3 transaction and its anchor-spend
   child relay as a package (1-parent-1-child topology); the anchor is
   spendable by anyone with an empty witness (the anchor output costs 13 vB;
   spending it adds 1 WU of witness). The same topology limits mean each
   level can enter the mempool only after the previous level has
   **confirmed**; a whole chain cannot sit unconfirmed at once. The 1p1c
   limit also means each unconfirmed exit transaction needs its **own** CPFP
   child: a single child cannot fee-bump two unconfirmed exit transactions
   at once (it would have two unconfirmed v3 parents), so sibling exits at
   the same depth — e.g. several leaves of one round tree — are bumped
   independently.
   * A signed exit transaction with one non-anchor output and a key-spend
     witness weighs 124 vB (`EXIT_TX_WEIGHT`); script-spend transitions
     (`hash-locked-cosigned`) are larger. Fee estimation for a level is the
     level's actual transaction weight plus the CPFP child size, paid by the
     child (and by the anchor's own `fee_amount`, when non-zero — e.g. the
     board fee, ARK #2).
3. **Fee-bumping.** While unconfirmed, the user SHOULD monitor the package
   and fee-bump (replace the CPFP child) as needed; in particular, a
   third-party spend of the anchor that pays too little SHOULD be replaced.
   If a transaction is evicted or never confirms, it is rebroadcast; the
   pre-signed chain never expires (all inputs use `nSequence = 0` and
   `nLockTime = 0`).
4. **Wait out the relative delay.** Once the final exit transaction is
   confirmed, the VTXO's output exists on-chain, but its unilateral spending
   path is a delayed clause: the user must wait the clause's relative-timelock
   delta (relative to the confirmation) before the claim input is valid. For
   the `pubkey` policy this delta is `exit_delta`; the HTLC policies use larger
   deltas (`2 * exit_delta` for the htlc-send recovery clause,
   `htlc_expiry_delta + exit_delta` for the htlc-recv claim clause, ARK #2).
5. **Claim.** The user spends the output via the policy's unilateral clause
   into its on-chain wallet. For a `pubkey` policy this is the
   delayed-sign leaf: witness `[<sig>, <script>, <control block>]` with
   `nSequence = exit_delta` (height-based). Other policies use their
   respective clauses and timing rules (ARK #2); a clause carrying an
   absolute timelock (the HTLC recovery clauses use `OP_CLTV`) additionally
   requires the claim transaction's `nLockTime` to be set to at least the
   clause's height — the output is unspendable before that height regardless
   of the relative delay, so a claim built with only the relative lock is
   consensus-invalid until then. Claiming is optional — the confirmed output
   is already the user's — but the output is a bespoke taproot output that
   most wallet software cannot detect or re-derive from a seed, so claiming
   consolidates the funds under ordinary keys. The claim transaction
   satisfaction weight per input is the clause's witness size (signature +
   script + control block; for the pubkey policy clause, control block depth
   0).

## Shared chains

Exit chains of different VTXOs may share prefixes (a round tree, a
checkpoint transaction). Once any party broadcasts a shared transaction,
other parties' exits simply start deeper in the chain; their validation and
procedure are unchanged. A user tracking multiple exiting VTXOs SHOULD
deduplicate shared transactions and broadcast each once.

The server watches for *confirmed* exit transactions (it does not react to
mempool-only exits); for arkoor chains it responds by broadcasting the
relevant checkpoint transaction (ARK #5), which does not interfere with the
exiting user's deeper transactions. (The reference server waits a configurable
grace period — a number of blocks past the exit transaction's confirmation
height, default 2 — before responding, preferring that the exiting party pay
the fees.)

## After expiry

After `expiry_height`, every cosign/leaf-cosign output in an unbroadcast
portion of the chain becomes sweepable by the server through its
timelock-sign expiry leaf, and checkpoint outputs likewise. An exit begun
too late can therefore fail for the un-broadcast remainder. Conversely,
outputs of *confirmed* exit transactions whose next transition is already
confirmed are out of the server's reach; only the frontier matters. This is
why step 2 proceeds root-first without waiting between levels beyond each
parent's confirmation.

## Requirements summary

* Users MUST persist all data required to construct the exit chain (the full
  VTXO encoding suffices) independently of the server.
* Users MUST monitor chain height against `expiry_height` and begin exits
  with enough margin for confirmation of the whole chain plus `exit_delta`.
* Users MUST NOT reuse or spend a VTXO off-chain after starting its exit.
* Wallets SHOULD expose the total cost estimate (sum of chain weights plus
  CPFP and claim costs) before starting an exit, since deep chains can be
  expensive; refreshing early (ARK #4) keeps chains shallow.
