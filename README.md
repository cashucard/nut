# NUT-CCP: Cashu Card Protocol

This repository holds the draft specification for Cashu NFC cards: smart cards that hold Cashu proofs and pay a reader with one tap, with the reader offline if need be. It is written as a Cashu NUT so it can be proposed to [cashubtc/nuts](https://github.com/cashubtc/nuts). Until the Cashu maintainers assign it a number, it is called NUT-CCP.

| File                                       | Contents                                                                  |
| ------------------------------------------ | ------------------------------------------------------------------------- |
| [`CCP.md`](CCP.md)                         | The core: selecting a card, card proofs, reading, paying, change, loading |
| [`tests/CCP-tests.md`](tests/CCP-tests.md) | Test vectors                                                              |

## Status

Draft. The core describes protocol 2.1 as the reference implementation runs it on NXP JCOP4 cards. Profiles for the owner key, the clock and limits, and the NFC tag are specified in the reference implementation's [command reference](https://github.com/cashucard/cashucard/blob/main/spec/APDU.md) until they are written here.

## Implementations

- [cashucard/cashucard](https://github.com/cashucard/cashucard): the Java Card applet, protocol 2.1, with a simulator, tools and an operator provisioner.
- [Flash](https://github.com/lnflash) wallet and point-of-sale apps read, load and pay from 2.1 cards.

The protocol grew out of the [Foxy](https://github.com/getfoxy/card) card's design, merged with Flash's work in protocol 2.0.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).
