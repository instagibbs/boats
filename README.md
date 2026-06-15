# ARK Specifications

> **BOATS** — *Basis Of Ark Technical Specifications*

A BOLT-style normative specification of the Ark protocol as implemented by
Second's `ark-bitcoin` codebase (server: `captaind`, client: `bark`). Each
document is precise enough that an independent party can build a compatible
client or server without reading the reference implementation.

The specifications are transport-agnostic: messages are defined by their
semantic content and binary object encodings, independent of the wire
transport the reference implementation happens to use.

Start with [ARK #0](00-overview.md) for conventions and the protocol overview.

## Documents

* [ARK #0: Overview and Conventions](00-overview.md)
* [ARK #1: Protocol Object Encoding](01-encoding.md)
* [ARK #2: VTXOs, Policies, and the Genesis Chain](02-vtxo.md)
* [ARK #3: Boarding](03-boarding.md)
* [ARK #4: Rounds](04-rounds.md)
* [ARK #5: Out-of-round (arkoor) Payments](05-arkoor.md)
* [ARK #6: Emergency Exit](06-exit.md)
* [ARK #7: Offboarding](07-offboard.md)
* [ARK #8: Channels](08-channels.md)

## License

[MIT](LICENSE).
