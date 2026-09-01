# The channel suite's flaky set — evidence and how to chase it

Three tests failed under parallel load during the security-remediation
battery runs and PASSED in isolation immediately afterwards. They are not
regressions from that work; they are load-sensitive. This is the evidence
so the next person does not have to re-derive it, plus what to try.

Nothing here is fixed. Nothing here is known-benign either — "passes in
isolation" rules out a logic regression, not a real race that only opens
under contention. A race in the harness is still a race.

## The set

**`bark-sdk lifecycle::client_punishes_revoked_server_close`**
FAILED at **6.5s** in a full `-j 4` bark-sdk run (83 tests) at the stack
tip. PASSED in isolation at **216.9s** immediately after, and its sibling
`…_with_htlc` at 285.8s. The vector normally takes ~200s, so a 6.5s
failure never reached the justice choreography at all — it died during
setup.

> **SOLVED 2026-08-31 — not environmental, a missing gate in the test.**
> Reproduced deterministically while writing the server-side justice
> vectors, which copy this one's shape (pay, export the commitment, pay
> again). The failure is `SendingFailed(RouteNotFound)` on the SECOND
> payment, and the router says why:
>
> ```
> First hop ... can send between 1msat and 10000000msat
> ...not including outbound HTLC 0 (value 30000000) due to state
>    (AwaitingRemovedRemoteRevoke)
> Ignoring first hop ... insufficient value contribution
>    (channel max ExactLiquidity { liquidity_msat: 10000000 })
> ```
>
> A payment reading `Sent` only means the preimage is back. LDK keeps the
> HTLC in `AwaitingRemovedRemoteRevoke` until the commitment dance
> finishes, and while it lingers it still consumes the channel's 40k
> in-flight allowance (`MAX_HTLC_IN_FLIGHT_PERCENT` of the 400k capacity)
> — leaving 10k, so the follow-up 20k payment finds no route. The BALANCE
> is there; the ALLOWANCE is not. Load widens the window, which is why it
> only showed under `-j 4`.
>
> Fix: gate on the HTLC being gone, not on `Sent` — `await_htlcs_settled`
> in `lifecycle.rs` polls `test_pending_htlc_expiries()` to empty.
> Applied to this vector and to both new `server_punishes_revoked_*`
> ones. NOT a retry: the race is real and a retry would hide it.
>
> Note the diagnostic below still holds and is what found this — the
> fast failure was indeed setup, just not the environment's fault.

**`bark-sdk vtxo_lock::concurrent_locks_serialize`**
FAILED at **5.2s** in a different full `-j 4` run at the tip. PASSED in
isolation, with the whole `vtxo_lock` group green (6/6, 5.9s total).
Untouched by the channels work.

**`server channels::downgrade_split_admission_and_group_registration`**
FAILED at ~**7.9s** at two adjacent commits of a 19-commit cascade
(`-j 4`), while PASSING at the commits immediately before and after
(01, 05, 08). Passed in isolation at **6.7s**. A pass/fail/pass pattern
across neighbouring commits cannot be a code regression — the commits
between them do not touch it.

## The diagnostic that separates flakes from real failures

**Fast failure = setup. Slow failure = the thing under test.**

Every genuine failure found during the remediation took hundreds of
seconds and died on its own assertion (929s "the funding is still
unspent", 636s "no close reached the chain", 545s "recovered only N sat").
Every flake died in 5–8 seconds, before its scenario could run.

Use that before assuming anything: a channel vector that fails in under
ten seconds did not fail at what it tests.

Do NOT chase these three, which failed for real during the remediation
and are FIXED: `force_close_server_claims_and_payer_scrapes`,
`peer_force_close_before_release_registers_then_exits`,
`theirs_commitment_sweeps`. They broke on a mis-implemented review
recommendation (an operator force-close was being refused when the
funding was not yet on-chain, which is a sanctioned flow) and on an
over-eager commitment relay that starved round funding. Both are
corrected; all three pass.

## Reproducing

```
# the environment (see the barkd-e2e-test-setup notes)
export BITCOIND_EXEC=/path/to/bitcoin-29.0/bin/bitcoind
export POSTGRES_BINS=/usr/lib/postgresql/16/bin
export CAPTAIND_EXEC=$PWD/target/debug/captaind      # built WITH channel-test-seams
export BARK_EXEC=$PWD/target/debug/bark
export BARKD_EXEC=$PWD/target/debug/barkd
export WATCHMAND_EXEC=$PWD/target/debug/watchmand

# reproduce (the whole suite, parallel — this is where they appear)
cargo nextest run -p ark-testing --test bark-sdk -j 4 --no-fail-fast

# confirm it is load-sensitive (one at a time)
cargo nextest run -p ark-testing --test bark-sdk -j 1 --no-fail-fast <name>
```

They are intermittent: a given run may show none, or one, and not the
same one. Budget several full runs before concluding anything, and keep
`--success-output final` off for the reproduction (it buries the failure).

## Where to look first

Each test spawns its own bitcoind, postgres, captaind and (usually) two
bark wallets; `-j 4` means four such stacks racing to start. The failures
all land in that window.

1. **Get the real panic.** The battery scripts grepped only summary
   lines, which is why the exact assertion is not recorded here. Capture
   full output for the failing test and read the panic and the daemon
   logs under `test/<suite>/<test-name>/`. Do this before theorising.
2. **Port allocation.** `ark_testing::ports::pick_port()` picks a free
   port and something else may take it before the daemon binds. A
   pick-then-bind gap is the classic version of this bug.
3. **Postgres and bitcoind startup timeouts.** Under four-way contention
   a daemon may exceed a readiness wait that is generous when idle.
4. **Shared bitcoind snapshot.** The harness restores from a snapshot
   directory (`test/_bitcoind_snapshot`); check whether concurrent
   restores can interleave.
5. **`concurrent_locks_serialize` specifically** tests lock
   serialization under concurrency — the one candidate here where the
   failure might be the product, not the harness. Give it its own look.

## Options if the cause is genuinely environmental

`.config/nextest.toml` already sets `retries = 3` on the
`ci-merge-train` profile only. Enabling retries on `default` would hide
these rather than fix them — acceptable for CI noise, wrong while the
cause is unknown. Prefer lowering parallelism for the heavy channel
vectors (they already have long `slow-timeout` overrides; a
`test-threads` override for that filter is the natural companion) over
blanket retries.

## Not flaky: the CLN set

Five failures are environmental on this machine and unrelated:
`lightning::pay_hold_refused`, `lightning::pay_hold_succeeds`,
`lightning::pay_hold_with_near_expiry_inputs_succeeds`,
`lightning::receive_claim_on_mailbox_notification`, and
`payment_request::build_valid_bip321_uri`. They fail at ~1.5s because
`lightningd` cannot start here (`gmp version mismatch: compiled 6.2.1,
now 6.3.0`). They need a box with a working CLN; there is nothing to fix
in the tree. A genuinely clean whole-suite sweep is not possible on this
machine until that is sorted.
