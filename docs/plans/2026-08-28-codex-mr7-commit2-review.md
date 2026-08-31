# Codex review record — MR-7 c2: the `bark channel` CLI noun

**Commit:** `cf423cb8a` "bark-cli: the `bark channel` noun" (amended
through the rounds), on c1 `cdfe88223`, branch
`ark8-channels-stage1-payments`.
**Verdict: PASS in 5 rounds.** The executable CLI smoke green on every
amend; full battery green.

## What shipped

- **`bark channel`**: open, open-status, list, close, exit,
  cancel-exit, pay, invoice, payments — thin over the wallet API,
  printing the SAME bark-json types REST serves (bark-rest's exported
  converters: one wallet→DTO mapping, two surfaces). No keysend.
- **One-shot lifecycle**: driver-needing verbs start the wallet daemon
  for the command's duration; payment-shaped verbs first wait for a
  USABLE channel (the new `usable` = ready AND peer-connected DTO
  field — right after daemon start, transport is a race, not a
  refusal); reads stay daemon-free. `open`/`close`/`pay` poll to a
  terminal outcome and exit non-zero on failure; `--no-wait` returns
  the handle.
- **Resume paths for `--no-wait`**: `open-status --wait` (validates
  the id daemon-free FIRST, then starts the daemon only for a
  confirmed in-progress open); re-running `close` on a mid-close
  record skips the initiate and waits. Both documented in the help.
- **barkd coexistence**: a barkd-hosted wallet refuses the CLI at the
  datadir PID lock — deliberate and load-bearing (two channel nodes
  over one monitor store would split-brain LDK state); with barkd
  running, REST is the interface. Stated in the module doc + commit.
- **e2e**: an EXECUTED smoke — refused dust open, synchronous open
  through the action resource, list/open-status consistency, an
  amountless own invoice, a real payment to captaind's collect leg
  (`sent`, hash-verified), the journal listing, the close waited to
  settlement.

## The rounds

- **r1 (3):** `cancel-exit` raced the daemon's exit loop (it could
  broadcast the bridge before the cancel takes the lock) → the verb is
  DAEMON-FREE; `close` counted `fallback_only`/`exiting` as success →
  only the cooperative settlements are, the fallback shapes print and
  exit non-zero; `--no-wait` orphaned pending actions with no resume
  path → the resume verbs above (maintenance deliberately does not
  drive channels; the daemon does).
- **r2 (1):** `open-status --wait` started the daemon before
  validating the id (a bogus id triggered startup sync and driving) →
  daemon-free validation first.
- **r3 (2):** exiting the process before the manager-persistence drain
  could re-read an accepted `--no-wait` pay as STRANDED → every
  channel command drains through `stop_daemon_wait()` (the driver's
  graceful shutdown performs the final persist), early returns
  included; a fallback that terminates in `closed` read as cooperative
  success → a `closed` WITH an exit entry reports fell-back.
- **r4 (1):** drain failures were logged-and-discarded → a failed
  drain FAILS the command (durability unknown must not read as
  success).
- **r5: PASS.**

## Key decisions for review

- **The CLI is for wallets barkd doesn't host** (the PID lock enforces
  it); the split is documented rather than worked around.
- **Cancel-exit runs daemon-free** — the safety rule (never progress
  an exit while canceling it) beats the uniform-lifecycle aesthetic.
- **A failed drain is a failed command** even when the verb itself
  succeeded.
