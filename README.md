# NUT-CCP: Cashu Card Protocol

This repository holds the draft specification for Cashu NFC cards: smart cards that hold Cashu proofs and pay a reader with one tap, including when the reader is offline. The card holder can pay without a phone at the point of sale. The specification is written as a Cashu NUT for proposal to [cashubtc/nuts](https://github.com/cashubtc/nuts). Until the Cashu maintainers assign it a number, it is called NUT-CCP.

| File                                                 | Contents                                                                                 |
| ---------------------------------------------------- | ---------------------------------------------------------------------------------------- |
| [`CCP.md`](CCP.md)                                   | The core: selecting a card, card proofs and their keys, reading, paying, change, loading |
| [`tests/CCP-tests.md`](tests/CCP-tests.md)           | Test vectors                                                                             |
| [`profiles/certificate.md`](profiles/certificate.md) | The certificate profile: an issuer's statement for the card key                          |

## Status

Draft and experimental. This proposal has had little outside review, and the only implementation tested on hardware is the reference one, on NXP JCOP4 cards, where it has run revision `02` of the core as written and, since 2026-11-13, revision `03` (protocol 2.3): on a J3R180 at a test mint, with real signatures, the card's own change finished with `LOAD_CHANGE`, and that change spent again, and with a refund key: dated proofs paid under their date's key and their dated change finished. Not yet: a phone, a production mint, a card torn on purpose, or a dated proof taken back after its date by the refund key; and change counted when the payment is signed, a clock set back by an owner's anchor on an empty card, a weak header that begins and ends no day, and the refresh that brings the clock back (**Clock and limits**), and the owner's `c` made from its wallet's seed and given at a claim or a replacement (**Owner**), have run only in a simulator so far. Questions go to [Discussions](https://github.com/cashucard/cashucard/discussions) in cashucard/cashucard. The certificate profile is specified here. The other profiles (the owner key, the clock and limits, the NFC tag, and those of protocol 2.3) are specified in the reference implementation's [command reference](https://github.com/cashucard/cashucard/blob/main/spec/APDU.md) until they are written here.

## Implementations

- [cashucard/cashucard](https://github.com/cashucard/cashucard): the Java Card applet, with a simulator, tools and an operator provisioner.
- [Flash](https://github.com/lnflash) wallet and point-of-sale apps read, load and pay from cards of revision `02`; revision `03` is still to come in them.

The protocol grew out of the [Foxy](https://github.com/getfoxy/card) card's design, merged with Flash's work.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).
