# NUT-CCP Test Vectors

All values are hex unless noted. Each vector can be checked with BIP-340, SHA-256 and the `hash_to_curve` of [NUT-00](https://github.com/cashubtc/nuts/blob/main/00.md).

## Card proof secrets

A card rebuilds each secret from its stored slot, its key and its refund key. These vectors come from the reference applet.

```json
{
  "card_key": "02fa05ed0a3649886270bacec5ddc1a3643009a759bee996f9a6ac060c9992a82e",
  "refund_key": "0279be667ef9dcbbac55a06295ce870b07029bfcdb2dce28d959f2815b16f81798"
}
```

Slot (82 bytes):

```
010059534ce0bfa19a00000008404142434445464748494a4b4c4d4e4f505152535455565758595a5b5c5d5e5f02414141414141414141414141414141414141414141414141414141414141414100000000
```

Secret:

```
["P2PK",{"nonce":"404142434445464748494a4b4c4d4e4f505152535455565758595a5b5c5d5e5f","data":"02fa05ed0a3649886270bacec5ddc1a3643009a759bee996f9a6ac060c9992a82e","tags":[["sigflag","SIG_ALL"]]}]
```

Slot (82 bytes):

```
010059534ce0bfa19a000000104748494a4b4c4d4e4f505152535455565758595a5b5c5d5e5f6061626364656602484848484848484848484848484848484848484848484848484848484848484800000001
```

Secret:

```
["P2PK",{"nonce":"4748494a4b4c4d4e4f505152535455565758595a5b5c5d5e5f60616263646566","data":"02fa05ed0a3649886270bacec5ddc1a3643009a759bee996f9a6ac060c9992a82e","tags":[["locktime","1"],["refund","0279be667ef9dcbbac55a06295ce870b07029bfcdb2dce28d959f2815b16f81798"],["sigflag","SIG_ALL"]]}]
```

Slot (82 bytes):

```
010059534ce0bfa19a000000204e4f505152535455565758595a5b5c5d5e5f606162636465666768696a6b6c6d024f4f4f4f4f4f4f4f4f4f4f4f4f4f4f4f4f4f4f4f4f4f4f4f4f4f4f4f4f4f4f4f6553f100
```

Secret:

```
["P2PK",{"nonce":"4e4f505152535455565758595a5b5c5d5e5f606162636465666768696a6b6c6d","data":"02fa05ed0a3649886270bacec5ddc1a3643009a759bee996f9a6ac060c9992a82e","tags":[["locktime","1700000000"],["refund","0279be667ef9dcbbac55a06295ce870b07029bfcdb2dce28d959f2815b16f81798"],["sigflag","SIG_ALL"]]}]
```

Slot (82 bytes):

```
010059534ce0bfa19a0000004055565758595a5b5c5d5e5f606162636465666768696a6b6c6d6e6f7071727374025656565656565656565656565656565656565656565656565656565656565656713fb300
```

Secret:

```
["P2PK",{"nonce":"55565758595a5b5c5d5e5f606162636465666768696a6b6c6d6e6f7071727374","data":"02fa05ed0a3649886270bacec5ddc1a3643009a759bee996f9a6ac060c9992a82e","tags":[["locktime","1900000000"],["refund","0279be667ef9dcbbac55a06295ce870b07029bfcdb2dce28d959f2815b16f81798"],["sigflag","SIG_ALL"]]}]
```

Slot (82 bytes):

```
010059534ce0bfa19a000000805c5d5e5f606162636465666768696a6b6c6d6e6f707172737475767778797a7b025d5d5d5d5d5d5d5d5d5d5d5d5d5d5d5d5d5d5d5d5d5d5d5d5d5d5d5d5d5d5d5dffffffff
```

Secret:

```
["P2PK",{"nonce":"5c5d5e5f606162636465666768696a6b6c6d6e6f707172737475767778797a7b","data":"02fa05ed0a3649886270bacec5ddc1a3643009a759bee996f9a6ac060c9992a82e","tags":[["locktime","4294967295"],["refund","0279be667ef9dcbbac55a06295ce870b07029bfcdb2dce28d959f2815b16f81798"],["sigflag","SIG_ALL"]]}]
```

## A payment of one proof

For each slot above, the card signed a payment of that one proof into no outputs. The message is `secret || C`, so its hash is `SHA-256(secret || C)`. The signature is BIP-340 by the card key.

```json
{
  "amount": 8,
  "date": 0,
  "msg_hash": "3dec83ed1cd887560921cbb24c8730dfe6d52ed37e2339ef307a0f618cbc3da5",
  "signature": "a55fe50cf94bae97a88e3343099d55d95cb8962ca53a2c03e939e4c207efd675794ac14129ef173081a7659779762da6e759926352074b7988421fa46e2a3419"
}
```

```json
{
  "amount": 16,
  "date": 1,
  "msg_hash": "dea05bc801903c31102f08b1599d2b1d9e1aa64494fd39b245b078b6702c69e3",
  "signature": "b4099f0639f7eebcd2519e0f65633b2653ae9fce975ce36881c40fc74f706ad897d813c7482acfc9cf00ce38507599b3680df632fd9467d75d35716ae6869272"
}
```

```json
{
  "amount": 32,
  "date": 1700000000,
  "msg_hash": "93d727fba024c4d58a716a7faa7133779530ea0487e295cae9c8bce8809d6daf",
  "signature": "81a39b436b2f74e564867d5a01fc2eb04a75295f801dfdc9646f52183798c46e92b985cd0f20c4d7889b66039f41d717566d3dabb94b2d875d335098a050ab3c"
}
```

```json
{
  "amount": 64,
  "date": 1900000000,
  "msg_hash": "e6451dc60f54c47f6397636568ba9e9b27424b3ea3599c959645715cc39e463c",
  "signature": "e888c5234b2209eb4a5ec161eed713f60129c43912224377a294a22a8474be9d87d8d223e6f4744f19f8d49f5cbfd4614a968fa87a2120f462621c1cb01efe78"
}
```

```json
{
  "amount": 128,
  "date": 4294967295,
  "msg_hash": "1b3d3eafe4eea03c02677bffcf0a64c98a5bf053260368d11a6e88560ba89cc1",
  "signature": "08059f583b4512c148c10ef5deefd7fe40a4d54e3696fffe30e972f70d089931cea954f16307dd2c2ab31db7c65e64f3b65d87b8ddfa31ab86021afe36ee971b"
}
```

## AUTH

`T = SHA-256("CashuCard/auth")`, and the card signs `SHA-256(T || T || reader_nonce || card_nonce || card_key)`.

```json
{
  "card_key": "02fa05ed0a3649886270bacec5ddc1a3643009a759bee996f9a6ac060c9992a82e",
  "reader_nonce": "a0a1a2a3a4a5a6a7a8a9aaabacadaeaf",
  "card_nonce": "3ce74d3ef4943100a0eacd79f28c7649",
  "msg_hash": "c339376c2ab0678e599cb054f139794eab1b412152e3782f14eaf1271b0afc4e",
  "signature": "3d78314dce2a61a99ba34c8d3ca71302a3121684149e44cc6c25460f90a31b1ae32419868631c8ad2ae9e9d53e1f3cebb651e5c39bf634507f5d995501affc6c"
}
```

## A payment of two proofs with change

A card holding proofs of 8 and 32 pays 32 to a reader and makes 8 of change for itself. The card's private key and the mint's keys are given so every value can be rebuilt. The keyset has no fees.

```json
{
  "card_private_key": "3bee8a332dc85965b145c733fd94a532418d23499dd93290c449ef93b64202d7",
  "card_key": "0248f87c8ddddced1b7870d0461b6e2e68608f101828c8ebb7bdc3bbeed9b6f287",
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
    "secret": "[\"P2PK\",{\"nonce\":\"d7a1a81efa4268f1b7a43db2501ddeab6f6ba63d1cbd2a035a17b03955bf2af2\",\"data\":\"0248f87c8ddddced1b7870d0461b6e2e68608f101828c8ebb7bdc3bbeed9b6f287\",\"tags\":[[\"sigflag\",\"SIG_ALL\"]]}]",
    "C": "03f0842d7b4da44cb02c71944a521481cd56717bf719fcbeaaa2400950c31f45d9"
  },
  {
    "amount": 32,
    "nonce": "61f41166d013508a53e670ef4ba10fbe4dceb9d58d8c5909f30daf2c79ddb613",
    "secret": "[\"P2PK\",{\"nonce\":\"61f41166d013508a53e670ef4ba10fbe4dceb9d58d8c5909f30daf2c79ddb613\",\"data\":\"0248f87c8ddddced1b7870d0461b6e2e68608f101828c8ebb7bdc3bbeed9b6f287\",\"tags\":[[\"sigflag\",\"SIG_ALL\"]]}]",
    "C": "025e1c7a2e84e898cde358b34a77ff2ddddd2edb4d6d78a5f6cb77d915c5a6b17a"
  }
]
```

The commands and the card's responses (data only; each ends `90 00`):

```json
[
  {
    "command": "SPEND_ALL_BEGIN",
    "apdu": "b022000002000104",
    "response": "00000028"
  },
  {
    "command": "SPEND_ALL_OUTPUTS",
    "apdu": "b023000025000000200211bccaf554963e836ce04494fec4e539d8ddbf3bac412c761f8e7849adf26f2c",
    "response": ""
  },
  {
    "command": "SPEND_ALL_CHANGE",
    "apdu": "b0260000040000000821",
    "response": "02f38349c2c6c6d44acd99ef8d0efac8b69bf643f43af1bd8179534c778a52f3a2"
  },
  {
    "command": "SPEND_ALL_SIGN",
    "apdu": "b024000040",
    "response": "61abdf4f5072b19d20aa08de173c4955744eeb800ee7eb024f79fa2acad10ad1797be7112a2a1bd7e069c4bf369bb5ff2190cd1788319771dccb2d907a557802"
  }
]
```

The reader's output is for the secret `9bcb4ac6d7eca4e268ca88200bb93ad4e1c1998583674593fa60ae58be75185d` (UTF-8) blinded with `r = e2fae0673f3205892ef427d3341088a752ef126309e1a1feb2ae3d7abf12934c`. The card's change output is for this secret, blinded with the `r` below:

```json
{
  "secret": "[\"P2PK\",{\"nonce\":\"f60f52083006f305833a08f2d7b9a94614665a29cae239f8af980e36252db426\",\"data\":\"0248f87c8ddddced1b7870d0461b6e2e68608f101828c8ebb7bdc3bbeed9b6f287\",\"tags\":[[\"sigflag\",\"SIG_ALL\"]]}]",
  "r": "a93cf0318c22652f73fa1e1a79d90386cbce181b614ac6623f035580d0921ecb",
  "B_": "02f38349c2c6c6d44acd99ef8d0efac8b69bf643f43af1bd8179534c778a52f3a2"
}
```

The message, as UTF-8 text:

```
["P2PK",{"nonce":"d7a1a81efa4268f1b7a43db2501ddeab6f6ba63d1cbd2a035a17b03955bf2af2","data":"0248f87c8ddddced1b7870d0461b6e2e68608f101828c8ebb7bdc3bbeed9b6f287","tags":[["sigflag","SIG_ALL"]]}]03f0842d7b4da44cb02c71944a521481cd56717bf719fcbeaaa2400950c31f45d9["P2PK",{"nonce":"61f41166d013508a53e670ef4ba10fbe4dceb9d58d8c5909f30daf2c79ddb613","data":"0248f87c8ddddced1b7870d0461b6e2e68608f101828c8ebb7bdc3bbeed9b6f287","tags":[["sigflag","SIG_ALL"]]}]025e1c7a2e84e898cde358b34a77ff2ddddd2edb4d6d78a5f6cb77d915c5a6b17a320211bccaf554963e836ce04494fec4e539d8ddbf3bac412c761f8e7849adf26f2c802f38349c2c6c6d44acd99ef8d0efac8b69bf643f43af1bd8179534c778a52f3a2
```

```json
{
  "msg_hash": "9a215c5682c90d6e5553a62fa528de359c96ca68ac00e048f26bdf4dc5297a27",
  "signature": "61abdf4f5072b19d20aa08de173c4955744eeb800ee7eb024f79fa2acad10ad1797be7112a2a1bd7e069c4bf369bb5ff2190cd1788319771dccb2d907a557802"
}
```

The swap the reader sends to the mint ([NUT-03](https://github.com/cashubtc/nuts/blob/main/03.md)):

```json
{
  "inputs": [
    {
      "amount": 8,
      "id": "00f1e4f5c2b1a090",
      "secret": "[\"P2PK\",{\"nonce\":\"d7a1a81efa4268f1b7a43db2501ddeab6f6ba63d1cbd2a035a17b03955bf2af2\",\"data\":\"0248f87c8ddddced1b7870d0461b6e2e68608f101828c8ebb7bdc3bbeed9b6f287\",\"tags\":[[\"sigflag\",\"SIG_ALL\"]]}]",
      "C": "03f0842d7b4da44cb02c71944a521481cd56717bf719fcbeaaa2400950c31f45d9",
      "witness": "{\"signatures\":[\"61abdf4f5072b19d20aa08de173c4955744eeb800ee7eb024f79fa2acad10ad1797be7112a2a1bd7e069c4bf369bb5ff2190cd1788319771dccb2d907a557802\"]}"
    },
    {
      "amount": 32,
      "id": "00f1e4f5c2b1a090",
      "secret": "[\"P2PK\",{\"nonce\":\"61f41166d013508a53e670ef4ba10fbe4dceb9d58d8c5909f30daf2c79ddb613\",\"data\":\"0248f87c8ddddced1b7870d0461b6e2e68608f101828c8ebb7bdc3bbeed9b6f287\",\"tags\":[[\"sigflag\",\"SIG_ALL\"]]}]",
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
      "B_": "02f38349c2c6c6d44acd99ef8d0efac8b69bf643f43af1bd8179534c778a52f3a2"
    }
  ]
}
```

## Finishing the change

`GET_CHANGE` with P1 `00` answers a count and the change's 80-byte opening (amount, keyset, date, nonce, `r`):

```
010000000800f1e4f5c2b1a09000000000f60f52083006f305833a08f2d7b9a94614665a29cae239f8af980e36252db426a93cf0318c22652f73fa1e1a79d90386cbce181b614ac6623f035580d0921ecb
```

The mint signs the change output with its key for 8. The reader unblinds `C = C_ - rK` and loads the proof with `LOAD_PROOF`:

```json
{
  "K": "037ae23922d0a6bc02f80cd5768b5b2d529e64c70add504aec9b498529bcfac095",
  "C_": "0298f4e65c927aa2f7af5dfff662c7bec0bda9b256683a89351d9e609112cca546",
  "C": "0253223d823a7ca34048f67abc6df1782f1c14e86c2aa0f1a16c5725ecc6a941ec",
  "load_proof_apdu": "b03000005100f1e4f5c2b1a09000000008f60f52083006f305833a08f2d7b9a94614665a29cae239f8af980e36252db4260253223d823a7ca34048f67abc6df1782f1c14e86c2aa0f1a16c5725ecc6a941ec0000000001"
}
```
