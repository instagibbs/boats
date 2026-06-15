# ARK #1: Protocol Object Encoding

This document specifies the binary encoding ("protocol encoding") used for
all Ark protocol objects exchanged between client and server and stored by
clients. Higher-numbered documents define object layouts in terms of the
primitives defined here.

Objects are encoded as a flat concatenation of fields with no framing, tags,
or padding. When transmitted as text, objects are hex-encoded in lowercase.

## Primitive types

| Type | Size | Encoding |
|---|---|---|
| `u8`, `u16`, `u32`, `u64` | 1/2/4/8 | unsigned little-endian |
| `bytes[N]` | N | raw bytes |
| `compact_size` | 1–9 | Bitcoin CompactSize ("varint"), see below |
| `pubkey` | 33 | secp256k1 public key, compressed SEC encoding |
| `xonly_pubkey` | 32 | BIP-340 x-only public key |
| `schnorr_sig` | 64 | BIP-340 signature |
| `sha256` | 32 | raw hash bytes |
| `taptweak` | 32 | BIP-341 TapTweak hash bytes |
| `musig_pub_nonce` | 66 | MuSig2 (BIP-327) public nonce, two compressed points |
| `musig_partial_sig` | 32 | MuSig2 partial signature scalar |
| `outpoint` | 36 | Bitcoin consensus encoding: txid (32, internal byte order) ‖ vout (`u32` LE) |
| `txout` | var | Bitcoin consensus encoding: value (`u64` LE) ‖ compact_size script length ‖ script |
| `transaction` | var | Bitcoin consensus encoding (with witness) |

### `compact_size`

Identical to Bitcoin's CompactSize:

* `0x00`–`0xFC`: the value itself in 1 byte
* `0xFD` ‖ `u16`: values `0xFD`–`0xFFFF`
* `0xFE` ‖ `u32`: values `0x1_0000`–`0xFFFF_FFFF`
* `0xFF` ‖ `u64`: larger values

A reader MUST reject non-minimal encodings (e.g. `0xFD 0x10 0x00` for the
value 16).

### Vectors

A vector of `T` (`vec<T>`) is encoded as:

```
compact_size: count
T * count:    elements
```

Before allocating, a decoder MUST check the element count against the global
limit: `count * size_of(T)` MUST NOT exceed `MAX_VEC_SIZE = 4_000_000` bytes,
where `size_of(T)` is the in-memory size of the element type. Decoders MUST
reject oversized vectors rather than attempting to allocate.

This bound is a local denial-of-service guard, not an interop-critical
threshold: `size_of(T)` is the decoder's in-memory element size and is
therefore implementation-defined (e.g. a 33-byte public key occupies more than
33 bytes in memory), so two conformant decoders MAY disagree on the exact
cutoff near the boundary. Implementations MUST NOT rely on a peer accepting a
vector whose count is close to the limit; the binding per-object maximum counts
are those fixed by the object layouts in later documents.

### Optional values

There is no generic option encoding; each optional field uses one of the
following schemes, specified per field in later documents:

* **`option<sha256>`** — a prefix byte: `0x00` for absent; `0x01` followed by
  the 32 hash bytes for present. Any other prefix byte is invalid.
* **`option<pubkey>`** — a single `0x00` byte for absent. If the first byte
  is non-zero, it is the first byte of a 33-byte compressed public key (whose
  first byte is always `0x02` or `0x03`, so no ambiguity arises). Decoders
  MUST reject keys that are not valid curve points.
* **`option<schnorr_sig>`** — always 64 bytes; the all-zero string denotes
  absent (the all-zero string is not a valid signature, so no ambiguity
  arises).

### `txout` restrictions

When decoding a `txout` from an untrusted source, the decoder MUST reject any
output whose `scriptPubkey` exceeds `MAX_SCRIPT_PUBKEY_SIZE = 100` bytes.
(All scriptPubkeys produced by this protocol are at most standard P2TR size;
the limit is a DoS bound.)

## Bounds on heights and deltas

Block deltas (relative timelocks, `u16`) and block heights (absolute heights,
`u32`) are subject to global bounds. Decoders MUST enforce them when decoding
VTXOs and policies (ARK #2). Composite objects that embed heights and deltas
do not all re-check them at decode time — the tree spec (ARK #4) decodes its
`expiry_height` and `exit_delta` unchecked, and policy *construction*
performs no bounds check — so a receiver of such an object MUST validate the
embedded values against these bounds before acting on it (see ARK #4's
participant requirements). The bounds:

* `MAX_BLOCK_DELTA = 16383` (`u16::MAX / 4`). A delta greater than this MUST
  be rejected.
* `MAX_BLOCK_HEIGHT = 500_000_000 - 1 - 4 * 16383 = 499_934_467`
  (`LOCK_TIME_THRESHOLD - 1 - 4 * MAX_BLOCK_DELTA`). A height greater than
  this MUST be rejected.

These bounds guarantee that any accepted height plus up to four accepted
deltas remains a valid absolute locktime height (below Bitcoin's
`LOCK_TIME_THRESHOLD` of 500,000,000), and that any sum of up to four deltas
fits in a `u16`.

## Error handling

Decoders MUST NOT panic or abort on malformed input. All decoding failures
(truncated input, invalid keys or signatures, unknown type/version bytes,
non-minimal compact sizes, oversized vectors, out-of-bounds heights/deltas)
MUST be surfaced as recoverable errors.

**Trailing bytes.** Because objects are unframed, a decoder reading an object
from a byte buffer stops once all of the object's fields are consumed. The
reference implementation does *not* reject trailing bytes after a
fully-decoded protocol object (it decodes from a slice and ignores the
remainder), whereas the embedded consensus-encoded `transaction`/`txout`
primitives *do* reject trailing data within their own bytes. Transports
SHOULD deliver each object in an exactly-sized container; a decoder MAY ignore
trailing bytes (reference behavior) but MUST NOT treat their presence as
additional fields.

## Versioning

The top-level composite objects that may evolve carry an explicit version
field as their first encoded field. Nested objects do not each carry a
version: VTXO policies and genesis transitions are identified by a leading
*type* byte (ARK #2), and objects such as the leaf spec (ARK #4) and the
ownership attestations carry neither a version nor a type byte and evolve
under their container's version. Current versions of the versioned objects:

| Object | Version field | Current value | Defined in |
|---|---|---|---|
| VTXO | `u16` | 2 (1 accepted for decoding) | ARK #2 |
| VTXO tree spec | `u8` | 2 | ARK #4 |
| Signed VTXO tree spec | `u8` | 2 (1 accepted for decoding) | ARK #4 |
| Hash-locked forfeit bundle | `u8` | 1 | ARK #4 |

Encoders MUST always emit the current version. Decoders MUST reject unknown
versions.
