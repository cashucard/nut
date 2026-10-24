# NUT-CCP: Cashu Card Protocol

`optional`

`depends on: NUT-00, NUT-02, NUT-03, NUT-10, NUT-11`

`uses: NUT-09, NUT-12`

---

This NUT specifies how a smart card holds Cashu proofs and spends them over NFC. The card makes a secp256k1 key pair at install and keeps the private key. Every proof on the card is locked to that key with [NUT-11][11] P2PK and the `SIG_ALL` flag. To pay, a reader (a point-of-sale terminal or a wallet) builds a [NUT-03][03] swap from the card's proofs, and the card signs the whole swap with one BIP-340 signature, marking each proof spent as it signs. The reader can prepare the swap while offline from the mint, and the card holder can pay without a phone at the point of sale.

Mints accept card payments as ordinary swaps of `SIG_ALL` proofs.

This document specifies the core every card and reader share: selecting the card, the format of its proofs, reading them, paying, the card's own change, and loading proofs. How a card is administered (an owner key, a PIN's management, spending limits and a clock) is left to profiles, listed under [Profiles](#profiles).

## Terms

- **Card**: an ISO/IEC 14443-4 contactless device running an application that implements this NUT.
- **Reader**: a device that speaks ISO/IEC 7816-4 commands to the card. A point-of-sale terminal paid by the card is a reader, and so is a wallet that loads it.
- **Card key** (`P_card`): the card's secp256k1 key pair. The card makes it on itself and **MUST NOT** export the private key.
- **Card proof**: a `Proof` ([NUT-00][00]) whose secret is the P2PK secret defined in [Card proofs](#card-proofs).
- **Slot**: the card's storage for one card proof.
- **Tap**: one session between a card and a reader, from selection until the card leaves the field.

## Transport

A reader sends ISO/IEC 7816-4 short APDUs. Every command in this NUT has class byte `B0`, except `SELECT`. Multi-byte numbers are big-endian unless a field says otherwise. A response ends with a two-byte status word; `90 00` is success. [Status words](#status-words) lists the others.

## Selecting the card

A reader selects the card's application by its full AID:

| Field | Value                              |
| ----- | ---------------------------------- |
| CLA   | `00`                               |
| INS   | `A4`                               |
| P1    | `04`                               |
| P2    | `00`                               |
| Data  | `F0 43 41 53 48 55 43 41 52 44 01` |

The AID is the proprietary RID byte `F0`, the ASCII string `CASHUCARD`, and `01`. The card answers two version bytes. A major byte of `02` identifies a card of this NUT. The minor byte counts additions to the commands and fields this NUT defines, and a reader **MUST** accept any minor byte it does not recognise.

A reader **MUST** select the full AID. A selection by the first ten bytes alone can reach a different application on the same chip. A response to `SELECT` that is not exactly two bytes beginning `02` **MUST** be treated as no card of this NUT, and so **MUST** a `6A 82` status word.

## Card proofs

Every card proof has a [NUT-10][10] P2PK secret whose `data` is `P_card` and whose tags end with `["sigflag","SIG_ALL"]`. A proof without a date has this secret:

```
["P2PK",{"nonce":"<nonce>","data":"<P_card>","tags":[["sigflag","SIG_ALL"]]}]
```

A proof with a date also carries a locktime and the card's refund key ([NUT-11][11]):

```
["P2PK",{"nonce":"<nonce>","data":"<P_card>","tags":[["locktime","<date>"],["refund","<refund key>"],["sigflag","SIG_ALL"]]}]
```

The secret **MUST** be serialized exactly as shown: no whitespace, keys and tags in this order, `<nonce>` as 64 lowercase hex characters, `<P_card>` and `<refund key>` as 66 lowercase hex characters of a compressed point, and `<date>` in decimal. The card stores the nonce and date and rebuilds each secret with this serialization. Payment signatures follow the `SIG_ALL` transcript defined below.

[NUT-11][11] requires every input of a `SIG_ALL` swap to have the same `data` and tags. Card proofs share `data`, so the proofs of one payment **MUST** share the date and the refund key. The card refuses a payment that mixes them.

A wallet puts proofs on a card by an ordinary [NUT-03][03] swap whose outputs are card proofs, then writes each proof to the card with [`LOAD_PROOF`](#load_proof).

### The stored form

A slot holds 82 bytes:

| Offset | Size | Field                                                       |
| ------ | ---- | ----------------------------------------------------------- |
| 0      | 1    | status: `00` empty, `01` unspent, `02` spent                |
| 1      | 8    | keyset: the first 8 bytes of the [NUT-02][02] keyset ID     |
| 9      | 4    | amount, unsigned                                            |
| 13     | 32   | nonce                                                       |
| 45     | 33   | `C`, the mint's unblinded signature, a compressed point     |
| 78     | 4    | date: the locktime in seconds, or `0` for a proof with none |

A card has 128 slots, numbered 0 to 127. A keyset ID longer than 8 bytes is stored by its first 8. A reader **MUST** expand the stored bytes against the mint's keysets ([NUT-02][02]) and **MUST** refuse a proof whose 8 bytes match more than one keyset.

## Reading the card

These commands need no authorisation.

| INS  | Command           | Response                                                  |
| ---- | ----------------- | --------------------------------------------------------- |
| `01` | `GET_INFO`        | the card's state ([GET_INFO](#get_info))                  |
| `10` | `GET_PUBKEY`      | `P_card`, 33 bytes                                        |
| `11` | `GET_BALANCE`     | the sum of the unspent proofs' amounts, 4 bytes           |
| `12` | `GET_PROOF_COUNT` | the number of slots that are not empty, 1 byte            |
| `13` | `GET_PROOF`       | the slot P1 (0 to 127), 82 bytes                          |
| `14` | `GET_SLOT_STATUS` | one status byte per slot, 128 bytes                       |
| `15` | `AUTH`            | proof that the card holds `P_card` ([AUTH](#auth))        |
| `16` | `GET_CARD`        | the card's mint and unit ([GET_CARD](#get_card))          |
| `17` | `GET_PIECES`      | several slots in one response ([GET_PIECES](#get_pieces)) |
| `19` | `GET_CHANGE`      | the openings of the card's own change ([Change](#change)) |
| `1A` | `GET_DLEQ`        | the mint's DLEQ proofs kept with proofs ([DLEQ](#dleq))   |

`GET_PROOF` and `GET_DLEQ` answer `6A 83` for a slot out of range.

### GET_INFO

P1 `00` returns 30 bytes, P1 `01` returns 42 and P1 `02` returns 43. The fields this NUT defines:

| Byte | Field                                                         |
| ---- | ------------------------------------------------------------- |
| 0    | version major, `02`                                           |
| 1    | version minor                                                 |
| 2    | number of slots                                               |
| 3    | unspent slots                                                 |
| 4    | spent slots not yet freed                                     |
| 5    | empty slots                                                   |
| 6    | capabilities                                                  |
| 7    | PIN state: `0` none, `1` set, `2` blocked                     |
| 8    | proof format, `04` for the card proofs of this NUT            |
| 9    | PIN tries left                                                |
| 10   | `1` when the card is locked and takes no more proofs          |
| 11   | `1` when the card has a mint and unit ([GET_CARD](#get_card)) |

Bytes 12 and beyond belong to [profiles](#profiles). A reader that implements no profile **MUST** ignore them.

### GET_CARD

The response is 109 bytes, then the mint URL, then 3 bytes:

| Offset  | Size | Field                                        |
| ------- | ---- | -------------------------------------------- |
| 0       | 1    | proof format, `04`                           |
| 1       | 1    | `1` when the fields below are set            |
| 2       | 1    | unit: `00` is `sat`                          |
| 7       | 33   | the refund key for dated proofs, or zeros    |
| 105     | 1    | the mint URL's length, at most 77            |
| 106     | n    | the mint URL, UTF-8                          |
| 106 + n | 3    | a design code chosen by the issuer, or zeros |

Bytes 3 to 6 and 40 to 104 belong to [profiles](#profiles). A reader **MUST** take every proof on the card as issued by the mint at this URL, in this unit.

### GET_PIECES

P1 is the first slot to report. P2 `00` returns one page of at most 255 bytes: the first slot the page does not cover (`128` at the end), then, for each slot from P1 that is not empty, a tag byte (the slot number, with bit 7 set when the slot is spent) followed, for an unspent slot, by its 81 bytes after the status byte. P2 `02` takes one to three slot numbers as data and returns each slot's 82 bytes in the order asked. Another P2 answers `6A 86`.

### AUTH

The reader sends 16 random bytes. The card answers with 16 random bytes of its own and a BIP-340 signature by `P_card` over

```
SHA-256( T || T || reader_nonce || card_nonce || P_card ),  T = SHA-256("CashuCard/auth")
```

with `P_card` compressed. A reader that verifies it knows the card present holds the key its proofs are locked to. The distinct tagged message authenticates possession of the private key corresponding to `P_card` without spending a proof.

## Paying

A payment is one tap with these commands, in this order:

| INS  | Command             | Data                                 | Response                    |
| ---- | ------------------- | ------------------------------------ | --------------------------- |
| `22` | `SPEND_ALL_BEGIN`   | the slots to spend, one byte each    | their total amount, 4 bytes |
| `23` | `SPEND_ALL_OUTPUTS` | per output: amount (4) and `B_` (33) | none                        |
| `26` | `SPEND_ALL_CHANGE`  | the change amount (4)                | the change's `B_`, 33 bytes |
| `24` | `SPEND_ALL_SIGN`    | none                                 | the signature, 64 bytes     |

`SPEND_ALL_OUTPUTS` **MAY** be sent more than once; its data **MUST** be a multiple of 37 bytes. `SPEND_ALL_CHANGE` **MAY** be sent more than once or not at all. Any other command between `SPEND_ALL_BEGIN` and `SPEND_ALL_SIGN` abandons the payment.

The card signs, with BIP-340 and `P_card`, the SHA-256 of the [NUT-11][11] `SIG_ALL` swap message:

```
msg = secret_0 || C_0 || ... || secret_n || C_n || amount_0 || B_0 || ... || amount_m || B_m
```

The inputs are the slots in the order named. The outputs are the reader's outputs in the order given, followed by the card's change outputs in the order made. Each secret is the card proof's secret text, each `C` and `B_` is lowercase hex, and each amount is decimal.

The reader then sends the swap to the mint ([NUT-03][03]) with the inputs and outputs in that order and the signature as the witness of the first input:

```json
{ "signatures": ["<signature>"] }
```

A card implementing this NUT:

- **MUST** mark every slot named as spent in the same atomic write that releases the signature. A card removed before that write has spent nothing.
- **MUST NOT** sign anything other than a payment's message and the `AUTH` message.
- **MUST** answer `SPEND_ALL_SIGN` with `00 01` and status `90 00` when it needs more time. The reader **MUST** send `SPEND_ALL_SIGN` again until it receives the signature or an error.
- **MUST** answer `SPEND_ALL_AGAIN` (INS `25`) with the last signature it released, for a reader that lost the response. This **MAY** need the same authorisation as the payment did.

`SPEND_ALL_BEGIN` answers `6A 83`, `6A 88` or `69 85` for a slot out of range, empty or spent; `6A 80` for a slot named twice or with a date or refund key other than the first slot's; and `6A 96` for more slots than one signature can cover.

### Change

The card can make change for itself. For `SPEND_ALL_CHANGE` the card builds a card proof secret with a fresh nonce and the payment's date and refund key, computes `Y = hash_to_curve(secret)` ([NUT-00][00]), blinds it with a fresh `r`, and returns `B_ = Y + rG`. It keeps the opening until the change is back on the card. The change amount **MUST** be greater than 0 and at most the inputs' total (`6A 80`); the card answers `6A 84` when it has no room to keep another opening.

The mint's swap response contains a blind signature `C_` for each change output. Whoever finishes the change unblinds it, `C = C_ - rK` with `K` the mint's key for that amount, and loads the proof with `LOAD_PROOF`. The card's change proof is locked to `P_card`, so the card can spend it regardless of who completes issuance.

`GET_CHANGE` (INS `19`, P1 a page from 0) returns a count byte and up to three openings of 80 bytes: amount (4), keyset (8), date (4), nonce (32) and `r` (32). A wallet that missed the mint's response **MAY** recover `C_` with [NUT-09][09] restore, using the change outputs rebuilt from these openings.

<a id="load_proof"></a>

## Loading proofs

`LOAD_PROOF` (INS `30`) writes card proofs to empty slots. P1 `00` takes one to three proofs, 81 bytes each:

| Size | Field           |
| ---- | --------------- |
| 8    | keyset          |
| 4    | amount          |
| 32   | nonce           |
| 33   | `C`, compressed |
| 4    | date, or `0`    |

and answers the slot of each. P1 `01` takes one proof followed by its [NUT-12][12] DLEQ proof `e || s || r` (96 bytes) and answers the slot and `01` when the card kept the DLEQ proof, or `00` when it had no room.

The card **MUST** refuse a nonce already in any slot, spent or not (`6A 94`), a `C` that is not a point (`6A 80`), a dated proof when it has no refund key (`6A 8E`), and any proof before its mint and unit are set (`6A 8C`). It answers `6A 84` when no slot is free. It **MUST** write the status byte last, so a write cut short leaves the slot empty.

Who may load is card policy ([Card policy](#card-policy)). A card **SHOULD** let the reader of a payment load that payment's change in the same tap or the next one.

`CLEAR_SPENT` (INS `31`) frees every spent slot and answers how many it freed. It is subject to the same policy as `LOAD_PROOF`.

## DLEQ

`GET_DLEQ` (INS `1A`) takes a slot in P1 and a second slot or `FF` in P2. For each slot asked it returns `01` and the 96 bytes `e || s || r` kept with an unspent proof, or `00`. DLEQ validation is the reader's responsibility. The reader checks a proof by rebuilding the secret, computing `Y = hash_to_curve(secret)`, `C' = C + rK` and `B' = Y + rG`, and verifying [NUT-12][12] with the mint's key `K` for the amount. A valid DLEQ proof shows the mint signed the proof. It does not show the proof is unspent.

## Card policy

A card **MAY** require authorisation for `LOAD_PROOF`, `CLEAR_SPENT`, `SPEND_ALL_SIGN` and `SPEND_ALL_AGAIN`, and **MAY** refuse or delay a payment under limits it keeps. Profiles define those rules. A reader **MUST** handle these responses:

| Status  | Meaning                                                                         |
| ------- | ------------------------------------------------------------------------------- |
| `69 82` | authorisation needed and not given in this tap                                  |
| `6A 94` | (`SPEND_ALL_SIGN`) the PIN is needed for this payment; the payment is abandoned |
| `6A 8F` | over a spending limit; nothing was signed or spent                              |
| `69 83` | the PIN is blocked                                                              |
| `69 86` | the card is locked                                                              |

A reader with a PIN from the holder presents it with `VERIFY_PIN` (INS `40`, P1 `00`, the PIN's 4 to 8 bytes as data) before `SPEND_ALL_SIGN`. A wrong PIN answers `63 CX`, with `X` the tries left; `69 84` means the card has no PIN. A verified PIN lasts until the end of the tap.

## Accepting a payment offline

A reader that accepts a payment without reaching the mint:

- **MUST** check that each input's keyset is a keyset of the mint named by `GET_CARD`, in the card's unit;
- **MUST** check each input's DLEQ proof ([DLEQ](#dleq)) and **SHOULD** decline a proof without one;
- **SHOULD** check `AUTH`;
- **MUST** swap at the mint as soon as it can, and **MUST NOT** report the payment settled until the mint accepts the swap.

The reader carries the risk that a proof was spent elsewhere before the card was presented. A copy of a proof can be spent elsewhere even though the card refuses to sign for a slot twice.

## Status words

| SW      | Meaning                                                          |
| ------- | ---------------------------------------------------------------- |
| `90 00` | success                                                          |
| `63 CX` | wrong PIN, `X` tries left                                        |
| `67 00` | wrong length                                                     |
| `69 82` | authorisation needed                                             |
| `69 83` | PIN blocked                                                      |
| `69 84` | no PIN set                                                       |
| `69 85` | conditions not satisfied: a slot already spent, no payment begun |
| `69 86` | card locked                                                      |
| `6A 80` | wrong data                                                       |
| `6A 82` | no application with this AID                                     |
| `6A 83` | slot out of range                                                |
| `6A 84` | no free slot or change opening                                   |
| `6A 86` | wrong P1 or P2                                                   |
| `6A 88` | slot empty, or no signature to give again                        |
| `6A 8C` | no mint and unit set                                             |
| `6A 8E` | a dated proof on a card with no refund key                       |
| `6A 8F` | over a spending limit                                            |
| `6A 94` | a proof already on the card, or the PIN needed for this payment  |
| `6A 96` | too many slots for one signature                                 |
| `6D 00` | instruction not supported                                        |
| `6E 00` | class not supported                                              |
| `6F 00` | signing failed                                                   |

Profiles **MAY** define further status words in the `6A 9X` range.

## Profiles

Instructions `18`, `32` to `34`, `36`, `37`, `41` to `45` and `50` to `53` are assigned to profiles, which define them. The profiles are:

- **Owner**: a P-256 owner key that administers the card, with signed commands for the card's mint and unit, limits, PIN changes, loading grants, locking and reset.
- **Clock and limits**: a daily limit, a per-payment limit and a no-PIN allowance, kept against a clock taken from Bitcoin block headers.
- **NFC tag**: an NFC Forum Type 4 tag carrying a web address.

Until each is written as its own document, they are specified in the reference implementation's [command reference](https://github.com/cashucard/cashucard/blob/main/spec/APDU.md).

## Mint support

A mint that supports [NUT-10][10], [NUT-11][11] with `SIG_ALL` on swaps, and [NUT-12][12] supports cards. It **SHOULD** advertise NUT-11 and NUT-12 in its info ([NUT-06][06]), since readers check DLEQ proofs offline.

## Security considerations

- **The card signs only what it spends.** Every signature is a payment's, released in the same atomic write that marks its inputs spent, or the `AUTH` signature, whose tagged message no secret can equal.
- **`P_card` locks the card's change outputs to the card.**
- **A reader can direct the payment's other outputs.** The holder trusts the reader the card is presented to for the amount; card policy bounds what one reader can take.
- **Offline acceptance carries double-spend risk** for the reader ([Accepting a payment offline](#accepting-a-payment-offline)).
- **A lost card is lost money**, within whatever its card policy allows to a finder.

## Test vectors

See [tests/CCP-tests.md][tests].

[00]: https://github.com/cashubtc/nuts/blob/main/00.md
[02]: https://github.com/cashubtc/nuts/blob/main/02.md
[03]: https://github.com/cashubtc/nuts/blob/main/03.md
[06]: https://github.com/cashubtc/nuts/blob/main/06.md
[09]: https://github.com/cashubtc/nuts/blob/main/09.md
[10]: https://github.com/cashubtc/nuts/blob/main/10.md
[11]: https://github.com/cashubtc/nuts/blob/main/11.md
[12]: https://github.com/cashubtc/nuts/blob/main/12.md
[tests]: tests/CCP-tests.md
