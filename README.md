# NUT-CCP: Cashu Card Protocol

This repository holds the draft specification for Cashu NFC cards: smart cards that hold Cashu proofs and pay a reader with one tap, including when the reader is offline. The card holder can pay without a phone at the point of sale. The specification is written as a Cashu NUT for proposal to [cashubtc/nuts](https://github.com/cashubtc/nuts). Until the Cashu maintainers assign it a number, it is called NUT-CCP.

| File                                       | Contents                                                                  |
| ------------------------------------------ | ------------------------------------------------------------------------- |
| [`CCP.md`](CCP.md)                         | The core: selecting a card, card proofs, reading, paying, change, loading |
| [`tests/CCP-tests.md`](tests/CCP-tests.md) | Test vectors                                                              |

## Design proposals

The [assisted-settlement proposal](proposals/assisted-settlement.md) covers authenticated pending-payment export, merchant recovery and holder-authorized slot reclamation. Its wire format and limits are under review before implementation.

## Status

Draft and experimental. This proposal has had little outside review, and the only implementation tested on hardware is the reference one, on NXP JCOP4 cards, where it runs the core as written. Questions go to [Discussions](https://github.com/cashucard/cashucard/discussions) in cashucard/cashucard. Profiles for the owner key, the clock and limits, and the NFC tag are specified in the reference implementation's [command reference](https://github.com/cashucard/cashucard/blob/main/spec/APDU.md) until they are written here.

## Implementations

- [cashucard/cashucard](https://github.com/cashucard/cashucard): the Java Card applet, with a simulator, tools and an operator provisioner.
- [Flash](https://github.com/lnflash) wallet and point-of-sale apps read, load and pay from cards.

The protocol grew out of the [Foxy](https://github.com/getfoxy/card) card's design, merged with Flash's work.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).
