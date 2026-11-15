# NUT-CCP: Cashu Card Protocol

`optional`

`depends on: NUT-00, NUT-02, NUT-03, NUT-10, NUT-11`

`uses: NUT-09, NUT-12`

---

This NUT specifies how a smart card holds Cashu proofs and spends them over NFC. The card makes a secp256k1 key pair at install and keeps the private key. Every proof on the card is locked with [NUT-11][11] P2PK and the `SIG_ALL` flag to a key the card derives from its own for the proof's date ([Keys per load](#keys-per-load)), so that a mint sees no key that ties one date's proofs to another's, or to the card. To pay, a reader (a point-of-sale terminal or a wallet) builds a [NUT-03][03] swap from the card's proofs, and the card signs the whole swap with one BIP-340 signature by that key, marking each proof spent as it signs. The reader can prepare the swap while offline from the mint, and the card holder can pay without a phone at the point of sale.

Mints accept card payments as ordinary swaps of `SIG_ALL` proofs.

This document specifies the core every card and reader share: selecting the card, the format of its proofs and their keys, the card's unit, reading them, paying, the card's own change, and loading proofs. How a card is administered (an owner key, a PIN's management, spending limits and a clock) is left to profiles, listed under [Profiles](#profiles).

## Terms

- **Card**: an ISO/IEC 14443-4 contactless device running an application that implements this NUT.
- **Reader**: a device that speaks ISO/IEC 7816-4 commands to the card. A point-of-sale terminal paid by the card is a reader, and so is a wallet that loads it.
- **Holder**: the person who carries the card and knows its PIN.
- **Card key** (`P_card`, private `d`): the card's secp256k1 key pair. The card makes it on itself and **MUST NOT** export the private key. No proof is locked to it ([Keys per load](#keys-per-load)).
- **Refund key** (`R`, private `r`): a key the holder's wallet gives the card when it is set up, which can take back dated proofs after their date ([NUT-11][11] `refund`). A card may have none.
- **Card secret** (`c`): 32 bytes the card draws for itself, or takes from its owner's wallet, made from the wallet's seed, where the Owner profile gives it ([Profiles](#profiles)); the keys of every date are derived from it ([Keys per load](#keys-per-load)).
- **Date**: a proof's [NUT-11][11] locktime, in seconds, or `0` for a proof with none.
- **Date key**: the card's key for a date, `P_card + t·G`; the **date refund key** is `R + u·G` ([Keys per load](#keys-per-load)).
- **Card proof**: a `Proof` ([NUT-00][00]) whose secret is the P2PK secret defined in [Card proofs](#card-proofs).
- **Slot**: the card's storage for one card proof.
- **Tap**: one session between a card and a reader, from selection until the card leaves the field.
- **Opening**: what the card keeps of a change output it made for itself until the change is back on the card ([Change](#change)).

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

The AID is the proprietary RID byte `F0`, the ASCII string `CASHUCARD`, and `01`. The card answers two version bytes. A major byte of `02` identifies a card of this NUT.

The minor byte names the revision of this NUT the card follows; this document is revision `03`. A revision adds commands and fields, and may change what a command an earlier revision defined takes or answers; this document says where it does. Where a revision changes what a command takes, the card refuses the earlier form (`6A 86` or `67 00`) before it reads a slot, writes or signs anything, so a reader of an earlier revision is stopped with nothing spent or written on the card. Revision `03` changed `GET_CHANGE`'s rows and `SPEND_ALL_CHANGE`'s data and added `LOAD_CHANGE` ([Change](#change)), and made every proof's key a date key: `SPEND_ALL_BEGIN`'s P1 and answer and `LOAD_PROOF`'s P2 ([Keys per load](#keys-per-load)).

- A reader **MUST NOT** refuse to read a card for a minor byte it does not recognise.
- A reader **MUST** read the version before it swaps ecash for a card, and **MUST NOT** swap ecash for a card whose minor byte names a revision it does not implement: the swap is made before the card can refuse the load, and ecash locked to keys the card does not sign with is lost ([Loading proofs](#load_proof)).
- A reader **MUST NOT** send `LOAD_CHANGE` to a card that answers less than `03`, and **MUST NOT** read `GET_CHANGE` from it as this document lays the answer out.

A reader **MUST** select the full AID. A selection by the first ten bytes alone can reach a different application on the same chip. A response to `SELECT` that is not exactly two bytes beginning `02` **MUST** be treated as no card of this NUT, and so **MUST** a `6A 82` status word.

## Card proofs

Every card proof has a [NUT-10][10] P2PK secret whose `data` is the card's date key for the proof's date and whose tags end with `["sigflag","SIG_ALL"]`. A proof without a date has this secret:

```
["P2PK",{"nonce":"<nonce>","data":"<date key>","tags":[["sigflag","SIG_ALL"]]}]
```

A proof with a date also carries the locktime and the date refund key ([NUT-11][11]):

```
["P2PK",{"nonce":"<nonce>","data":"<date key>","tags":[["locktime","<date>"],["refund","<date refund key>"],["sigflag","SIG_ALL"]]}]
```

The secret **MUST** be serialized exactly as shown: no whitespace, keys and tags in this order, `<nonce>` as 64 lowercase hex characters, `<date key>` and `<date refund key>` as 66 lowercase hex characters of a compressed point, and `<date>` in decimal. The card stores the nonce and date and rebuilds each secret with this serialization and the keys of the date. Payment signatures follow the `SIG_ALL` transcript defined below.

[NUT-11][11] requires every input of a `SIG_ALL` swap to have the same `data` and tags. The proofs of one date share their keys, so the proofs of one payment **MUST** share their date. The card refuses a payment that mixes dates.

A wallet puts proofs on a card by an ordinary [NUT-03][03] swap whose outputs are card proofs, then writes each proof to the card with [`LOAD_PROOF`](#load_proof). The card's own change goes back with [`LOAD_CHANGE`](#load_change).

### Keys per load

Every card of revision `03` or later locks its proofs this way; it is part of the card proof format, not a profile a card may leave out, and the card says so in `GET_INFO` P1 `04` byte 56 bit 1 ([GET_INFO](#get_info)).

- **The card secret.** The card draws `c` (32 bytes) at install, and a new `c` whenever the card is handed on: at a reset, and where a profile lets an owner be replaced, at the replacement. Where the Owner profile's claim or replacement brings the owner's `c`, made from the owner's wallet's seed so that a wallet restored from its seed has it, the card takes that `c` instead, made new by a count of the card's own ([Profiles](#profiles), **Owner**). The card never gives `c` in the clear.
- **The keys of a date.** For a date `D`, as the four bytes a slot keeps (`00000000` for a proof with none):

  ```
  t = SHA-256( "CashuCard/key" || c || D ) mod n           date key        = P_card + t·G
  u = SHA-256( "CashuCard/refund-key" || c || D ) mod n    date refund key = R + u·G
  ```

  with the labels in ASCII and `n` the order of secp256k1. A proof of date `0` names no refund key. The card signs a payment of date `D` with `d + t mod n`; its own change is of the payment's date, so under the same keys. The holder of `r` and `c` spends the refund path of a dated proof with `r + u mod n` after its date.

- **Never lock ecash to `P_card`.** A sender, wallet or tool **MUST NOT** lock ecash to `P_card`, the key `GET_PUBKEY` answers and a certificate names. The card signs payments only with a date key, never with `d` alone, so ecash locked to `P_card` can never be spent by the card: undated, it is lost; dated, it comes back to the refund key after its date. `P_card` is for checking the card ([AUTH](#auth), [certificate](profiles/certificate.md)), not for locking money.
- **Who learns which keys.**
  - A paying reader takes `t` and the date refund key for its payment's date from `SPEND_ALL_BEGIN` ([Paying](#paying)) and works out the date key from `P_card`.
  - The holder's wallet reads `c`, sealed, with `GET_CARD` P1 `03` ([GET_CARD](#get_card)), and works out every date's keys itself. It keeps `c` with `r`, since a refund needs both. It reads `c` at set-up, again after a reset or an owner replaced (each gives the card a new `c`), and, while it holds `r`, at any tap the holder authorises. A wallet that gave the card the owner's `c` works it out again from its seed and the card's key `P_card`, with the card lost too; to take back a lost card's dated proofs it also needs `P_card` and the proofs themselves (their secrets and the mint's signatures), which no seed path gives, so it keeps both where its restore reaches them.
  - Any other loader asks the card for the keys of the date it loads, with a profile's command (the reference implementation's `GROUP_KEYS`, INS `1D`, under the PIN given in the tap or the owner's grant; [Profiles](#profiles)).
- **The reader's word.** A reader says it builds secrets with the date keys: `SPEND_ALL_BEGIN` P1 bit 1, `LOAD_PROOF` and `LOAD_CHANGE` P2 bit 1. The card cannot check how a proof was locked; it refuses a command without the word (`6A 86`) before it reads a slot or the data, so a reader that would lock to `P_card`, or build secrets with it, spends nothing and writes nothing.
- **The load date.** `D` is the proofs' locktime: the time from which the refund key can spend them. A wallet picks `D` in the future, never the time of the load, which would open the refund at once: the holder's wallet could then spend proofs still on the card, and a reader paid by the card offline would lose. Top-ups share one date for 7 days: a wallet reuses the date of the week's first load for every load within 7 days of it, and takes a new date after that. So `D` is at least 8 days and the time to move ahead at its first use (the week of loads, the readers' day below, and the time the wallet takes to move the proofs; for example 30 days). The mint can tie the spends of one week's proofs to each other, never across weeks or to the card.
- **Before the date.** The card does not read dates: its change keeps the payment's date, and a proof stays payable after its date. The holder's wallet moves the proofs of a date to a new date (it pays them to itself and loads the swap's outputs under the new date) so that the move ends before `D` less a day (the readers' margin below), starting it at least the time the move takes before that: from `D` less a day every honest reader, the holder's own wallet among them, leaves those proofs out of a payment, the move's too, and they wait on the card until the owner takes them back a day after `D`. A reader, online or not, **MUST** leave out of a payment a dated proof whose locktime is less than a day after its own time, or will pass before it can swap ([Paying](#paying), [Accepting a payment offline](#accepting-a-payment-offline)): the clocks of readers and of the owner's wallet differ, and the owner takes proofs back only a day after their date ([Profiles](#profiles), Inbox).
- **Undated proofs** are all of the date `0`, one group for the card's life (until a new `c`): the mint ties every undated spend of the card to one key. A card with no refund key takes only undated proofs (`6A 8E`), and gains nothing from keys per load; the gain is a wallet's that loads dated proofs.
- **Before a top-up**, a wallet that keeps `P_card`, `R` and `c` compares the card's keys for the date it is about to load (the profile's command above) with the ones it works out itself. A card reset since, or whose owner was replaced, has a new `c` and answers other keys.
- **A gift goes to the holder's wallet**, which loads it. A sender who is not the holder has no way to lock ecash to the card: the date keys are the holder's.

Nothing about a mint changes: the secrets are ordinary [NUT-11][11] P2PK, every input of a swap has the same key and tags, and the mint checks the signature against `data`.

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

### Units

A card holds the proofs of one mint. Its unit is named by one byte, in [`GET_CARD`](#get_card) and wherever a profile names a unit:

| Code | Unit  | Amounts are counted in |
| ---- | ----- | ---------------------- |
| `00` | `sat` | satoshis               |
| `01` | `usd` | cents                  |

Other codes are reserved. Amounts on the card, its balance and any limit a profile keeps are in the card's unit: on a `usd` card, cents. A card of proof format `05` also holds proofs of the other unit at the same mint, told apart by keyset; how is a profile's ([Profiles](#profiles)). Its proofs are format `04`'s card proofs, and its balance (`GET_BALANCE`) is the sum of the proofs of the card's unit alone. A reader that does not implement that profile **MUST** refuse a card whose format is `05`.

## Reading the card

These commands need no authorisation.

| INS  | Command           | Response                                                          |
| ---- | ----------------- | ----------------------------------------------------------------- |
| `01` | `GET_INFO`        | the card's state ([GET_INFO](#get_info))                          |
| `10` | `GET_PUBKEY`      | `P_card`, 33 bytes                                                |
| `11` | `GET_BALANCE`     | the sum of the unspent proofs' amounts, 4 bytes ([Units](#units)) |
| `12` | `GET_PROOF_COUNT` | the number of slots that are not empty, 1 byte                    |
| `13` | `GET_PROOF`       | the slot P1 (0 to 127), 82 bytes                                  |
| `14` | `GET_SLOT_STATUS` | one status byte per slot, 128 bytes                               |
| `15` | `AUTH`            | proof that the card holds `P_card` ([AUTH](#auth))                |
| `16` | `GET_CARD`        | the card's mint and unit ([GET_CARD](#get_card))                  |
| `17` | `GET_PIECES`      | several slots in one response ([GET_PIECES](#get_pieces))         |
| `19` | `GET_CHANGE`      | the card's own change still owed ([Change](#change))              |
| `1A` | `GET_DLEQ`        | the mint's DLEQ proofs kept with proofs ([DLEQ](#dleq))           |

`GET_PROOF` and `GET_DLEQ` answer `6A 83` for a slot out of range, and `GET_PROOF` answers `6A 88` for an empty slot.

A card **MAY** keep its spent slots as the holder's history, said only to a reader with the card's authorisation in the tap (card policy, [Profiles](#profiles)). To any other reader such a card answers a spent slot as an empty one: `GET_PROOF` answers `6A 88`, `GET_PIECES` does not list it or answers 82 zero bytes for it. The counts of spent slots (`GET_INFO` byte 4, `GET_SLOT_STATUS`) are said to anyone. A reader that reads spent slots, to [reclaim them](#reclaiming-spent-slots), gives the card's authorisation first. `GET_CARD` P1 `03` is the one reading form that always wants authorisation ([GET_CARD](#get_card)).

### GET_INFO

P1 `00` returns 30 bytes, P1 `01` returns 42 and P1 `02` returns 43; P1 `03` and `04`, whose further bytes belong to profiles, return 51 and 59. The fields this NUT defines:

| Byte  | Field                                                                                                    |
| ----- | -------------------------------------------------------------------------------------------------------- |
| 0     | version major, `02`                                                                                      |
| 1     | version minor                                                                                            |
| 2     | number of slots                                                                                          |
| 3     | unspent slots                                                                                            |
| 4     | spent slots not yet freed                                                                                |
| 5     | empty slots                                                                                              |
| 6     | capabilities                                                                                             |
| 7     | PIN state: `0` none, `1` set, `2` blocked                                                                |
| 8     | proof format: `04`, or `05` for a card of two units ([Units](#units))                                    |
| 9     | PIN tries left                                                                                           |
| 10    | `1` when the card is locked and takes no more proofs                                                     |
| 11    | `1` when the card has a record of its mint and unit ([GET_CARD](#get_card))                              |
| 29    | `1` while an opening of the card's own change is owed ([Change](#change))                                |
| 55    | P1 `04`: profile bits ([Profiles](#profiles))                                                            |
| 56    | P1 `04`: profile bits; bit 1 is keys per load, set by every card of revision `03` or later               |
| 57–58 | P1 `04`: the proofs the card will still load today, where it bounds them ([Loading proofs](#load_proof)) |

The other bytes from 12 belong to [profiles](#profiles). A reader that implements no profile **MUST** ignore them. A card that answers fewer than 57 bytes to P1 `04` keeps none of the profiles. Bytes 57 and 58 are said to a reader with the card's authorisation in the tap, and are zeros to any other, which **MUST NOT** read them as a number: they do not say that no proof is left to load.

### GET_CARD

P1 `00` (any P1 but `02` and `03`): the response is 109 bytes, then the mint URL, then 3 bytes:

| Offset  | Size | Field                                            |
| ------- | ---- | ------------------------------------------------ |
| 0       | 1    | proof format, `04` or `05`, as `GET_INFO` byte 8 |
| 1       | 1    | `1` when the fields below are set                |
| 2       | 1    | unit ([Units](#units)): `00` `sat`, `01` `usd`   |
| 7       | 33   | the refund key `R`, or zeros                     |
| 105     | 1    | the mint URL's length, at most 77                |
| 106     | n    | the mint URL, UTF-8                              |
| 106 + n | 3    | a design code chosen by the issuer, or zeros     |

Bytes 3 to 6 and 40 to 104 belong to [profiles](#profiles). A reader **MUST** take every proof on the card as issued by the mint at this URL, in this unit. A mint URL of `-` alone is a factory record: an issuer may write one to carry the card's design before the card is set up. It names no mint, and the card takes no proof under it (`6A 8C`). P1 `02` belongs to a profile ([Profiles](#profiles)).

**P1 `03`: the card secret `c`, sealed.** Only to a reader with the holder's authorisation in this tap, as card policy defines it (`69 82` otherwise; the reference implementation: the owner's grant on a card with an owner, and on one with none the PIN given in the tap). The reader first asks `GET_NONCE` (INS `44`) P1 `01`, a profile command, for 16 fresh bytes of the card's, which `GET_CARD` P1 `03` uses up. Its answer depends on the card's record:

- **A card with a refund key `R`.** The reader sends no data. The card draws a fresh `k` and answers 97 bytes: `K = k·G` (65, uncompressed) and `c XOR S`, where

  ```
  S = SHA-256( "CashuCard/seal" || 01 || x(k·R) || K || nonce || 16 )
  ```

  with `x(k·R)` the 32-byte x of the point, `nonce` the 16 bytes and `16` the instruction byte. The holder of `r` opens it with `r·K = k·R`. A card **MUST** give `c` so to the holder at any tap it authorises, so that a wallet restored from its seed, or one that missed the set-up, reads it again, as long as it has `r`.

- **A card with no refund key.** The reader sends a key of its own for this one message, `E` (65, uncompressed), and the card answers 32 bytes, `c XOR S` with `S` as above over the x that `E` and the card's PIN key share (the PIN key is the 33 bytes after the 16 in `GET_NONCE` P1 `01`'s answer, signed there by the card key) and `E` in `K`'s place. Whoever the PIN is typed into can open this form, so a card **MAY** give it only in its first set-up (the reference implementation does so on a card with no owner). The card **MUST** refuse an `E` that is not a point of the curve (`6A 80`) before its key agreement sees it; so too for every sealed command of a profile, where an envelope whose `E` is not a point does not open.

It answers `69 82` without authorisation, `67 00` (no refund key) for data that is not 65 bytes, `69 85` with no 16 bytes live, and `6A 80` (no refund key) for an `E` that is not a point; the 16 bytes are used up by an answer, by `69 85` and by `6A 80`. A listener on the air learns nothing of `c`. A reader **MUST** check, before it takes `c`, that the card set `GET_INFO` P1 `04` byte 56 bit 1, and that the answer is 97 bytes on a card with a refund key, 32 on one with none: an earlier card ignores P1 and answers its record.

### GET_PIECES

P1 is the first slot to report. P2 `00` returns one page of at most 255 bytes: the first slot the page does not cover (`128` at the end), then, for each slot from P1 that is not empty, a tag byte (the slot number, with bit 7 set when the slot is spent) followed, for an unspent slot, by its 81 bytes after the status byte. P2 `02` takes one to three slot numbers as data and returns each slot's 82 bytes in the order asked. P2 `03` is a short listing a profile defines. Another P2 answers `6A 86`. A card that keeps spent slots as history lists them only as [Reading the card](#reading-the-card) says.

### AUTH

The reader sends 16 random bytes. The card answers with 16 random bytes of its own and a BIP-340 signature by `P_card` over

```
SHA-256( T || T || reader_nonce || card_nonce || P_card ),  T = SHA-256("CashuCard/auth")
```

with `P_card` compressed. A reader that verifies it knows the card present holds `P_card`, the key every date key is built on ([Keys per load](#keys-per-load)). The distinct tagged message authenticates possession of the private key corresponding to `P_card` without spending a proof.

## Paying

A payment is one tap with these commands, in this order:

| INS  | Command             | Data                                                                          | Response                                                                |
| ---- | ------------------- | ----------------------------------------------------------------------------- | ----------------------------------------------------------------------- |
| `22` | `SPEND_ALL_BEGIN`   | the slots to spend, one byte each                                             | their total amount (4), `t` (32) and the date refund key (33), 69 bytes |
| `23` | `SPEND_ALL_OUTPUTS` | per output: amount (4) and `B_` (33)                                          | none                                                                    |
| `26` | `SPEND_ALL_CHANGE`  | the change amount (4), the mint's key for it (33) and the change's keyset (8) | the change's `B_`, 33 bytes                                             |
| `24` | `SPEND_ALL_SIGN`    | none                                                                          | the signature, 64 bytes                                                 |

`SPEND_ALL_BEGIN` takes P1 with bit 1 set (`02`): the reader's word that it builds every input's secret with the date keys ([Keys per load](#keys-per-load)). Without it the card answers `6A 86` before it reads a slot, and nothing is spent or written. Bit 0 (`03`) belongs to a profile ([Units](#units)); the other bits are not read. The answer gives `t` for the payment's date, the date of its slots, and the date refund key, or 33 zero bytes for a payment of date `0` and on a card with no refund key. The reader works out the date key `P_card + t·G` from `GET_PUBKEY` and builds each input's secret with it.

`SPEND_ALL_OUTPUTS` **MAY** be sent more than once. P1 `00`: the data is the outputs, a multiple of 37 bytes. P1 `01`: the outputs' 8-byte keyset, then the outputs (8 + 37·k bytes); a reader uses `01` when its outputs are not in the first input's keyset, so that a card that keeps a record of the payment files them under the right keyset. A reader sends no other P1. `SPEND_ALL_CHANGE` **MAY** be sent more than once or not at all. The reader's outputs and the card's change together **MUST NOT** come to more than the inputs' total: the card refuses an output or a change that would take them past it (`6A 80`) and gives the payment up. Any other command between `SPEND_ALL_BEGIN` and `SPEND_ALL_SIGN` abandons the payment, `VERIFY_PIN` among them.

**Fees.** The card does not know the mint's fees and checks none. At a keyset with an input fee ([NUT-02][02] `input_fee_ppk`), the mint takes the swap only when the outputs come to the inputs' total less the fee, `ceil(sum of each input's input_fee_ppk / 1000)`. A reader **MUST** work the fee out from the mint's keysets and make the reader's outputs and the card's change together come to exactly the inputs' total less the fee. The card spends the slots when it signs, whatever the outputs come to, and a swap the mint refuses for its amounts cannot be undone on the card: an undated proof so spent is lost, since only the card could sign for it, a dated one can be spent by the refund key after its date, and the payment's change stays owed ([Change](#change)).

**Inputs.** The card spends every slot named when it signs, and does not read dates. So a reader, online or not, **MUST** leave out of a payment a dated proof whose locktime is less than a day after its own time, or will pass before it can swap: the refund key may have spent it, and the mint would refuse the swap with every other proof of the payment spent on the card. An online reader **SHOULD**, before `SPEND_ALL_SIGN`, check that every input's keyset is a keyset of the mint `GET_CARD` names, in the card's unit, and ask that mint every input's state ([NUT-07][07]), and give the payment up for a proof the mint does not answer `UNSPENT` for, so that one bad proof never burns good ones.

The card signs, with BIP-340 and the date key, the SHA-256 of the [NUT-11][11] `SIG_ALL` swap message:

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
- **MUST NOT** sign, with `P_card` or any date key, a message that could be a payment's other than the payment it releases. Every other signature the card makes is over a tagged message: the SHA-256 of an input that begins with a label of the card's own, or with the SHA-256 of one as `AUTH`'s does. Profiles define such messages (the reference implementation's are the PIN key's statement under `CashuCard/pinkey` and the payment record's under `CashuCard/assisted-settlement/v1`). A payment's message begins with a proof's secret, whose text begins `["P2PK"`, and so is none of them.
- **MUST** answer `SPEND_ALL_SIGN` with `00 01` and status `90 00` when it needs more time. The reader **MUST** send `SPEND_ALL_SIGN` again until it receives the signature or an error. A card **MAY** make every payment wait so, within its limits or not (the 2.3 applet does not: a payment within its limit, the first in its time in the field, is signed at its first `SPEND_ALL_SIGN`).
- **MUST** answer `SPEND_ALL_AGAIN` (INS `25`, P1 `00`) with the last signature it released, for a reader that lost the response. This **MAY** need the same authorisation as the payment did, and a card that wants it answers `69 82` before `6A 88`.
- **MAY** answer `SPEND_ALL_AGAIN` P1 `01`, with the SHA-256 of a payment's message as its 32 bytes of data, with the signature of that payment among those it keeps (`6A 88` for a hash it does not keep). Without the authorisation, a card answers only for payments it signed with none, and `6A 88` for every other hash: a reader without it learns of no payment under the PIN that it was the card's.

`SPEND_ALL_BEGIN` answers `6A 86` without P1 bit 1; `67 00` for no slot; `6A 96` for more slots than one signature can cover; for each slot in the order named, `6A 83`, `6A 88` or `69 85` for a slot out of range, empty or spent, and `6A 80` for a slot named twice or of another date than the first slot's; `6A 94` or `6A 8F` where the slots' amounts together pass what four bytes hold. A profile **MAY** refuse before these (a card with no PIN, `6A 97`; a tap that claimed an owner or read a history, `69 85`) and, among each slot's own checks, after them: two units in one payment (`6A 9A`: the first slot of no unit, or a later slot of another unit than the first's), and a payment of the other unit by a reader that reads no unit (`6A 9B`, at the first slot). A slot's `6A 9A` or `6A 9B` comes before a later slot's `6A 83`, `6A 88`, `69 85` or `6A 80`.

`SPEND_ALL_BEGIN` answers before the card asks for a PIN or applies a limit, because a reader needs `t` to check the proofs before it asks the holder for the PIN. A reader that does so then presents the PIN and begins the payment again ([Card policy](#card-policy)). So any reader holding the card learns the date key and the date refund key of every date with a proof on the card, with no authorisation, and gives the payment up unsigned ([Security considerations](#security-considerations)).

### Change

The card makes change for itself, and only the card can finish it. After the reader's outputs, `SPEND_ALL_CHANGE` names the change amount (4), the mint's public key `K` for that amount (33, compressed), and the 8 bytes of the keyset the change is to be signed in (45 bytes in all): the mint's active keyset of the payment's unit, which the reader posts the change output under. The card builds a card proof secret with a fresh nonce and the payment's date, so under that date's keys, computes `Y = hash_to_curve(secret)` ([NUT-00][00]), blinds it with a fresh `r`, and returns `B_ = Y + rG`. It keeps an **opening**: the amount, the keyset, the date, the nonce, `r`, `B_` and `K`, which it is then **bound** to. The change amount **MUST** be greater than 0 (`6A 80`); a `K` that is not a compressed point of the curve is `6A 80`, any length but 45 `67 00`; the card answers `6A 84` when it has no room to keep another opening. A profile **MAY** refuse a keyset whose keys the card has not been shown (`6A 9F`) or of another unit than the payment's (`6A 9B`). A card that bounds the proofs it loads in a day answers `6A 8F`, last, for change past what the day has left with the change before it in the payment ([Loading proofs](#load_proof)); a reader then begins the payment again with proofs that need no change, or no more change outputs than the card made before the refusal, or pays after the day has turned. That `6A 8F` comes before any authorisation is asked, so it tells anyone holding the card how many loads are left today where fewer than 8 are. Where the card sets `GET_INFO` P1 `04` byte 55 bit 6 ([Profiles](#profiles), keyset keys), a reader **MUST** teach the card the change keyset (`KEYSET_KEYS` P1 `00` to `02`, under the PIN given in the tap) before a payment makes change, and show the bound key (`KEYSET_KEYS` P1 `03`) before `LOAD_CHANGE`: without them the change is refused, or stays owed. Every refusal gives the payment up. `SPEND_ALL_OUTPUTS` after `SPEND_ALL_CHANGE` is `69 85`: the reader's outputs come first. Revision `02` took the amount alone (4 bytes); a card of revision `03` answers `67 00` to it, and to 37 bytes with no keyset.

An opening is a draft until the payment is signed, and is **owed** from the atomic write that releases the signature until its proof is on the card. A draft of a payment never signed is let go at the next `SPEND_ALL_BEGIN`. A card keeps at most 8 openings, those owed by earlier payments among them, and makes a change output for each opening it has free.

The card **MUST NOT** give `r` or the nonce of an opening to a reader while the change is owed. `GET_CHANGE` (INS `19`, P1 a page from 0) returns a count byte and up to three owed openings of 83 bytes:

| Size | Field                                            |
| ---- | ------------------------------------------------ |
| 1    | the opening's number, 0 to 7: `LOAD_CHANGE`'s P1 |
| 4    | amount                                           |
| 8    | keyset                                           |
| 4    | date, or `0`                                     |
| 33   | `B_`, compressed                                 |
| 33   | `K`, the key the opening is bound to, compressed |

An opening whose proof is on the card already is let go before the answer. The mint's swap response contains a blind signature `C_` for each change output; a reader that missed it **MAY** recover `C_` with [NUT-09][09] restore, using `B_`.

<a id="load_change"></a>

### LOAD_CHANGE

`LOAD_CHANGE` (INS `27`, P1 the opening's number, P2 `02`) gives the card the mint's answer for one opening, 194 bytes:

| Size | Field                                                         |
| ---- | ------------------------------------------------------------- |
| 65   | `C_`, the mint's blind signature, uncompressed (`04 ‖ x ‖ y`) |
| 65   | `K`, the mint's key for the amount, uncompressed              |
| 32   | `e` of the mint's [NUT-12][12] DLEQ proof                     |
| 32   | `s` of the mint's [NUT-12][12] DLEQ proof                     |

P2 bit 1 (`02`) is the reader's word on keys per load, as `LOAD_PROOF` takes it; the other bits are not read. The card:

1. refuses a `K` that is not the key the opening is bound to, its x and the parity of its y (`6A 80`), before any arithmetic;
2. refuses a `C_` or a `K` that is not a point on the curve (`6A 80`);
3. verifies the DLEQ proof against `K`, its own `B_` and `C_` as [NUT-12][12] specifies (`R1 = sG - eK`, `R2 = sB_ - eC_`, and `e` the NUT-12 hash of `R1`, `R2`, `K` and `C_`), and refuses one that does not hold (`6A 80`);
4. unblinds the proof, `C = C_ - rK`, and writes it to an empty slot as `LOAD_PROOF` writes a proof: the opening's keyset, amount, nonce and date, and `C`, with the status byte last;
5. keeps `e`, `s` and `r` as the proof's DLEQ proof ([DLEQ](#dleq)) where it has room, and lets the opening go.

It answers 83 bytes: the slot (1), `01` when the card kept the DLEQ proof or `00` when it had no room (1), and the slot's 81 bytes after the status byte. In order, it answers `6A 86` without P2 bit 1, before anything is read; `6A 83` for a number past 7; `6A 88` for an opening that is not owed; `6A 94` for an opening whose proof is on the card already (the opening is let go); `6A 84` when no slot is free; `67 00` for any length but 194; then the `6A 80` of the steps above. A card that bounds the proofs it loads in a day **MUST NOT** refuse `LOAD_CHANGE` for that bound: the payment counted its change when it was signed ([Loading proofs](#load_proof)). A profile **MAY** refuse before `6A 86` (a card with no PIN, `6A 97`) and before `6A 80` (a key not shown, below).

A card **MUST** take `LOAD_CHANGE` under the bound key from any reader, with no authorisation, on a locked card and one whose PIN is blocked as well: receiving what it is owed changes nothing of the card's settings. A profile **MAY** first require a proof, which any reader can give, that the bound key is one of its keyset's keys (`6A 9F` until it is given). A card **MUST NOT** write a change proof whose DLEQ proof does not hold under the bound key.

The card cannot tell that `K` is the mint's key. A reader that binds an opening to a key that is not the mint's makes change the mint's answer cannot fit, and the opening stays owed. While any opening is owed a card **MUST NOT** change the mint, the unit, the refund key or `c` that the opening's secret was built with. A card **SHOULD** let its holder give up an opening nothing can load, and **MAY** let its holder bind an opening to another key, under authority a profile defines; profiles also define how a card holds keys to their keyset ID ([Profiles](#profiles)).

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

P2 bit 1 (`02`) is the loader's word that each proof is locked to the card's keys for its date ([Keys per load](#keys-per-load)). Without it the card answers `6A 86` before the data is read, with nothing written. P2 bit 0 (`03`) puts a unit byte before the proofs, which a profile defines ([Units](#units)). P2 bit 3 (`08`) sends one proof a command with `C` whole, `04 || x || y` (65 bytes in place of 33: 113 bytes, 209 with the DLEQ proof): a card that takes this form **MUST** check the length, the `04`, and x and y below the field's prime itself, and **MUST** hold the point to the curve before it writes anything (`6A 80` otherwise), and stores the proof exactly as the same proof sent compressed. A card **MAY** leave the form out, or leave it to an engine of its chip that it has seen refuse a point off the curve; a card that does not take it answers `6B 00` before the data is read, and the reader sends the compressed form. A P2 with any other bit set is `6B 00`.

The card **MUST** refuse a nonce already in any slot, spent or not (`6A 94`), a `C` that is not a point of the curve (`6A 80`), a dated proof when it has no refund key (`6A 8E`), and any proof before its mint and unit are set, or under a factory record (`6A 8C`). It answers `6A 84` when no slot is free. In order: the card's policy first (a profile's authorisation, `69 82`); `6A 8C`; `6A 84` with no slot free; `6B 00`; `6A 86`; `6B 00` for bit 3 on a card that does not take it; `67 00` for a length that is not one to three proofs (or one and its DLEQ proof at P1 `01`; with bit 3, one proof); then for each proof, `6A 84`, `6A 8E`, `6A 80`, a profile's refusals of its unit, `6A 94`, and `6A 8F` last. It **MUST** write the status byte last, so a write cut short leaves the slot empty. Several proofs are taken in order, each as one alone would be: if a later one cannot be stored, the answer is the slots of those before it.

A card **MAY** bound the proofs it loads in a day. A card that does **MUST** count a payment's change against the bound when the payment is signed, and **MUST NOT** refuse `LOAD_CHANGE` for the bound, so that change owed is never stranded by it; change past what the day has left is refused at `SPEND_ALL_CHANGE` (`6A 8F`, the payment given up). Change counted on one day and loaded on the next is in neither day's loads, so a day may take up to the bound and 8 more (one for each opening). A `LOAD_PROOF` past the bound is `6A 8F`, with nothing written, and is taken on a later day. A card **MAY** refuse a proof larger than its daily limit (`6A 8F`), since no payment could take it off. A reader tells the two apart by the proof. A loader that swaps for a card:

- reads the version first ([Selecting the card](#selecting-the-card)) and the card's keys for the date it loads ([Keys per load](#keys-per-load));
- splits the top-up into proofs no larger than the card's daily limit before it swaps, since a proof already locked to the card cannot be split by anyone but the mint;
- swaps for no more proofs than the card's empty slots (`GET_INFO` byte 5) less up to 8 left free for the card's change (change that finds no empty slot stays owed until one is freed), and the proofs it will still load today (`GET_INFO` P1 `04` bytes 57 and 58), and prefers dated proofs: an undated top-up swapped past what the card takes waits with the loader, locked to the card's keys, and is lost if the card is reset or lost first;
- checks, before it swaps, that the card will take proofs of this mint's keyset, unit and date: a dated proof needs a refund key (`6A 8E`), and a card that keeps **Two units on one card** ([Profiles](#profiles)) takes a proof only where its keyset table has the keyset or room for it, in a unit the card takes, as that profile says (in the reference implementation: the table in `GET_INFO` P1 `03` bytes 43, 48 and 49, the unit in `GET_CARD`, and a rate where the card has a limit and the proof is of the other unit).

Who may load is card policy ([Card policy](#card-policy)). A card **MAY** require its PIN for every `LOAD_PROOF`, in the tap after a payment as in any other: the card's own change goes back with [`LOAD_CHANGE`](#load_change), which needs no authorisation.

`CLEAR_SPENT` (INS `31`) frees every spent slot and answers how many it freed. It is subject to the same policy as `LOAD_PROOF`.

### Reclaiming spent slots

A spent slot records that the card committed its proof to a payment. The mint may still report that proof as `UNSPENT` or `PENDING`, for example while the accepting terminal is offline. The retained proof and payment records may be needed to finish that payment.

A reader **MUST NOT** send `CLEAR_SPENT` as an unconditional part of loading proofs. Before clearing, it **MUST** read every retained spent proof from the card (with the card's authorisation, on a card that keeps spent slots as history), reconstruct each proof's secret with the keys of its date and query its mint with [NUT-07][07]. Every proof **MUST** be reported as `SPENT`. A `PENDING` or `UNSPENT` result, an unavailable mint, an incomplete or contradictory response, or a proof the reader cannot reconstruct **MUST** prevent clearing. The response **MUST** cover exactly the requested proof identifiers.

The reader **MUST** bind the check to the same card and proof contents before sending the clear. It **MUST** revalidate any snapshot obtained in an earlier card session. An empty terminal queue, a displayed zero balance or the card's own spent status **MUST NOT** substitute for mint verification. Because `CLEAR_SPENT` frees all spent slots, the check applies to all of them.

Mint consumption of the inputs does not establish that the merchant recovered its outputs or that the card received its change. Implementations **MUST** preserve the original payment and output-recovery records while those obligations remain unresolved. Clearing proof slots **MUST NOT** be interpreted as permission to discard those records or change openings. A reader resuming its own interrupted cleanup **MUST** retain the verified cleanup scope and reconcile the card state; malformed contents alone are not mint settlement evidence.

## DLEQ

`GET_DLEQ` (INS `1A`) takes a slot in P1 and a second slot or `FF` in P2. For each slot asked it returns `01` and the 96 bytes `e || s || r` kept with an unspent proof, or `00`. DLEQ validation is the reader's responsibility. The reader checks a proof by rebuilding the secret with the keys of its date, computing `Y = hash_to_curve(secret)`, `C' = C + rK` and `B' = Y + rG`, and verifying [NUT-12][12] with the mint's key `K` for the amount. A valid DLEQ proof shows the mint signed the proof. It does not show the proof is unspent. `C'` is the blind signature the mint issued: with the mint's records, `r` ties the swap that made the proof (the payment, for the card's own change; the holder's top-up, for a proof loaded with its DLEQ proof) to the card, for as long as the proof is on it.

## Card policy

A card **MAY** require authorisation for `LOAD_PROOF`, `CLEAR_SPENT`, `SPEND_ALL_SIGN` and `SPEND_ALL_AGAIN`, and **MAY** refuse or delay a payment under limits it keeps. It **MUST NOT** require authorisation for `LOAD_CHANGE` under the bound key. Profiles define those rules. A reader **MUST** handle these responses:

| Status  | Meaning                                                                                                                                                                                                    |
| ------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `69 82` | authorisation needed and not given in this tap                                                                                                                                                             |
| `6A 94` | (`SPEND_ALL_SIGN`) the PIN is needed for this payment; the payment is abandoned                                                                                                                            |
| `6A 9E` | (`SPEND_ALL_SIGN`) more than the no-PIN allowance has left; the PIN is needed; the payment is abandoned. Treat it as `6A 94`                                                                               |
| `6A 8F` | over a spending limit, nothing signed or spent; at a load, a proof over the daily limit or the day's loads used, nothing written; at `SPEND_ALL_CHANGE`, change past the day's loads, the payment given up |
| `6A 97` | the card has no PIN: it takes and pays nothing until it has one                                                                                                                                            |
| `6A 98` | keep holding: the card is counting a held tap; send the same command again                                                                                                                                 |
| `69 83` | the PIN is blocked                                                                                                                                                                                         |
| `69 86` | the card is locked                                                                                                                                                                                         |

A reader **MUST** take a status word in the `6A 9X` range that it does not know as a refusal. A reader that does not send a command again on `6A 98` takes it as a refusal; nothing was done.

A reader with a PIN from the holder presents it with `VERIFY_PIN` (INS `40`, P1 `00`, the PIN's 4 to 8 bytes as data) before `SPEND_ALL_BEGIN`: `VERIFY_PIN` between the payment's commands abandons the payment, and the `69 85` that `SPEND_ALL_SIGN` then answers means no payment is in hand, with nothing spent. A reader that asks for the PIN after `SPEND_ALL_BEGIN` sends `VERIFY_PIN` and begins the payment again. A wrong PIN answers `63 CX`, with `X` the tries left (on a card that makes each try wait instead of blocking, the wrong tries in a row); `69 84` means the card has no PIN. A card **MAY** make a try wait: it answers `6A 98` with no data, and the reader sends the same `VERIFY_PIN` again for as long as it does. A verified PIN lasts until the end of the tap.

## Accepting a payment offline

A reader that accepts a payment without reaching the mint:

- **MUST** check that each input's keyset is a keyset of the mint named by `GET_CARD`, in the unit the reader takes the payment in ([Units](#units));
- **MUST** check each input's DLEQ proof ([DLEQ](#dleq)) and **SHOULD** decline a proof without one;
- **MUST** refuse a dated input whose locktime is less than a day after the reader's own time, or will pass before the reader can swap it: the refund key may spend it first ([Keys per load](#keys-per-load));
- **SHOULD** check `AUTH`. The reader builds each input's secret with `P_card + t·G`, from the key `AUTH` proved and the `t` of `SPEND_ALL_BEGIN`, so a DLEQ proof that holds also shows the proof is locked to this card's key for its date;
- **MUST** swap at the mint as soon as it can, and **MUST NOT** report the payment settled until the mint accepts the swap.

The reader carries the risk that a proof was spent elsewhere before the card was presented. A copy of a proof can be spent elsewhere even though the card refuses to sign for a slot twice.

## Status words

| SW      | Meaning                                                                                                             |
| ------- | ------------------------------------------------------------------------------------------------------------------- |
| `90 00` | success                                                                                                             |
| `62 00` | success, a warning: lasting memory unchanged (a profile's `ALLOW_LOAD`: the grant given for the tap)                |
| `63 CX` | wrong PIN                                                                                                           |
| `67 00` | wrong length                                                                                                        |
| `69 82` | authorisation needed                                                                                                |
| `69 83` | PIN blocked                                                                                                         |
| `69 84` | no PIN set                                                                                                          |
| `69 85` | conditions not satisfied: a slot already spent, no payment begun, no nonce live                                     |
| `69 86` | card locked                                                                                                         |
| `6A 80` | wrong data                                                                                                          |
| `6A 82` | no application with this AID                                                                                        |
| `6A 83` | slot, or opening, out of range                                                                                      |
| `6A 84` | no free slot or room for another opening                                                                            |
| `6A 86` | wrong P1 or P2: among them a payment or a load without the word on keys per load                                    |
| `6A 88` | slot empty, no signature to give again, or an opening not owed                                                      |
| `6A 8C` | no mint and unit set, or a factory record                                                                           |
| `6A 8E` | a dated proof on a card with no refund key                                                                          |
| `6A 8F` | over a spending limit; at a load, a proof over the daily limit or the day's loads used; change past the day's loads |
| `6A 94` | a proof already on the card, or the PIN needed for this payment                                                     |
| `6A 96` | too many slots for one signature, or a write too large for the card's atomic write, nothing written                 |
| `6A 97` | no PIN: the card takes and pays nothing until it has one                                                            |
| `6A 98` | keep holding: send the same command again                                                                           |
| `6A 9E` | the PIN needed: more than the no-PIN allowance has left (`SPEND_ALL_SIGN`), as `6A 94`                              |
| `6B 00` | wrong P1 or P2 (a P2 with bits a command does not define)                                                           |
| `6D 00` | instruction not supported                                                                                           |
| `6E 00` | class not supported                                                                                                 |
| `6F 00` | signing failed                                                                                                      |

Profiles **MAY** define further status words in the `6A 9X` range.

## Profiles

Instructions `18`, `1B` to `1E`, `32` to `34`, `36` to `38`, `41` to `45` and `50` to `54` are assigned to profiles, which define them; `2E` and `2F` are kept for a later profile. A card says which profiles it keeps in `GET_INFO` P1 `04`, bytes 55 and 56 (the bit after each name below). The profiles are:

- **Owner**: a P-256 owner key that administers the card, with signed commands for the card's mint and unit, limits, PIN changes, the owner's grant for a tap, locking and reset, and a holder's way to give up an opening of change. An owner is replaced only on an empty card with no change owed. The replacement is a hand-over: it gives the card a new `c` (the new owner's, below, or one the card draws); clears the card's history as a reset does, but keeps the card key and its settings; lets the record's refund key go, so that `c` is never sealed to the last owner's key and no dated piece is taken until the new owner names its own refund key; and takes no PIN until the new owner sets one of its own. So neither owner can work out the other's keys or read the other's payments, and the last owner holds no refund key to the new owner's money.
  - **The owner's `c`.** The setting of an owner key on a card that has none (a claim), or the replacement, **MAY** give the card the new owner's `c`, so that a wallet restored from its seed has the card's `c` with no card. That alone does not take back a lost card's dated proofs: the wallet also needs the card's key `P_card` and the proofs themselves (their secrets and the mint's signatures), which it keeps where its restore reaches them. The wallet works out `s = HMAC-SHA256(seed, "CashuCard/c" || 0x00 || P_card)` (the 64-byte BIP-39 seed as the key, `P_card` compressed) and seals it to the card's PIN key `Q` (`GET_NONCE` P1 `01`, its signature by the card key checked first) with a key of its own for this one message, `E = e·G`: the card is sent `E || (s XOR B1) || B0[0..15]`, `Bi = SHA-256("CashuCard/c" || i || x(e·Q) || E [|| the 32 sealed bytes, for B0])`, `i` one byte, `B0[0..15]` the tag, (the reference implementation's `SET_OWNER` P1 `01`), and the claim's or replacement's data ends with `SHA-256("CashuCard/c" || s)[0..15]`, under the old owner's proof on a replacement. The wallet works `s` out from the card's own `P_card`, read at its first tap with the card (with its certificate, where it has one), never from a key it was handed. The card refuses an envelope that does not open, its tag not `B0`'s (`6A 80`) and a check that is not its `s`'s (`6A 80`; `69 85` with no `s` given), and takes `c = SHA-256("CashuCard/c" || s || n)`, `n` the number of owners' `c` it has taken, this one with them (four bytes, big-endian, from 1), a count it keeps for its life and never lowers: so it never takes a `c` it has had, for the same owner neither, and an old claim or replacement sent again never sets an old `c`. The wallet reads `c` after it ([GET_CARD](#get_card)) and finds the `n` that makes it; where none does, another party's `c` was taken (a reader in the claim's tap, or at a replacement the old owner's wallet, which signs the check and so can send an envelope of its own), and the wallet **MUST NOT** lock dated proofs to the card's keys or file proofs for it. A card with no owner, and a reset, draw their own `c`. Leaked words then give every such card's `c` as well: its dates' keys, and with `r` the refund of its dated proofs after their date ([test vectors](tests/CCP-tests.md#the-owners-c)).
- **Clock and limits**: a daily limit, a per-payment limit and a no-PIN allowance, kept against a clock taken from Bitcoin block headers, and a rollover budget that bounds how far the clock can be walked. With it: **a no-PIN window** (55 bit 1), in which the allowance is counted and which only the holder's PIN or the owner renews, never the clock alone; and **a wait on every payment** (55 bit 3). The clock moves forward with each later header, and back only in four ways: to the anchor of a reset or hand-over of an empty card (a block header the command carries, under the owner's proof, or with the holder's authorisation on a card with no owner), which the card takes even where it is older; to the first set-up's anchor of an empty card (the header it takes in the session that gives a card with no PIN its PIN, while it holds nothing: the claim's anchor, or a plain header on a card with no owner), taken so too; from a weak clock, to an older header that is not weak, in the owner's or the holder's refresh (the reference card's `REFRESH`, `53`); and, in the owner's refresh that asks it, to any older header that is not weak. A header is **weak** when it carries less than half the work of the hardest the card has taken, or, on a card that has taken none, less than twice the least work the clock takes; a weak header never becomes the hardest, and setting the card up or resetting it keeps the hardest. At a clock a weak header set, no day the card counts (its loads, its limits) ends, and none begins at it. A reader **MUST NOT** give a card a block header whose time is more than two hours ahead of its own clock (the network's own rule), and **SHOULD** give only headers of the best chain it has checked itself. A wallet that sets a card up **MUST** give it, in its first set-up and before any load, the newest header of the best chain it has checked, as the first set-up's anchor: the card takes it as its clock even where its clock is later, and as its hardest where it is harder, so a far header a stranger gave it before is let go, and a far header after it costs at least half that header's work. This binds honest readers only; anyone holding the card can give it a header. A header made for a time far ahead moves the clock there:
  - **at the least work the clock takes** (a quarter of the network's, so weak, on every card), it begins and ends no day: the card's days pause where they were until a refresh with a real header brings the clock back, and then end by it as before. A reader that finds the card's clock later than the newest header it has **SHOULD** have the holder or the owner refresh it;
  - **with at least half the work of the hardest header the card has taken**, it is a clock like any other: a day over by it begins at it, and ends only when the clock passes that time by a day, or never, where a day past it does not fit four bytes. The owner's refresh that asks it brings the clock back, and the day's limit's window and the day of loads begun after that header begin again at it, keeping what they counted. A card with no owner has no such way, and its money above what that day has left stays on it.

  Change is counted when the payment that makes it is signed, so change owed always loads ([Loading proofs](#load_proof)); an empty card's reset or hand-over with a real header as the anchor brings any clock back and begins the day of loads again, and the first set-up's anchor brings any clock back and begins again the day of loads begun after it; nothing else moves the day of loads but the owner's refresh that asks it.

- **PIN before money** (55 bit 0): a card takes and pays nothing until it has a PIN, and every load needs the PIN given in the tap.
- **An owner claimed slowly** (55 bit 2): an owner key set on an empty card that has none takes a few seconds of the card being held (four askings, about 3 seconds on the J3R180, 0.59 to 0.66 s an asking), under its PIN, never in a tap that pays.
- **Two units on one card** (55 bit 4): proofs of `sat` and `usd` at one mint on one card of format `05`, kept apart by a keyset table on the card, and limits counted across both at a rate.
- **A holder with no phone** (55 bit 5): a card with a PIN and no owner, its numbers and record set once, in its first set-up and in this order: the limits (in the unit the record will name), the record, a block header; the wallet then reads them back before the first load. Its history is read by its PIN after a hold (five askings, about 4 seconds on the J3R180), and it is reset by its PIN after a hold of about 15 seconds (twenty askings, after the history's hold).
- **Keyset keys** (55 bit 6): the card learns a keyset's keys, checks them against the keyset ID, and writes a change proof only under a key it has shown to be one of them.
- **Paid list** (55 bit 7): the card keeps the proofs it has signed, so that a proof it signed is not loaded and signed again, and loads at most a set number of proofs a day, a payment's change counted when the payment is signed.
- **Certificate** (56 bit 0): an issuer's certificate for the card key, written once and read by anyone ([profiles/certificate.md](profiles/certificate.md)).
- **Date keys for a loader** (`1D`, with keys per load, 56 bit 1): the keys of one date, given to a reader with the PIN given in the tap or the owner's grant.
- **Inbox** (`1E`): where money waits for the card, `SHA-256("CashuCard/inbox" || c || W)` and `W`, `W` the card's clock's time divided by 604800 seconds, rounded down, as four bytes, big-endian, in the hash as in the answer ([test vectors](tests/CCP-tests.md#inbox)); given to any reader with no PIN on a card that is not locked, so that a reader can find what was sent to the card before the holder gives it the PIN. A card that does not keep it answers `6D 00`.
  - **Which weeks.** The card answers the week of its own clock, and anyone holding the card can move the clock forward with real block headers, so a holder can read the name of any week from the card's last header up to now; never a week ahead of real time, but for one limit: a header made for a time far ahead moves the clock there (a reset or a hand-over with no anchor keeps it), and the card then answers that far week and readers find nothing under its names, until the clock is brought back ([Profiles](#profiles), **Clock and limits**). No header a reader gives alone moves the clock back. A reader that looks for waiting money reads `INBOX` for every week from the later of the card's own week and the week two weeks before now, up to this week, giving the card a real header of each week in order and reading `INBOX` after each, so that money filed in the week before, or at a week's edge, is found; and last it gives the card its newest header. Two weeks cover the 14-day hold with pickup closing at day 7.
  - **What waits.** The name is only an address: ecash that waits under it **MUST** be dated (never undated, never the date `0`), locked to a key only the card signs for, the date key `P_card + t·G` with the card's date refund key `R + u·G`, as the owner works them out from `c` (never `P_card` alone or a key the keeper of the name or a reader holds), of a date `D` 14 days after it is filed and never less (no later date, and never moved at pickup), and its secret **MUST** be the card's exact proof secret ([Card proofs](#card-proofs): its serialization, its tags and their order, `SIG_ALL`, as the card rebuilds it). A filing's `D` is its own, 14 days from the filing: never the date the owner's wallet shares among the week's top-ups ([Keys per load](#keys-per-load)). So knowing the name never lets anyone collect, and money the card never collects comes back to the owner. Only the card's owner's phone files under the name, under each week the pickup window touches (two at most); a reader never files.
  - **Which cards.** The card answers `INBOX` whatever its record holds: with a refund key or none, with an owner or none. A phone **MUST NOT** file for a card with no owner (no one keeps `c` and `r` for it) or with no refund key (`GET_CARD` bytes 7 to 39 zeros: such a card takes no dated proof, `6A 8E`, and an undated proof filed for it would be lost if the card were lost, reset or handed over first). It reads both before it files: the refund key from `GET_CARD`, and the owner from the profile that keeps one (the reference implementation's `GET_INFO` byte 16). The holder of such a card loads in person.
  - **At pickup.** The ecash goes onto the card only by a load with the PIN, as any load. Before it gives `LOAD_PROOF`'s P2 bit 1 for a waiting proof, a reader **MUST**, for each proof:
    - rebuild the proof's secret from the card's keys for its date (**Date keys for a loader**, under the PIN given in the tap) and its nonce, and check that it equals the filed secret byte for byte;
    - check that the proof's keyset is a keyset of the mint `GET_CARD` names, in the card's unit, as that mint publishes its keysets ([NUT-02][02]), and that its [NUT-12][12] DLEQ proof verifies under that mint's key for the keyset and the amount: never under a key the filed token, the keeper or another mint names;
    - check that at least 7 days are left before `D`;
    - where it is online, ask that mint the proof's state ([NUT-07][07]) and refuse a proof the mint does not answer `UNSPENT` for;
    - run the checks of any load ([Loading proofs](#load_proof)): no more proofs than the card's empty slots less up to 8 left free for its change, and than it will still load today; no proof above its daily limit; and room in its keyset table for the keyset, where it keeps one.

    It refuses the proof otherwise: it loads nothing of it, and the proof stays where it waits, for a later pickup or until it comes back to the owner after `D`.

  - **What the owner is told.** At the mint a proof on the card and one still waiting both read `UNSPENT`. A reader that loads waiting proofs **SHOULD** leave the owner a pickup proof where it found them, at a random moment 20 seconds to 5 minutes after the tap: `k` (1), the nonces it loaded (32 bytes each, in the order loaded), `rand16` (16 random bytes of its own) and the card's `AUTH` answer (80) to `m = SHA-256("CashuCard/pickup" ‖ nonces ‖ rand16)[0..15]` in that tap. The owner works out `m` and checks the answer's signature under `P_card` ([AUTH](#auth); a vector is in the [tests](tests/CCP-tests.md#pickup-proof)). It shows only that the card was in the reader's field at a tap after those nonces were chosen: `AUTH` answers any reader, with no authorisation, over any 16 bytes, so a pickup proof never shows that anything was loaded. It tells the owner's phone where to look; what is on the card, the card says (its proofs' nonces).
  - **After pickup, and after `D`.** A picked-up proof reaches the card with 7 to 14 days left before `D`, and the card does not read dates. The owner's phone moves picked-up proofs to a new date at its next tap with the card, ending the move before `D` less a day (Before the date, [Keys per load](#keys-per-load)); nothing gives them a new date at pickup. Every reader, online or not, leaves out of a payment a dated proof whose `D` is less than a day after its own time ([Paying](#paying)), so from a day before `D` the card pays no proof of that date. A day after `D` (`D` + 86,400, by the network's tip its wallet has checked) the owner **MAY** take back by the refund key every proof of that date, picked up or not, once its phone has read the card at a tap after that (and shown its holder which were on it), or at once where the card is lost: a day on each side covers a reader's clock slow and the owner's fast. A proof taken back while it is on the card is one the card still shows and no reader pays; the owner's phone frees its slot at its next tap, by a payment of it that it does not take to the mint.
  - **What is lost.** Nothing that waits: a card lost, reset or handed over before pickup, or a proof no reader picks up, leaves the proofs locked to a date key no one else can sign with before `D`; after `D` they come back to the refund key, which the owner spends with `r` and the `c` they were filed under (so it keeps that `c` with `r` until every proof filed under it is picked up or taken back).
  - **What it tells.** Whoever keeps what waits under a name learns that the card was sent something that week, and that the name is the card's, tied to `P_card`, which any reader reads.
- **Assisted settlement** (`1B`): the record of the card's last payment, for a relay to finish its swap, given to the paying tap and otherwise to the holder.
- **NFC tag**: an NFC Forum Type 4 tag carrying a web address.

The certificate profile is specified in this repository. Until each of the others is written as its own document, they are specified in the reference implementation's [command reference](https://github.com/cashucard/cashucard/blob/main/spec/APDU.md).

## Mint support

A mint that supports [NUT-10][10], [NUT-11][11] with `SIG_ALL` on swaps, and [NUT-12][12] supports cards. It **SHOULD** advertise NUT-11 and NUT-12 in its info ([NUT-06][06]), since readers check DLEQ proofs offline.

## Security considerations

- **The card signs only what it spends.** Every payment signature is released in the same atomic write that marks its inputs spent. Every other signature is over a tagged message no proof's secret can equal ([Paying](#paying)).
- **The date key locks the card's change outputs to the card, and only the card can finish them.** The blinding factor of an owed opening never leaves the card, and the card writes a change proof only when the mint's DLEQ proof holds under the key bound at the payment.
- **A reader can direct the payment's other outputs.** The holder trusts the reader the card is presented to for the amount; card policy bounds what one reader can take.
- **Offline acceptance carries double-spend risk** for the reader ([Accepting a payment offline](#accepting-a-payment-offline)), and a dated proof past its locktime can be taken back by the refund key.
- **A lost card is lost money**, within whatever its card policy allows to a finder.
- **What a mint links.** The proofs of one date share their key, so a mint can tie the spends of one date's proofs to each other: a week's top-ups, or every undated proof of the card. It sees no key that ties one date to another, or a date to the card.
- **What someone holding the card learns**, with no PIN and no grant, and so, with the mint's records, what they can tie to the card: the date keys of every date with a proof on the card (`SPEND_ALL_BEGIN`), and so every spend of those dates, spent proofs of the same week among them; the swap that made each proof still on the card (`GET_DLEQ`'s `r`); and, on a card that keeps them, which of the card's recent payments were made with no PIN (`SPEND_ALL_AGAIN` P1 `01`), the inbox name of each week from the card's last header up to now, so whether money waited for the card in those weeks, and, on a card that bounds its daily loads, how many are left today where fewer than 8 are (`SPEND_ALL_CHANGE`'s `6A 8F`, asked before any authorisation, the payment then given up with nothing owed). This is the limit of the card's privacy against someone holding it. A reader given the PIN learns every date's keys. A listener on the air learns what the reader it listens to is told; `c` is never sent in the clear.
- **Ecash locked to keys the card does not sign with.** Ecash locked to `P_card` is never spendable by the card; ecash locked to a card's date keys and not loaded before a reset or an owner replaced is under the old `c`, which the card no longer has: dated, it comes back to the refund key after its date; undated, it is lost. A wallet loads or refunds what it swapped before the card is handed on, and checks the card's keys before each top-up.
- **A held reset.** A card a profile lets its holder reset by the PIN alone (a card with no owner, held for about 15 seconds on the J3R180: the history's seven askings, then the reset's seventeen, 24 askings in all; empty) can be reset and set up again by anyone who has the PIN and holds the card that long, with numbers, a refund key or an owner of its own. No till holds a card that long. The holder's wallet sees it before the next top-up: the card answers other keys for the date.
- **A lost record.** Where a card cannot keep a payment's receipt within its atomic write, the payment stands (its proofs spent, the signature given), and that payment has no receipt and no payment record for a relay; the reader that took it holds the signature.

## Test vectors

See [tests/CCP-tests.md][tests].

[00]: https://github.com/cashubtc/nuts/blob/main/00.md
[02]: https://github.com/cashubtc/nuts/blob/main/02.md
[03]: https://github.com/cashubtc/nuts/blob/main/03.md
[06]: https://github.com/cashubtc/nuts/blob/main/06.md
[07]: https://github.com/cashubtc/nuts/blob/main/07.md
[09]: https://github.com/cashubtc/nuts/blob/main/09.md
[10]: https://github.com/cashubtc/nuts/blob/main/10.md
[11]: https://github.com/cashubtc/nuts/blob/main/11.md
[12]: https://github.com/cashubtc/nuts/blob/main/12.md
[tests]: tests/CCP-tests.md
