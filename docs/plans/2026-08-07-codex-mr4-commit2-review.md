# Codex review record — MR-4 commit 2: "channel node discovery + retry codes"

Commit under review: bark `fe8b1fa6` (jj change `xzwryxvt`) on
`ark8-channels-stage1-client`, child of commit 1 (`c53eaedf`).
Reviewer: codex, `model_reasoning_effort=max`, read-only sandbox, two
rounds. Design contract: `2026-08-06-mr4-client-design.md` §2.

Verdict history: R1 **FAIL** (3 findings, all adopted) → R2 **PASS**.

## Round 1 — three findings, all adopted

1. **65-byte uncompressed node ids passed conversion.**
   `PublicKey::from_slice` accepts both encodings; the contract is the
   33-byte compressed form BOLT node ids use. Fix: length check before
   parse; regression case (a valid uncompressed key is refused) added
   to the round-trip test.
2. **Bare IPv6 mis-validated as dialable.** The
   `advertise_addresses` port check split on the last `:`, so `::1`
   passed with its final hextet as the "port" — an undialable
   advertisement. Fix: a non-`SocketAddr` entry must be `host:port`
   with no `:` in the host (bracketed IPv6 parses as `SocketAddr`;
   bare IPv6 is refused with a bracket-it hint), empty hosts and junk
   ports refused, empty list still valid (means "cannot open
   channels"). Validation test module added.
3. **The client-visible retry text changed.** `RetryableConflict`'s
   `Display` prepended `retryable conflict:` to the two scan-epoch
   abort messages — clients matching on the existing text would break,
   and the tag's whole point is to move retry signaling INTO the gRPC
   code. Fix: transparent `Display` (inner context only); the two
   messages stay byte-identical to before the tag, with the rule
   stated in a comment.

## Round 2 — PASS

All three fixes verified: the length check is correctly typed as
`ConvertError`; the address validation accepts hostnames/IPv4/
bracketed-IPv6 (including numeric scope ids) and rejects bare IPv6,
empty hosts, and invalid ports; `RetryableConflict` is a root error
with transparent `Display`, so the anyhow `{:#}` chain renders exactly
the pre-tag text at both sites, and both map to `Code::Aborted`.

Round 1 also affirmatively checked: proto field numbering (22–24,
last-index comment), optional/repeated wire compat in both directions,
the bark-json mirror + field-completeness test, openapi regeneration,
population ordering (the channels subsystem is constructed before any
RPC server serves, so `channel_node_id` is `Some` whenever
`supports_channels`), the listen address never being advertised
implicitly, retry-code precedence (`NotFound`/`UnusableInputs`/
`BadArgument` before `Aborted`), and the cosign parse mapping to
`InvalidArgument` (previously: `StatusContext::context` routed the
decode failure through anyhow to `Internal` — the design note's claim
confirmed against the code).
