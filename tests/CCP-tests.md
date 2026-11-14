# NUT-CCP Test Vectors

All values are hex unless noted. Each vector can be checked with BIP-340, SHA-256, the `hash_to_curve` of [NUT-00](https://github.com/cashubtc/nuts/blob/main/00.md) and the DLEQ proof of [NUT-12](https://github.com/cashubtc/nuts/blob/main/12.md). Every vector comes from the reference applet, protocol `02 03`.

## Keys per load

The card's key, its refund key and its secret `c`. The refund key is `G`, so its private key `r` is 1. For each date `D` (four bytes, big-endian), `t = SHA-256("CashuCard/key" || c || D) mod n`, the date key is `P_card + t·G`, `u = SHA-256("CashuCard/refund-key" || c || D) mod n` and the date refund key is `R + u·G`. The date `0` names no refund key. These dates exercise the arithmetic and the text; a wallet loads under a date in the future ([NUT-CCP](../CCP.md#keys-per-load)).

```json
{
  "card_key": "02fa05ed0a3649886270bacec5ddc1a3643009a759bee996f9a6ac060c9992a82e",
  "refund_key": "0279be667ef9dcbbac55a06295ce870b07029bfcdb2dce28d959f2815b16f81798",
  "c": "000102030405060708090a0b0c0d0e0f101112131415161718191a1b1c1d1e1f"
}
```

```json
[
  {
    "date": 0,
    "t": "ee4a815ff661fb07d56146fa40bfac6cfdf9b8dcc5897c84d74b34565bf75e11",
    "date_key": "03fe244efa804b025d7ade581226ab35511bde8c606c2c3cfa43b674ddeb9f5724",
    "date_refund_key": null
  },
  {
    "date": 1,
    "t": "73fc4ce1a67980f0661c641be376d54e34fff60fb82cba06eb780a6ae6549f59",
    "date_key": "03fdc363fa2178baca97fdd15c50134aed36ddfd45d8b4cdfeac0a154b9ab55d8f",
    "date_refund_key": "02e10682fa22760120af2ffbeb2994be6f015f0d8b7bcce61840ea34210bdd93a5"
  },
  {
    "date": 1700000000,
    "t": "0256d5e47cfeef95af4f9656afffb0ba7bf47dc2563a28e187fe055d44939b2e",
    "date_key": "0292255633842ae726912bf9cd8120e8ecc4e60129baa1c6e3175912a085b32308",
    "date_refund_key": "02390098aacd9d6a5a547b05d1cc3c5fb9b15a200dcf3f28e36a25a773bd2bd12d"
  },
  {
    "date": 1900000000,
    "t": "ef2d23cb2b02d35f60567671aa522e3cd6ea1fcbb6003b7d9602aaed0ec6a4a7",
    "date_key": "02a4fbd7024e706231d0af069405947ad2f207b82432aea319564de1e4805486f0",
    "date_refund_key": "03a8e8c6bb7f431722dd6700fbf1e6e9f69021177dfcb1f4e1524a2eb6ac3ab47d"
  },
  {
    "date": 4294967295,
    "t": "ffa0deae5468ad4faf6e5926333b0029c16c8060bd9717c5a997db95ca289c6b",
    "date_key": "02092f84168d21735fe3be9d3184544c729b82336be5d3f9057e4f6799906bc230",
    "date_refund_key": "03553c3b2b10c394476fec39d8330508a92e07c81da767cdb77bf1fca9e11d0522"
  }
]
```

## Card proof secrets

A card rebuilds each secret from its stored slot and the keys of the slot's date.

Slot (82 bytes):

```
010059534ce0bfa19a00000008404142434445464748494a4b4c4d4e4f505152535455565758595a5b5c5d5e5f02414141414141414141414141414141414141414141414141414141414141414400000000
```

Secret:

```
["P2PK",{"nonce":"404142434445464748494a4b4c4d4e4f505152535455565758595a5b5c5d5e5f","data":"03fe244efa804b025d7ade581226ab35511bde8c606c2c3cfa43b674ddeb9f5724","tags":[["sigflag","SIG_ALL"]]}]
```

Slot (82 bytes):

```
010059534ce0bfa19a000000104748494a4b4c4d4e4f505152535455565758595a5b5c5d5e5f6061626364656602484848484848484848484848484848484848484848484848484848484848484a00000001
```

Secret:

```
["P2PK",{"nonce":"4748494a4b4c4d4e4f505152535455565758595a5b5c5d5e5f60616263646566","data":"03fdc363fa2178baca97fdd15c50134aed36ddfd45d8b4cdfeac0a154b9ab55d8f","tags":[["locktime","1"],["refund","02e10682fa22760120af2ffbeb2994be6f015f0d8b7bcce61840ea34210bdd93a5"],["sigflag","SIG_ALL"]]}]
```

Slot (82 bytes):

```
010059534ce0bfa19a000000204e4f505152535455565758595a5b5c5d5e5f606162636465666768696a6b6c6d024f4f4f4f4f4f4f4f4f4f4f4f4f4f4f4f4f4f4f4f4f4f4f4f4f4f4f4f4f4f4f4f6553f100
```

Secret:

```
["P2PK",{"nonce":"4e4f505152535455565758595a5b5c5d5e5f606162636465666768696a6b6c6d","data":"0292255633842ae726912bf9cd8120e8ecc4e60129baa1c6e3175912a085b32308","tags":[["locktime","1700000000"],["refund","02390098aacd9d6a5a547b05d1cc3c5fb9b15a200dcf3f28e36a25a773bd2bd12d"],["sigflag","SIG_ALL"]]}]
```

Slot (82 bytes):

```
010059534ce0bfa19a0000004055565758595a5b5c5d5e5f606162636465666768696a6b6c6d6e6f7071727374025656565656565656565656565656565656565656565656565656565656565656713fb300
```

Secret:

```
["P2PK",{"nonce":"55565758595a5b5c5d5e5f606162636465666768696a6b6c6d6e6f7071727374","data":"02a4fbd7024e706231d0af069405947ad2f207b82432aea319564de1e4805486f0","tags":[["locktime","1900000000"],["refund","03a8e8c6bb7f431722dd6700fbf1e6e9f69021177dfcb1f4e1524a2eb6ac3ab47d"],["sigflag","SIG_ALL"]]}]
```

Slot (82 bytes):

```
010059534ce0bfa19a000000805c5d5e5f606162636465666768696a6b6c6d6e6f707172737475767778797a7b025d5d5d5d5d5d5d5d5d5d5d5d5d5d5d5d5d5d5d5d5d5d5d5d5d5d5d5d5d5d5d5dffffffff
```

Secret:

```
["P2PK",{"nonce":"5c5d5e5f606162636465666768696a6b6c6d6e6f707172737475767778797a7b","data":"02092f84168d21735fe3be9d3184544c729b82336be5d3f9057e4f6799906bc230","tags":[["locktime","4294967295"],["refund","03553c3b2b10c394476fec39d8330508a92e07c81da767cdb77bf1fca9e11d0522"],["sigflag","SIG_ALL"]]}]
```

## A payment of one proof

For each slot above, the card signed a payment of that one proof into no outputs. `SPEND_ALL_BEGIN` with P1 `02` names the slot and answers the amount, `t` and the date refund key (zeros for the date `0`). The message is `secret || C`, so its hash is `SHA-256(secret || C)`. The signature is BIP-340 by the date key, and does not verify under the card key.

```json
{
  "amount": 8,
  "date": 0,
  "spend_all_begin": "b0220200010045",
  "response": "00000008ee4a815ff661fb07d56146fa40bfac6cfdf9b8dcc5897c84d74b34565bf75e11000000000000000000000000000000000000000000000000000000000000000000",
  "msg_hash": "0d989eb3852862f5b4080e332ef4e9f331a8dc7bde24da84d6417354400f9e30",
  "signature": "33f2e875f9baf671beb937aa90e1728b764498b74e31a7b4a651d5eda37c96759fc2b51be58beca48fb173b877c2b29d967c848a41cb63319d57536d7ab2e4ea"
}
```

```json
{
  "amount": 16,
  "date": 1,
  "spend_all_begin": "b0220200010145",
  "response": "0000001073fc4ce1a67980f0661c641be376d54e34fff60fb82cba06eb780a6ae6549f5902e10682fa22760120af2ffbeb2994be6f015f0d8b7bcce61840ea34210bdd93a5",
  "msg_hash": "6941e17d736e200ce4e8a54e9b032f56151e97b0c0f5b1e80c1b9f5c2574b88f",
  "signature": "4db863c3f263f8e00ce4f4503d8001281458d404a15d430fb863f834ce4edf007adda6d706367e9ba678a91677bc6537e9fc179e3145153ababa5ec01323cce0"
}
```

```json
{
  "amount": 32,
  "date": 1700000000,
  "spend_all_begin": "b0220200010245",
  "response": "000000200256d5e47cfeef95af4f9656afffb0ba7bf47dc2563a28e187fe055d44939b2e02390098aacd9d6a5a547b05d1cc3c5fb9b15a200dcf3f28e36a25a773bd2bd12d",
  "msg_hash": "8ff8426457c9b7349a21d3798d0f27b63c10c245cbd3043c14f7e647d5545943",
  "signature": "5461343362a250ac91b7ea5a351a123aac5cf885a3655298c687e6b8206c71c0b480939934493215250ba28351bc7b66ebe4e39612c1ecfb94af9bebceade1bc"
}
```

```json
{
  "amount": 64,
  "date": 1900000000,
  "spend_all_begin": "b0220200010345",
  "response": "00000040ef2d23cb2b02d35f60567671aa522e3cd6ea1fcbb6003b7d9602aaed0ec6a4a703a8e8c6bb7f431722dd6700fbf1e6e9f69021177dfcb1f4e1524a2eb6ac3ab47d",
  "msg_hash": "46d307ae9098f0a249908133fc77d2d3c429390651932596f06603f5ec636bd6",
  "signature": "9af2b8a39dff9d4dd5514c320b47cb637bb3958fc24b06dea6b4933003491349cc3f08a2b56d195bd029079a043128e0c3d61cbca520abf20cf3aa06f38e041b"
}
```

```json
{
  "amount": 128,
  "date": 4294967295,
  "spend_all_begin": "b0220200010445",
  "response": "00000080ffa0deae5468ad4faf6e5926333b0029c16c8060bd9717c5a997db95ca289c6b03553c3b2b10c394476fec39d8330508a92e07c81da767cdb77bf1fca9e11d0522",
  "msg_hash": "276a16c6b53292d74cc94cb2c627fe337f846715b5e0ce16eae90336b361caa3",
  "signature": "0dd48c48f37dbf8c5836f84fcd8e789d65a2a818c1d26e81cd1883cd1385bbd9328298e48534791fe2469e36ff63cb1c8063406a44c677f7d0b64d4cf4cc6fef"
}
```

## AUTH

`T = SHA-256("CashuCard/auth")`, and the card signs `SHA-256(T || T || reader_nonce || card_nonce || card_key)` with the card key itself.

```json
{
  "card_key": "02fa05ed0a3649886270bacec5ddc1a3643009a759bee996f9a6ac060c9992a82e",
  "reader_nonce": "a0a1a2a3a4a5a6a7a8a9aaabacadaeaf",
  "card_nonce": "34a0b46537817d1c53a9824a8ed4b7b9",
  "msg_hash": "b0e4d5f760a991bef3d5c04a4a048880c3f355072497b92df0eedea773cc1534",
  "signature": "caaf4138ff6980bf7429f35efd4f2d9ebae5b933c6af219aaab25380a2614c8fcfd6b6bb76cbfa74228b7acba79a2e7aa53813df5617794d02f2d163c10c745d"
}
```

## The card secret, sealed

`GET_NONCE` P1 `01` gives 16 fresh bytes, the PIN key and the card key's signature over `SHA-256("CashuCard/pinkey" || PIN key)`; `GET_CARD` P1 `03` then answers `c` sealed, with the block

```
S = SHA-256( "CashuCard/seal" || 01 || x || K || nonce || 16 )
```

where `nonce` is the 16 bytes and `16` the instruction byte. Both cards hold the `c` above.

A card with the refund key `7·G`: no data; the answer is `K = k·G` (65) and `c XOR S` with `x` the x of `r·K = k·R`.

```json
{
  "refund_private_key": "0000000000000000000000000000000000000000000000000000000000000007",
  "refund_key": "025cbdf0646e5db4eaa398f365f2ea7a0e3d419b7e0330e39ce92bddedcac4f9bc",
  "get_nonce": "b044010000",
  "nonce": "f9c06213585289654996f0c40467b9a6",
  "get_card": "b016030000",
  "response": "042bbbd093fa488f1fb511ba7a2c095755b2187c3501943fc29ad48140fa807ba8f38569aa7f31b73e33ec87489538f0a0ce9309564d06945ac0d75683e2bc743aefaaefaac0213fc24083d2d29734a1bf88d669f6ea1fcb5573d77c00baaed548"
}
```

A card with no refund key, in its first set-up: the reader sends its key for this one message, `E = e·G`, and the answer is `c XOR S` with `x` the x of `e·(PIN key)` and `E` in `K`'s place. The card key of this card is the `card_key` of [Keys per load](#keys-per-load), which signs the PIN key.

```json
{
  "get_nonce_response": "f9c06213585289654996f0c40467b9a602cf9bfbdca0c087fdea7097d5482d3a7b30b5fa1cb52b621f89c0a61619b782d47fb4580ccd7149a9ea7a362e270cd4c7a466c77b01ecc952633a8071c9962f50d8e17bd0dd9883e9630ddf4dbc14e026bada6202833b06e17b866151019fd348",
  "pin_key": "02cf9bfbdca0c087fdea7097d5482d3a7b30b5fa1cb52b621f89c0a61619b782d4",
  "e_private_key": "1111111111111111111111111111111111111111111111111111111111111111",
  "E": "044f355bdcb7cc0af728ef3cceb9615d90684bb5b2ca5f859ab0f0b704075871aa385b6b1b8ead809ca67454d9683fcf2ba03456d6fe2c4abe2b07f0fbdbb2f1c1",
  "get_card": "b016030041044f355bdcb7cc0af728ef3cceb9615d90684bb5b2ca5f859ab0f0b704075871aa385b6b1b8ead809ca67454d9683fcf2ba03456d6fe2c4abe2b07f0fbdbb2f1c100",
  "response": "59a5e31483930a6430e1fca56d76474b1efa64ec2f17cbd12e53e0ba4a028aa3"
}
```

## Inbox

The inbox profile's name for a week ([NUT-CCP](../CCP.md#profiles), **Inbox**): `SHA-256("CashuCard/inbox" || c || W)`, the label in ASCII, `c` of [Keys per load](#keys-per-load), and `W` the card's clock's time divided by 604800 seconds, rounded down, as four bytes, big-endian. The card answers `INBOX` (`b01e000024`) with the name, then `W`. Two weeks of the card's clock, the second from its first second:

```json
{
  "c": "000102030405060708090a0b0c0d0e0f101112131415161718191a1b1c1d1e1f",
  "command": "b01e000024",
  "weeks": [
    {
      "time": 1800000000,
      "week": 2976,
      "w": "00000ba0",
      "hashed": "4361736875436172642f696e626f78000102030405060708090a0b0c0d0e0f101112131415161718191a1b1c1d1e1f00000ba0",
      "name": "e73098b5ebee7a625ecc2251b747ddef155325488896bb756375f0c89c8c09ae",
      "response": "e73098b5ebee7a625ecc2251b747ddef155325488896bb756375f0c89c8c09ae00000ba09000"
    },
    {
      "time": 1800489600,
      "week": 2977,
      "w": "00000ba1",
      "hashed": "4361736875436172642f696e626f78000102030405060708090a0b0c0d0e0f101112131415161718191a1b1c1d1e1f00000ba1",
      "name": "1591e7afe066f29a8346b038d379c80cd7337ed022fbf6abc42759b57f30944d",
      "response": "1591e7afe066f29a8346b038d379c80cd7337ed022fbf6abc42759b57f30944d00000ba19000"
    }
  ]
}
```

## Pickup proof

The pickup proof a reader leaves the owner after it loads waiting proofs ([NUT-CCP](../CCP.md#profiles), **Inbox**, What the owner is told): `m = SHA-256("CashuCard/pickup" || n1 || ... || nk || rand16)[0..15]`, the label in ASCII, the loaded proofs' nonces (32 bytes each) in the order loaded, and 16 random bytes of the reader's; the card's `AUTH` answer to `m` (its 16 bytes, then a BIP-340 signature by `P_card` over `SHA-256(T || T || m || its 16 || P_card)`, `T = SHA-256("CashuCard/auth")`); and the proof, `k (1) || nonces || rand16 || answer`. Two nonces, the card's own answer; the card's 16 bytes and the signature are fresh at every `AUTH`, so a reader checks the signature rather than these bytes:

```json
{
  "cardKey": "02fa05ed0a3649886270bacec5ddc1a3643009a759bee996f9a6ac060c9992a82e",
  "nonces": [
    "3888b09589f797e4be5b84a3022cc4abf6e5c748aba5d1abeb4fe0906736a182",
    "c28a1681cd90d65fc2662fcc03811cb57c7454f5d4631baded50cffe3420f474"
  ],
  "rand16": "a0a1a2a3a4a5a6a7a8a9aaabacadaeaf",
  "hashed": "4361736875436172642f7069636b75703888b09589f797e4be5b84a3022cc4abf6e5c748aba5d1abeb4fe0906736a182c28a1681cd90d65fc2662fcc03811cb57c7454f5d4631baded50cffe3420f474a0a1a2a3a4a5a6a7a8a9aaabacadaeaf",
  "m": "cca276575b15928f1bc28200f851166d",
  "auth": "b015000010cca276575b15928f1bc28200f851166d50",
  "answer": "9adf8de1fc1b1b5d5f2576b99a79130a63a06a6165c5f286d119cedefc068718c371f2902fac3fdccc1c1faea6df97ef8448b4b84e02f82f20cf3f55c423f49575f92fbbf4ae9b1061799a121bedcedb",
  "proof": "023888b09589f797e4be5b84a3022cc4abf6e5c748aba5d1abeb4fe0906736a182c28a1681cd90d65fc2662fcc03811cb57c7454f5d4631baded50cffe3420f474a0a1a2a3a4a5a6a7a8a9aaabacadaeaf9adf8de1fc1b1b5d5f2576b99a79130a63a06a6165c5f286d119cedefc068718c371f2902fac3fdccc1c1faea6df97ef8448b4b84e02f82f20cf3f55c423f49575f92fbbf4ae9b1061799a121bedcedb"
}
```

## The owner's c

The owner's `c` given to the card at a claim ([NUT-CCP](../CCP.md#profiles), **Owner**): BIP-39's test words with no passphrase; `seed`, their 64-byte BIP-39 seed; `s = HMAC-SHA256(seed, "CashuCard/c" || 0x00 || cardKey)`; the card's PIN key and the card key's signature over `SHA-256("CashuCard/pinkey" || pinKey)` (fresh at every card: a reader checks it rather than the bytes); `e`, the reader's scalar for this one message, `E = e·G` and `x`, the x of `e·pinKey`; the envelope, `E || (s XOR B1) || B0[0..15]` with `Bi = SHA-256("CashuCard/c" || i || x || E [|| the 32 sealed bytes, for B0])`, `i` one byte and `B0[0..15]` the tag, and the reference card's command that carries it (`stage`); the check at the end of the claim's data, `SHA-256("CashuCard/c" || s)[0..15]`; `c = SHA-256("CashuCard/c" || s || n)` for the card's first owner's `c`, `n = 1`; and, under that `c`, the card's own answer for the date `1900000000` with the refund key `G` (`r = 1`): `t`, `P_card + t·G` and `R + u·G`, which `r + u` spends after the date:

```json
{
  "label": "CashuCard/c",
  "words": "abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon about",
  "seed": "5eb00bbddcf069084889a8ab9155568165f5c453ccb85e70811aaed6f6da5fc19a5ac40b389cd370d086206dec8aa6c43daea6690f20ad3d8d48b2d2ce9e38e4",
  "cardKey": "02fa05ed0a3649886270bacec5ddc1a3643009a759bee996f9a6ac060c9992a82e",
  "pinKey": "02cf9bfbdca0c087fdea7097d5482d3a7b30b5fa1cb52b621f89c0a61619b782d4",
  "pinKeySig": "7fb4580ccd7149a9ea7a362e270cd4c7a466c77b01ecc952633a8071c9962f50d8e17bd0dd9883e9630ddf4dbc14e026bada6202833b06e17b866151019fd348",
  "s": "a36e3b67fce644fa43254871d2739aec25cd466696651b86c96959d1331f09e2",
  "e": "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90",
  "E": "04555a14d855d2bb43117e50a9783bdc573c23a3e6999029ddae7347490dbed3b2268f9a17a047458e7703b48db713e713993a40e5e3319d10872514672916f692",
  "x": "caa7718811fa7fa0e726d07033b719e30645f8202e1b16fdfb0ee6bbdc15ca40",
  "envelope": "04555a14d855d2bb43117e50a9783bdc573c23a3e6999029ddae7347490dbed3b2268f9a17a047458e7703b48db713e713993a40e5e3319d10872514672916f692dde9c131154a3287bc970564d6e79a75532e911e9c6ea9e31b3aa8e411ec7badc4d68c61be393be59fff30be4f6633b8",
  "stage": "b04301007104555a14d855d2bb43117e50a9783bdc573c23a3e6999029ddae7347490dbed3b2268f9a17a047458e7703b48db713e713993a40e5e3319d10872514672916f692dde9c131154a3287bc970564d6e79a75532e911e9c6ea9e31b3aa8e411ec7badc4d68c61be393be59fff30be4f6633b8",
  "check": "6dc67a743528c1e31b74ced659a7cf24",
  "n": 1,
  "c": "1513f5befd1fd287d28a80ac372234bb893107527598ae5bc20f8c7ff2bfff23",
  "refundKey": "0279be667ef9dcbbac55a06295ce870b07029bfcdb2dce28d959f2815b16f81798",
  "date": 1900000000,
  "groupKeys": "a80dfc16045e28ee2e42852377264793f8658de697e2561fd802cca4b24477e702d75bc353575a82d42da35c485333b24fb7ab6b998525a9e4dd9bf51fa94185bf031a3903b17a0d2da21251c369b904481fb42894d0e75d3f5067f64c0845a20e0d"
}
```

## A payment of two proofs with change

A card holding proofs of 8 and 32 pays 32 to a reader and makes 8 of change for itself. The card's private key, its secret `c` and the mint's keys are given so every value can be rebuilt. The keyset has no fees. On a card that sets `GET_INFO` P1 `04` byte 55 bit 6 (keyset keys), the reader has taught it the change keyset first (`KEYSET_KEYS` P1 `00` to `02`, under the PIN given in the tap), and shows the bound key (`KEYSET_KEYS` P1 `03`) before `LOAD_CHANGE` ([Change](../CCP.md#change)); those commands are a profile's, and are not recorded here. Without them the reference implementation answers `SPEND_ALL_CHANGE` and `LOAD_CHANGE` `6A 9F`. Both proofs are undated, so the payment is of the date `0`: `t` is the one of [Keys per load](#keys-per-load), and the date key is `P_card + t·G`.

```json
{
  "card_private_key": "3bee8a332dc85965b145c733fd94a532418d23499dd93290c449ef93b64202d7",
  "card_key": "0248f87c8ddddced1b7870d0461b6e2e68608f101828c8ebb7bdc3bbeed9b6f287",
  "c": "000102030405060708090a0b0c0d0e0f101112131415161718191a1b1c1d1e1f",
  "t": "ee4a815ff661fb07d56146fa40bfac6cfdf9b8dcc5897c84d74b34565bf75e11",
  "date_key": "037e16b938f79cccf6f3b0b3168291b23fb5f354cc78087374972dc5c056a27a90",
  "mint_keys": {
    "8": "037ae23922d0a6bc02f80cd5768b5b2d529e64c70add504aec9b498529bcfac095",
    "32": "035697cd55bfe86f9100d85b2703228f77e03652a175c967ab2064f7828b78fbe3"
  },
  "keyset": "00f1e4f5c2b1a090"
}
```

The inputs, in slots 0 and 1:

```json
[
  {
    "amount": 8,
    "nonce": "d7a1a81efa4268f1b7a43db2501ddeab6f6ba63d1cbd2a035a17b03955bf2af2",
    "secret": "[\"P2PK\",{\"nonce\":\"d7a1a81efa4268f1b7a43db2501ddeab6f6ba63d1cbd2a035a17b03955bf2af2\",\"data\":\"037e16b938f79cccf6f3b0b3168291b23fb5f354cc78087374972dc5c056a27a90\",\"tags\":[[\"sigflag\",\"SIG_ALL\"]]}]",
    "C": "03f0842d7b4da44cb02c71944a521481cd56717bf719fcbeaaa2400950c31f45d9"
  },
  {
    "amount": 32,
    "nonce": "61f41166d013508a53e670ef4ba10fbe4dceb9d58d8c5909f30daf2c79ddb613",
    "secret": "[\"P2PK\",{\"nonce\":\"61f41166d013508a53e670ef4ba10fbe4dceb9d58d8c5909f30daf2c79ddb613\",\"data\":\"037e16b938f79cccf6f3b0b3168291b23fb5f354cc78087374972dc5c056a27a90\",\"tags\":[[\"sigflag\",\"SIG_ALL\"]]}]",
    "C": "025e1c7a2e84e898cde358b34a77ff2ddddd2edb4d6d78a5f6cb77d915c5a6b17a"
  }
]
```

The commands and the card's responses (data only; each ends `90 00`):

```json
[
  {
    "command": "SPEND_ALL_BEGIN",
    "apdu": "b022020002000145",
    "response": "00000028ee4a815ff661fb07d56146fa40bfac6cfdf9b8dcc5897c84d74b34565bf75e11000000000000000000000000000000000000000000000000000000000000000000"
  },
  {
    "command": "SPEND_ALL_OUTPUTS",
    "apdu": "b023000025000000200211bccaf554963e836ce04494fec4e539d8ddbf3bac412c761f8e7849adf26f2c",
    "response": ""
  },
  {
    "command": "SPEND_ALL_CHANGE",
    "apdu": "b02600002d00000008037ae23922d0a6bc02f80cd5768b5b2d529e64c70add504aec9b498529bcfac09500f1e4f5c2b1a09021",
    "response": "0214d74d45b98f556be3ecb84cb7b0570f0fa6815dca1d8251c43d7ae55b003758"
  },
  {
    "command": "SPEND_ALL_SIGN",
    "apdu": "b024000040",
    "response": "b7dbdaf81c68acb6d3d5547d84cc89b4a30db47fab2fca3c17a6e9ebe77d0212d9305871afb213ddb1fc944fcf1a04a899e40a820b2e202233db3b4d0ab925be"
  }
]
```

`SPEND_ALL_BEGIN` (P1 `02`, the word on keys per load) answers the total, 40, then `t` and 33 zero bytes: an undated payment names no refund key. `SPEND_ALL_CHANGE` names the change amount, 8, the mint's key for 8 (`mint_keys["8"]`), which the opening is bound to, and the keyset the change is signed in. Neither the key nor the keyset is part of the signed message. A card of revision `03` answers `67 00` to the forms of earlier revisions, `b0260000040000000821` (the amount alone) and the same 37 bytes with no keyset.

The reader's output is for the secret `9bcb4ac6d7eca4e268ca88200bb93ad4e1c1998583674593fa60ae58be75185d` (UTF-8) blinded with `r = e2fae0673f3205892ef427d3341088a752ef126309e1a1feb2ae3d7abf12934c`. The card's change output is for this secret, under the same date key, blinded with the `r` below:

```json
{
  "secret": "[\"P2PK\",{\"nonce\":\"f60f52083006f305833a08f2d7b9a94614665a29cae239f8af980e36252db426\",\"data\":\"037e16b938f79cccf6f3b0b3168291b23fb5f354cc78087374972dc5c056a27a90\",\"tags\":[[\"sigflag\",\"SIG_ALL\"]]}]",
  "r": "a93cf0318c22652f73fa1e1a79d90386cbce181b614ac6623f035580d0921ecb",
  "B_": "0214d74d45b98f556be3ecb84cb7b0570f0fa6815dca1d8251c43d7ae55b003758"
}
```

The message, as UTF-8 text:

```
["P2PK",{"nonce":"d7a1a81efa4268f1b7a43db2501ddeab6f6ba63d1cbd2a035a17b03955bf2af2","data":"037e16b938f79cccf6f3b0b3168291b23fb5f354cc78087374972dc5c056a27a90","tags":[["sigflag","SIG_ALL"]]}]03f0842d7b4da44cb02c71944a521481cd56717bf719fcbeaaa2400950c31f45d9["P2PK",{"nonce":"61f41166d013508a53e670ef4ba10fbe4dceb9d58d8c5909f30daf2c79ddb613","data":"037e16b938f79cccf6f3b0b3168291b23fb5f354cc78087374972dc5c056a27a90","tags":[["sigflag","SIG_ALL"]]}]025e1c7a2e84e898cde358b34a77ff2ddddd2edb4d6d78a5f6cb77d915c5a6b17a320211bccaf554963e836ce04494fec4e539d8ddbf3bac412c761f8e7849adf26f2c80214d74d45b98f556be3ecb84cb7b0570f0fa6815dca1d8251c43d7ae55b003758
```

```json
{
  "msg_hash": "0507d1ae3d79a68b87c19381fa3ac139a05563b8bd8380e6a407428d68931e7a",
  "signature": "b7dbdaf81c68acb6d3d5547d84cc89b4a30db47fab2fca3c17a6e9ebe77d0212d9305871afb213ddb1fc944fcf1a04a899e40a820b2e202233db3b4d0ab925be"
}
```

The signature verifies under the date key, not under the card key.

The swap the reader sends to the mint ([NUT-03](https://github.com/cashubtc/nuts/blob/main/03.md)):

```json
{
  "inputs": [
    {
      "amount": 8,
      "id": "00f1e4f5c2b1a090",
      "secret": "[\"P2PK\",{\"nonce\":\"d7a1a81efa4268f1b7a43db2501ddeab6f6ba63d1cbd2a035a17b03955bf2af2\",\"data\":\"037e16b938f79cccf6f3b0b3168291b23fb5f354cc78087374972dc5c056a27a90\",\"tags\":[[\"sigflag\",\"SIG_ALL\"]]}]",
      "C": "03f0842d7b4da44cb02c71944a521481cd56717bf719fcbeaaa2400950c31f45d9",
      "witness": "{\"signatures\":[\"b7dbdaf81c68acb6d3d5547d84cc89b4a30db47fab2fca3c17a6e9ebe77d0212d9305871afb213ddb1fc944fcf1a04a899e40a820b2e202233db3b4d0ab925be\"]}"
    },
    {
      "amount": 32,
      "id": "00f1e4f5c2b1a090",
      "secret": "[\"P2PK\",{\"nonce\":\"61f41166d013508a53e670ef4ba10fbe4dceb9d58d8c5909f30daf2c79ddb613\",\"data\":\"037e16b938f79cccf6f3b0b3168291b23fb5f354cc78087374972dc5c056a27a90\",\"tags\":[[\"sigflag\",\"SIG_ALL\"]]}]",
      "C": "025e1c7a2e84e898cde358b34a77ff2ddddd2edb4d6d78a5f6cb77d915c5a6b17a"
    }
  ],
  "outputs": [
    {
      "amount": 32,
      "id": "00f1e4f5c2b1a090",
      "B_": "0211bccaf554963e836ce04494fec4e539d8ddbf3bac412c761f8e7849adf26f2c"
    },
    {
      "amount": 8,
      "id": "00f1e4f5c2b1a090",
      "B_": "0214d74d45b98f556be3ecb84cb7b0570f0fa6815dca1d8251c43d7ae55b003758"
    }
  ]
}
```

## Input fees

The card does not know the mint's fees ([Paying](../CCP.md#paying)). At a keyset with an input fee ([NUT-02](https://github.com/cashubtc/nuts/blob/main/02.md) `input_fee_ppk`), the fee of a swap is `ceil(sum of each input's input_fee_ppk / 1000)`, and the reader's outputs and the card's change together must come to the inputs' total less it. Each case below gives the inputs, the fee, the total the outputs must come to, and one split into a reader's outputs and the card's change (each a power of two, as the mint's keys are). The first case is [A payment of two proofs with change](#a-payment-of-two-proofs-with-change); in the second, that payment's split (32 and 8 of change, 40 in all) is one the card signs and spends and the mint refuses.

```json
[
  {
    "inputs": [
      {
        "amount": 8,
        "input_fee_ppk": 0
      },
      {
        "amount": 32,
        "input_fee_ppk": 0
      }
    ],
    "fee": 0,
    "outputs_total": 40
  },
  {
    "inputs": [
      {
        "amount": 8,
        "input_fee_ppk": 100
      },
      {
        "amount": 32,
        "input_fee_ppk": 100
      }
    ],
    "fee": 1,
    "outputs_total": 39,
    "reader_outputs": [32],
    "change": [4, 2, 1]
  },
  {
    "inputs": [
      {
        "amount": 8,
        "input_fee_ppk": 1000
      },
      {
        "amount": 32,
        "input_fee_ppk": 1000
      }
    ],
    "fee": 2,
    "outputs_total": 38,
    "reader_outputs": [32],
    "change": [4, 2]
  },
  {
    "inputs": [
      {
        "amount": 8,
        "input_fee_ppk": 100
      },
      {
        "amount": 32,
        "input_fee_ppk": 400
      }
    ],
    "fee": 1,
    "outputs_total": 39,
    "reader_outputs": [32],
    "change": [4, 2, 1]
  },
  {
    "inputs": [
      {
        "amount": 32,
        "input_fee_ppk": 100
      },
      {
        "amount": 32,
        "input_fee_ppk": 100
      },
      {
        "amount": 32,
        "input_fee_ppk": 100
      },
      {
        "amount": 32,
        "input_fee_ppk": 100
      },
      {
        "amount": 32,
        "input_fee_ppk": 100
      },
      {
        "amount": 32,
        "input_fee_ppk": 100
      },
      {
        "amount": 32,
        "input_fee_ppk": 100
      },
      {
        "amount": 32,
        "input_fee_ppk": 100
      },
      {
        "amount": 32,
        "input_fee_ppk": 100
      },
      {
        "amount": 32,
        "input_fee_ppk": 100
      },
      {
        "amount": 32,
        "input_fee_ppk": 100
      }
    ],
    "fee": 2,
    "outputs_total": 350,
    "reader_outputs": [256, 64],
    "change": [16, 8, 4, 2]
  }
]
```

## Finishing the change

`GET_CHANGE` with P1 `00` answers a count and the owed opening's 83 bytes: its number, the amount, the keyset, the date, `B_` and the key it is bound to. The nonce and `r` are not in it.

```
01000000000800f1e4f5c2b1a090000000000214d74d45b98f556be3ecb84cb7b0570f0fa6815dca1d8251c43d7ae55b003758037ae23922d0a6bc02f80cd5768b5b2d529e64c70add504aec9b498529bcfac095
```

The mint's private key for `mint_keys["8"]` is not published, so its answer cannot be rebuilt. This vector of `LOAD_CHANGE` binds the same opening to a test key whose private key is published, `k = 8` (`K = 8·G`): `SPEND_ALL_CHANGE` names that key instead. The key is in neither `B_` nor the message, so `B_`, the message and the signature are the ones above, and `GET_CHANGE`'s row ends with this `K`. The mint signs `C_ = k·B_` and proves it with [NUT-12](https://github.com/cashubtc/nuts/blob/main/12.md) from the nonce `p` given (`R1 = p·G`, `R2 = p·B_`, `e` the hash of `R1`, `R2`, `K` and `C_` as uncompressed lowercase hex, `s = p + e·k`). `LOAD_CHANGE` with P1 `00` and P2 `02` gives the card `C_` and `K` uncompressed and `e`, `s`. The card checks the proof, unblinds `C = C_ - rK` itself with the `r` it kept, and writes the proof to the first empty slot, slot 2:

```json
{
  "mint_private_key_8": "0000000000000000000000000000000000000000000000000000000000000008",
  "K": "022f01e5e15cca351daff3843fb70f3c2f0a1bdd05e5af888a67784ef3e10a2a01",
  "K_uncompressed": "042f01e5e15cca351daff3843fb70f3c2f0a1bdd05e5af888a67784ef3e10a2a015c4da8a741539949293d082a132d13b4c2e213d6ba5b7617b5da2cb76cbde904",
  "spend_all_change": "b02600002d00000008022f01e5e15cca351daff3843fb70f3c2f0a1bdd05e5af888a67784ef3e10a2a0100f1e4f5c2b1a09021",
  "get_change_response": "01000000000800f1e4f5c2b1a090000000000214d74d45b98f556be3ecb84cb7b0570f0fa6815dca1d8251c43d7ae55b003758022f01e5e15cca351daff3843fb70f3c2f0a1bdd05e5af888a67784ef3e10a2a01",
  "mint_dleq_nonce": "0000000000000000000000000000000000000000000000000000000000000107",
  "C_": "033d23f51fbd7a6e2e2d270450468fb69e0fb096ddf6ddc2a7744a2b69b4065de1",
  "C_uncompressed": "043d23f51fbd7a6e2e2d270450468fb69e0fb096ddf6ddc2a7744a2b69b4065de1b4fd673bf45286cd427f446da1f5583195dcdd37fa71e0d276f0a2a4a49f454f",
  "e": "56befa8e9ead09858672757331d0c0375e961db21e90868afd87ed1b62fba245",
  "s": "b5f7d474f5684c2c3393ab998e8601bd7f5333c395f2f3e06c9aabc1777090ad",
  "load_change": "b0270002c2043d23f51fbd7a6e2e2d270450468fb69e0fb096ddf6ddc2a7744a2b69b4065de1b4fd673bf45286cd427f446da1f5583195dcdd37fa71e0d276f0a2a4a49f454f042f01e5e15cca351daff3843fb70f3c2f0a1bdd05e5af888a67784ef3e10a2a015c4da8a741539949293d082a132d13b4c2e213d6ba5b7617b5da2cb76cbde90456befa8e9ead09858672757331d0c0375e961db21e90868afd87ed1b62fba245b5f7d474f5684c2c3393ab998e8601bd7f5333c395f2f3e06c9aabc1777090ad53",
  "C": "030af3843a771002220ba6f792934279fb8bb1493f9f03e41b45c5bb4e498aeb11",
  "load_change_response": "020100f1e4f5c2b1a09000000008f60f52083006f305833a08f2d7b9a94614665a29cae239f8af980e36252db426030af3843a771002220ba6f792934279fb8bb1493f9f03e41b45c5bb4e498aeb1100000000"
}
```

The response is the slot (`02`), `01` for the DLEQ proof kept, and the slot's 81 bytes after the status byte. The card keeps `e`, `s` and its own `r` with the proof, so `GET_DLEQ` of slot 2 answers `01 ‖ e ‖ s ‖ r`, which a wallet checks as [NUT-CCP](../CCP.md#dleq) says. The card refuses each of these and writes nothing; the last is sent after the good answer, the others before it:

| Case                                                      | Status  |
| --------------------------------------------------------- | ------- |
| P2 `00`: no word on keys per load                         | `6A 86` |
| `K` of another key than the bound one                     | `6A 80` |
| `K` of the bound x with the other parity                  | `6A 80` |
| `C_` not on the curve                                     | `6A 80` |
| `e` changed in its last byte                              | `6A 80` |
| P1 `08`                                                   | `6A 83` |
| P1 of an opening not owed                                 | `6A 88` |
| the same `LOAD_CHANGE` again, after the first is answered | `6A 88` |
| 193 bytes                                                 | `67 00` |

## Spent-slot cleanup conformance

These cases describe reader behavior. They do not change the payment preimage, signature or APDU encoding above. `A` and `B` name distinct proof identifiers reconstructed from the card's retained spent slots.

| Card snapshot and mint observation                                                                                              | Reader behavior                                                                                                               |
| ------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| No spent slots                                                                                                                  | Loading may use empty slots without sending `CLEAR_SPENT`.                                                                    |
| Spent `A`; mint reports `A: UNSPENT` or `A: PENDING`                                                                            | Preserve `A`; do not clear.                                                                                                   |
| Spent `A, B`; mint reports only `A: SPENT`                                                                                      | Incomplete coverage; do not clear.                                                                                            |
| Spent `A, B`; mint returns duplicate `A` instead of `B`, an unrelated identifier or contradictory states                        | Invalid coverage; do not clear.                                                                                               |
| Spent `A, B`; mint reports both `SPENT`, but card identity or any retained proof changes before cleanup                         | Discard the cleanup authorization and inspect again.                                                                          |
| Spent `A, B`; the exact snapshot is revalidated and both mint states are `SPENT`                                                | Proof-slot cleanup may proceed under card policy. Preserve unresolved merchant-output and change-recovery records separately. |
| Mint request fails, proof reconstruction fails or retained contents are malformed without a verified interrupted-cleanup record | Stop cleanup and preserve recovery evidence.                                                                                  |
