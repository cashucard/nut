# NUT-CCP profile: Certificate

`optional`

`depends on: NUT-CCP`

---

This profile lets whoever vouches for a card, its **issuer** (a certifier), sign a statement that the card key `P_card` was made on a chip it issued, and keep that statement on the card for any reader to fetch. There is no single issuer: each issuer signs its own cards with its own keys, which this profile does not handle. A wallet checks a card against the issuers it chooses to trust.

The certificate names `P_card`. Every `AUTH` answer is a BIP-340 signature by that key ([NUT-CCP](../CCP.md#auth)), and every payment is signed by a date key built on it, `P_card + t·G`, with the `t` the card answers to `SPEND_ALL_BEGIN` ([Keys per load](../CCP.md#keys-per-load)). A reader that has checked the certificate and then sees an `AUTH` signature under the key it names knows it is talking to that chip, now.

A certificate is public. It holds nothing secret and the card signs nothing with it.

## Why a certificate

A reader can check a good deal about a card for itself: that it holds the key it names (`AUTH`), that the proofs on it are the mint's (their DLEQ proofs), that it answers the protocol correctly. It cannot check from outside what code the card runs, or that the card's private key was made on the chip and has never left it. Someone has to vouch for how the card was made. A certificate carries that word to the reader, bound to the card's key: checked with a live `AUTH` under the key it names, it is useless on any other chip.

**What an issuer takes on.** An issuer vouches only for cards made by the process it states. It keeps the records that let it say which cards it vouched for. When a card or a batch it vouched for turns out bad, it says so, so that readers stop believing it. Its keys are what make its word checkable: whoever holds them can vouch as the issuer.

**The risks of a card with no certificate, or with one from an issuer the reader does not know or trust.**

- To the holder: whoever made the card may hold its key or have changed its code, and may spend what is loaded on it. Its PIN and its limits may not hold.
- To a reader paid offline: the proofs it is paid with may be copies, or may be taken back by whoever holds the key, and nothing on the card can show otherwise until the reader reaches the mint.

A wallet weighs this before it loads a card, or takes a payment from one offline.

## Terms

- **Issuer**: whoever signs certificates for cards. An issuer has an **issuer ID**, 1 to 32 characters from `a` to `z`, `0` to `9`, `.` and `-`.
- **Statement**: the one line of text the issuer signs.
- **Container**: the bytes written to the card: the card's own part, the statement, the signature and the issuer's chain.

## The container

All numbers are big-endian.

| Size   | Field                                                                    |
| ------ | ------------------------------------------------------------------------ |
| 8      | `CCP-CERT`, ASCII                                                        |
| 1      | version, `01`                                                            |
| 1      | form: `01` one card, `02` a batch                                        |
| 33     | form `01`: `P_card`, compressed                                          |
| 2      | form `02`: the card's leaf index in the batch tree                       |
| 1      | form `02`: the depth of the tree, 0 to 16                                |
| 32 × d | form `02`: the path, from the leaf's sibling up                          |
| 32     | form `02`: the tree's top                                                |
| 2, n   | the statement's length and the statement, ASCII                          |
| 2, m   | the signature's length and the signature                                 |
| 2, c   | the chain's length and the chain: the issuer's own credentials, or empty |

The container is at most 1,024 bytes, and ends with the chain.

### The batch tree

An issuer that signs many cards at once signs one statement over a hash tree of their keys:

```
leaf  = SHA-256( 0x00 || "CashuCard/cert-leaf/1" || P_card )
empty = SHA-256( 0x02 || "CashuCard/cert-empty/1" )
node  = SHA-256( 0x01 || left || right )
```

The leaves are padded with `empty` to the next power of two, and numbered from 0. Bit `i` of a leaf's index, counted from the lowest, is `1` when the node at height `i` on the way up is a right child; the index has no bit set at or past the depth. An issuer **SHOULD** order the leaves by their own hash, so that a leaf's index says nothing about the order the cards were made in.

## The statement

One line of ASCII, fields in this order, separated by one space, with no other characters:

```
cashu-card-cert/1 issuer=<issuer ID> form=card key=<P_card> applet=<version> build=<hash> date=<date> scheme=<scheme>
cashu-card-cert/1 issuer=<issuer ID> form=batch top=<top> applet=<version> build=<hash> date=<date> scheme=<scheme>
cashu-card-cert/1 issuer=<issuer ID> form=batch top=<top> count=<count> applet=<version> build=<hash> date=<date> scheme=<scheme>
```

- `<P_card>`: 66 lowercase hex characters, the compressed key.
- `<top>`: 64 lowercase hex characters, the tree's top.
- `<count>`: the number of cards in the batch, 1 to 65535, in decimal with no leading zero. It is optional: an issuer that does not want to say how many cards it made leaves the field out, and a reader **MUST NOT** require it.
- `<version>`: the two bytes the card answers to `SELECT`, as 4 lowercase hex characters.
- `<hash>`: the SHA-256 of the card application's load file, the bytes exactly as they are LOADed onto the card, 64 lowercase hex characters: the tag `C4`; the BER length of what follows (one byte under 128, `81` and one byte under 256, `82` and two bytes, big-endian, above); then the CAP's components Header, Directory, Import, Applet, Class, Method, StaticField, Export, ConstantPool and RefLocation, in that order, those the CAP has, each whole as the CAP stores it (its own tag and size included), with no Descriptor or Debug component. It is not the hash of the CAP archive, which carries the time it was built: an issuer publishes the load file's hash beside the CAP's, and a reader compares this field with the load file's.
- `<date>`: the day the statement was signed, `YYYY-MM-DD`.
- `<scheme>`: the signature scheme, below.

A line that does not match exactly is not a statement. The `key` or `top` of the statement **MUST** be the one in the container's own part.

### Signature schemes

| Scheme   | Signature                                                                                                                  |
| -------- | -------------------------------------------------------------------------------------------------------------------------- |
| `bip340` | 64 bytes: a BIP-340 signature over `SHA-256( T \|\| T \|\| line )`, `T = SHA-256("CashuCard/cert/1")`, by the issuer's key |
| `bip137` | 65 bytes: a Bitcoin signed message of the line, in the compact form for a compressed key                                   |

A reader **MUST** refuse a scheme it does not know. How a reader comes to trust an issuer's key, what the chain holds and how an issuer withdraws a batch are outside this profile: each wallet decides which issuers it trusts and keeps its own list.

## On the card

The card stores the container and checks only that it is well formed and names the card's own key. It reads neither the statement nor the signature, and checks no signature: that is the reader's work.

### CERT_READ (INS `1C`)

Anyone, no PIN, any state.

| P1        | Answer                                                                                                  |
| --------- | ------------------------------------------------------------------------------------------------------- |
| `00`–`03` | 256 bytes of the container from offset `P1 × 256`, fewer for the last page; `6A 83` past its end        |
| `FF`      | 4 bytes: `01` when a certificate is sealed, else `00`; the container's version, or `00`; its length (2) |

A page is answered only from a sealed certificate: `6A 88` otherwise. Any other P1, or a P2 other than `00`, is `6A 86`.

### Writing and sealing

The issuer writes the container with a profile command (the reference implementation's `CERT_WRITE`, INS `54`: P1 `00` writes 1 to 240 bytes at an offset, P1 `01` seals), in pieces, then seals it. A card:

- **MUST** take a write only while the card has never had a PIN, has no owner key, and holds no proof and no owed change, and while no certificate is sealed;
- **MUST** count nothing written until the seal, and never serve bytes that were not sealed;
- **MUST** seal only the container its writer sent: the reference implementation's seal carries the container's length and SHA-256, and the card refuses a seal whose hash is not that of the bytes it holds, so that bytes another reader wrote in between, or a write that went wrong on the air, are never sealed;
- **MUST**, at the seal, check that the container is well formed and names its own key: for form `01`, that the 33 bytes are `P_card`; for form `02`, that the leaf made from `P_card`, folded up the path by its index, is the top. It refuses the seal otherwise (`6A 80`) and seals nothing;
- **MUST** commit the seal by one atomic write, after which the container is never written again;
- **MUST** keep a sealed certificate, and the card key it names, for as long as the card's application is installed, through every reset.

The reference implementation's `CERT_WRITE` answers, in order: `6A 86` for a P1 other than `00` or `01` or a P2 other than `00`; `69 85` once a certificate is sealed or the card has had a PIN; `6A 91` on a card with an owner; `6A 8D` with a proof or change owed on the card; `67 00` for a write of no bytes or more than 240, or a seal not of 34 bytes; `6A 83` for bytes that would end past 1,024; `6A 84` where the card has no room for the certificate; then, at the seal, `6A 80` for a hash that is not that of the bytes held, or a container that is not well formed or does not name the card's key.

An issuer that certifies a card writes and seals the certificate before the card's first PIN, before the card leaves its hands.

## Checking a card

A reader that checks a card:

1. reads `CERT_READ` P1 `FF`, then the pages;
2. parses the container and the statement, and refuses either if it is not exactly as above;
3. checks that the statement's key or top is the container's, and for form `02` folds the leaf of the key it reads from the card (`GET_PUBKEY`) up the path to the top;
4. checks the signature under the scheme named, and the issuer, its chain and the `applet` and `build` against the ones it trusts;
5. in the same tap, asks `AUTH` with fresh bytes of its own and checks the answer under the key the certificate names.

Steps 3 and 5 make the certificate useless on any other chip: a copied certificate names a key the other chip cannot sign for, and a chip's own key does not fold to another card's top. A reader paid by the card builds each input's secret with `P_card + t·G` from that key and the `t` of `SPEND_ALL_BEGIN`; only the chip that holds `P_card` can sign for it.

## Security considerations

- **A certificate shows where a key was made, not what the chip does now.** A key taken out of a chip, or a chip made to misbehave by its maker, still passes. A certificate is worth what its issuer is.
- **A reset keeps the key.** A card that keeps its certificate through a reset keeps the key it names, so anyone the card pays, and anyone who reads it (`GET_PUBKEY`, `AUTH`, the certificate), can tell it is the same card under every holder. The mint does not see `P_card`: the proofs name date keys ([Keys per load](../CCP.md#keys-per-load)).
- **Ecash before a hand-over.** A reset draws a new card secret `c`, so the card builds other date keys after it. Ecash locked to the card's date keys before a hand-over and not loaded is not the next holder's: dated, it is the old holder's to take back with the refund key after its date; undated, it is lost. The holder loads or takes back what is owed before handing the card on.
- **Never lock ecash to the certified key.** `P_card` is for checking the card. The card never signs a payment with it, so ecash locked to it is never spendable by the card ([NUT-CCP](../CCP.md#keys-per-load)). A sender who is not the holder gives a gift to the holder's wallet, which loads it.
- **Who may write.** A card that has never had a PIN and has no owner is open to whoever holds it. An issuer writes and seals the certificate before the card leaves its hands. A card sealed by someone else, with a signature no reader trusts, takes no other certificate and keeps its key through every reset: a reader treats a certificate it cannot check as none, and knows that such a card's key does not change at a reset.

## Test vectors

Recorded from the reference implementation. The issuer key and the card keys are test keys only, each the SHA-256 of its name (`CashuCard/cert test issuer`, `CashuCard/cert test card 0` to `3`) reduced mod `n`; `build` is the SHA-256 of `CashuCard/cert test build`. The statements are signed under `bip340` with 32 zero bytes of auxiliary randomness, so the signatures can be made again. Each `cert_write` is a `CERT_WRITE` (INS `54`, P1 `00`: the offset (2) and up to 240 bytes), and `cert_seal` its seal (P1 `01`: the container's length (2) and its SHA-256).

The issuer:

```json
{
  "issuer_private_key": "b1dd5491eaa6cfd0ec86a64609c3508e07aa880f277ced17b2b6e67fbd786ba7",
  "issuer_public_key": "f290447ea9b3341452649d3f35e9723a176444420be27623c30313a6c090ba23",
  "statement_tag": "CashuCard/cert/1",
  "signature_aux": "0000000000000000000000000000000000000000000000000000000000000000"
}
```

Form `01`, one card. `cert_read_ff` is `CERT_READ` P1 `FF` after the seal: sealed, version `01`, and the length.

```json
{
  "card_private_key": "cf5f10ae0c999a0db5ea97023a78a94ec54baca950978208e27e59458e94dce9",
  "card_key": "027ac53e398ae03ac0adf29e7903596cf2a142e5b8eb40f401db326df3dab8081e",
  "statement": "cashu-card-cert/1 issuer=test-issuer form=card key=027ac53e398ae03ac0adf29e7903596cf2a142e5b8eb40f401db326df3dab8081e applet=0203 build=749622759b6a4b4013c09eff4c5347d195c974757e90a7dc339bcccbf382eb34 date=2026-11-12 scheme=bip340",
  "signature": "6ac5fdb5ce99c40f852681d731777f0aeead7ba2fb7e27a37cf95389c72503297cc0a669acab49d63b9eb7067d4f2eb4b0ce9db2a15da2bcabc3f21e53cb2ea4",
  "container": "4343502d434552540101027ac53e398ae03ac0adf29e7903596cf2a142e5b8eb40f401db326df3dab8081e00e663617368752d636172642d636572742f31206973737565723d746573742d69737375657220666f726d3d63617264206b65793d303237616335336533393861653033616330616466323965373930333539366366326131343265356238656234306634303164623332366466336461623830383165206170706c65743d30323033206275696c643d3734393632323735396236613462343031336330396566663463353334376431393563393734373537653930613764633333396263636362663338326562333420646174653d323032362d31312d313220736368656d653d62697033343000406ac5fdb5ce99c40f852681d731777f0aeead7ba2fb7e27a37cf95389c72503297cc0a669acab49d63b9eb7067d4f2eb4b0ce9db2a15da2bcabc3f21e53cb2ea40000",
  "cert_write": [
    "b0540000f200004343502d434552540101027ac53e398ae03ac0adf29e7903596cf2a142e5b8eb40f401db326df3dab8081e00e663617368752d636172642d636572742f31206973737565723d746573742d69737375657220666f726d3d63617264206b65793d303237616335336533393861653033616330616466323965373930333539366366326131343265356238656234306634303164623332366466336461623830383165206170706c65743d30323033206275696c643d3734393632323735396236613462343031336330396566663463353334376431393563393734373537653930613764633333396263636362663338",
    "b05400006900f0326562333420646174653d323032362d31312d313220736368656d653d62697033343000406ac5fdb5ce99c40f852681d731777f0aeead7ba2fb7e27a37cf95389c72503297cc0a669acab49d63b9eb7067d4f2eb4b0ce9db2a15da2bcabc3f21e53cb2ea40000"
  ],
  "cert_seal": "b0540100220157a3912b141235f1ba82dfcf5849281c340b66e19a06097f5eb6e78724abbde9dd",
  "cert_read_ff": "01010157"
}
```

Form `02`, a batch of four cards. The leaves are ordered by their own hash and numbered from 0; each card's container carries its index, its path from its leaf's sibling up, and the top. The statement here carries the optional `count`.

```json
{
  "leaf_tag": "CashuCard/cert-leaf/1",
  "statement": "cashu-card-cert/1 issuer=test-issuer form=batch top=3910e1237a23a4bd2f603294b96049c91cd28f33c825b4b2e7717e31e38da46f count=4 applet=0203 build=749622759b6a4b4013c09eff4c5347d195c974757e90a7dc339bcccbf382eb34 date=2026-11-12 scheme=bip340",
  "signature": "842c10772491159af3f8983e148cc85c0444d429ad33634f9748199c21f35e4acbea39b48f99f11fff1b5c8343fa8bc5b7d8cac57459010b3e11f3e2818e4542",
  "top": "3910e1237a23a4bd2f603294b96049c91cd28f33c825b4b2e7717e31e38da46f"
}
```

```json
{
  "index": 0,
  "card_private_key": "a1340683b76547f04e1e803886f1e9b0107b29ed478fb9042599b0699f485b9f",
  "card_key": "03983f5f907c7bda7bcc08d9bdf8352103c9c2ab9728daa05de1362293aee257c4",
  "leaf": "5817f8ce787c46a1ed70630bad3206edb9d65ee121ab0d4dda510aebf598c74f",
  "path": [
    "76b4f370e18db293c31aeea24765ec800d16510939ccf009776a1057113e0c88",
    "ea67b80bb29043df3097c23e40a25b3896507ede7e5023df3beeaf320708b656"
  ],
  "container": "4343502d43455254010200000276b4f370e18db293c31aeea24765ec800d16510939ccf009776a1057113e0c88ea67b80bb29043df3097c23e40a25b3896507ede7e5023df3beeaf320708b6563910e1237a23a4bd2f603294b96049c91cd28f33c825b4b2e7717e31e38da46f00ed63617368752d636172642d636572742f31206973737565723d746573742d69737375657220666f726d3d626174636820746f703d3339313065313233376132336134626432663630333239346239363034396339316364323866333363383235623462326537373137653331653338646134366620636f756e743d34206170706c65743d30323033206275696c643d3734393632323735396236613462343031336330396566663463353334376431393563393734373537653930613764633333396263636362663338326562333420646174653d323032362d31312d313220736368656d653d6269703334300040842c10772491159af3f8983e148cc85c0444d429ad33634f9748199c21f35e4acbea39b48f99f11fff1b5c8343fa8bc5b7d8cac57459010b3e11f3e2818e45420000"
}
```

```json
{
  "index": 1,
  "card_private_key": "cf5f10ae0c999a0db5ea97023a78a94ec54baca950978208e27e59458e94dce9",
  "card_key": "027ac53e398ae03ac0adf29e7903596cf2a142e5b8eb40f401db326df3dab8081e",
  "leaf": "76b4f370e18db293c31aeea24765ec800d16510939ccf009776a1057113e0c88",
  "path": [
    "5817f8ce787c46a1ed70630bad3206edb9d65ee121ab0d4dda510aebf598c74f",
    "ea67b80bb29043df3097c23e40a25b3896507ede7e5023df3beeaf320708b656"
  ],
  "container": "4343502d4345525401020001025817f8ce787c46a1ed70630bad3206edb9d65ee121ab0d4dda510aebf598c74fea67b80bb29043df3097c23e40a25b3896507ede7e5023df3beeaf320708b6563910e1237a23a4bd2f603294b96049c91cd28f33c825b4b2e7717e31e38da46f00ed63617368752d636172642d636572742f31206973737565723d746573742d69737375657220666f726d3d626174636820746f703d3339313065313233376132336134626432663630333239346239363034396339316364323866333363383235623462326537373137653331653338646134366620636f756e743d34206170706c65743d30323033206275696c643d3734393632323735396236613462343031336330396566663463353334376431393563393734373537653930613764633333396263636362663338326562333420646174653d323032362d31312d313220736368656d653d6269703334300040842c10772491159af3f8983e148cc85c0444d429ad33634f9748199c21f35e4acbea39b48f99f11fff1b5c8343fa8bc5b7d8cac57459010b3e11f3e2818e45420000"
}
```

```json
{
  "index": 2,
  "card_private_key": "b95551f203713a61f7100b79b9f490f11fca5fb901ac0ca8951953b8907a80bc",
  "card_key": "02cc13c17998f6b440b4e489ff22cc17f01aacb3a15627c25c74db48ad13c736d8",
  "leaf": "cc9dbc15e4686812c9cdd5312319fd5ab38924c6efc6174265a7a124bdc3db6d",
  "path": [
    "df81ffcf2528eca58079302ee786490d955cff49ba8ae9d11efe0f598bf1d035",
    "549af7396ed4eddb979b32a0d9b0626e2f4a48eed171cc47043b98592d6b39ec"
  ],
  "container": "4343502d434552540102000202df81ffcf2528eca58079302ee786490d955cff49ba8ae9d11efe0f598bf1d035549af7396ed4eddb979b32a0d9b0626e2f4a48eed171cc47043b98592d6b39ec3910e1237a23a4bd2f603294b96049c91cd28f33c825b4b2e7717e31e38da46f00ed63617368752d636172642d636572742f31206973737565723d746573742d69737375657220666f726d3d626174636820746f703d3339313065313233376132336134626432663630333239346239363034396339316364323866333363383235623462326537373137653331653338646134366620636f756e743d34206170706c65743d30323033206275696c643d3734393632323735396236613462343031336330396566663463353334376431393563393734373537653930613764633333396263636362663338326562333420646174653d323032362d31312d313220736368656d653d6269703334300040842c10772491159af3f8983e148cc85c0444d429ad33634f9748199c21f35e4acbea39b48f99f11fff1b5c8343fa8bc5b7d8cac57459010b3e11f3e2818e45420000"
}
```

```json
{
  "index": 3,
  "card_private_key": "86da2016da4e06f9b287e0d36feb444a4fb012a7cc58023b5a2591c9eb57a936",
  "card_key": "034b9cd351f57a909125125702fa652bba00bea7acf9d54ea3522d696cc9736086",
  "leaf": "df81ffcf2528eca58079302ee786490d955cff49ba8ae9d11efe0f598bf1d035",
  "path": [
    "cc9dbc15e4686812c9cdd5312319fd5ab38924c6efc6174265a7a124bdc3db6d",
    "549af7396ed4eddb979b32a0d9b0626e2f4a48eed171cc47043b98592d6b39ec"
  ],
  "container": "4343502d434552540102000302cc9dbc15e4686812c9cdd5312319fd5ab38924c6efc6174265a7a124bdc3db6d549af7396ed4eddb979b32a0d9b0626e2f4a48eed171cc47043b98592d6b39ec3910e1237a23a4bd2f603294b96049c91cd28f33c825b4b2e7717e31e38da46f00ed63617368752d636172642d636572742f31206973737565723d746573742d69737375657220666f726d3d626174636820746f703d3339313065313233376132336134626432663630333239346239363034396339316364323866333363383235623462326537373137653331653338646134366620636f756e743d34206170706c65743d30323033206275696c643d3734393632323735396236613462343031336330396566663463353334376431393563393734373537653930613764633333396263636362663338326562333420646174653d323032362d31312d313220736368656d653d6269703334300040842c10772491159af3f8983e148cc85c0444d429ad33634f9748199c21f35e4acbea39b48f99f11fff1b5c8343fa8bc5b7d8cac57459010b3e11f3e2818e45420000"
}
```

The seals a card refuses with `6A 80`, writing nothing: a form `01` container naming another card's key; a form `02` container whose path has one bit turned (the last bit of its second node); and the form `01` container sealed with a length one byte short (and the hash of those bytes), which is not a whole container.

```json
{
  "what": "form 01 naming another key, sealed by the form 01 card",
  "container": "4343502d43455254010103983f5f907c7bda7bcc08d9bdf8352103c9c2ab9728daa05de1362293aee257c400e663617368752d636172642d636572742f31206973737565723d746573742d69737375657220666f726d3d63617264206b65793d303339383366356639303763376264613762636330386439626466383335323130336339633261623937323864616130356465313336323239336165653235376334206170706c65743d30323033206275696c643d3734393632323735396236613462343031336330396566663463353334376431393563393734373537653930613764633333396263636362663338326562333420646174653d323032362d31312d313220736368656d653d62697033343000402f3cc066c19208ed62cb00cc53923ceeec3ac0a35ba4d928dc6993648bf9448e1c0ba54d2ef1d0cab3f718f950bc5393dd39118b659da71e2db19854d8cfc9950000",
  "cert_seal": "b0540100220157f75ea5108f139a7b8de15100276913d42b9458ddf1a0cad0401c9fcb6f511c95",
  "sw": "6a80"
}
```

```json
{
  "what": "form 02 of index 0 with the last bit of its second node turned, sealed by card 0 of the tree",
  "container": "4343502d43455254010200000276b4f370e18db293c31aeea24765ec800d16510939ccf009776a1057113e0c88ea67b80bb29043df3097c23e40a25b3896507ede7e5023df3beeaf320708b6573910e1237a23a4bd2f603294b96049c91cd28f33c825b4b2e7717e31e38da46f00ed63617368752d636172642d636572742f31206973737565723d746573742d69737375657220666f726d3d626174636820746f703d3339313065313233376132336134626432663630333239346239363034396339316364323866333363383235623462326537373137653331653338646134366620636f756e743d34206170706c65743d30323033206275696c643d3734393632323735396236613462343031336330396566663463353334376431393563393734373537653930613764633333396263636362663338326562333420646174653d323032362d31312d313220736368656d653d6269703334300040842c10772491159af3f8983e148cc85c0444d429ad33634f9748199c21f35e4acbea39b48f99f11fff1b5c8343fa8bc5b7d8cac57459010b3e11f3e2818e45420000",
  "cert_seal": "b05401002201a02c00826a8a0c4590c62284598f1df88ddbc2913de72a10afd30e5b8090b44b4c",
  "sw": "6a80"
}
```

```json
{
  "what": "the form 01 container sealed one byte short",
  "container": "4343502d434552540101027ac53e398ae03ac0adf29e7903596cf2a142e5b8eb40f401db326df3dab8081e00e663617368752d636172642d636572742f31206973737565723d746573742d69737375657220666f726d3d63617264206b65793d303237616335336533393861653033616330616466323965373930333539366366326131343265356238656234306634303164623332366466336461623830383165206170706c65743d30323033206275696c643d3734393632323735396236613462343031336330396566663463353334376431393563393734373537653930613764633333396263636362663338326562333420646174653d323032362d31312d313220736368656d653d62697033343000406ac5fdb5ce99c40f852681d731777f0aeead7ba2fb7e27a37cf95389c72503297cc0a669acab49d63b9eb7067d4f2eb4b0ce9db2a15da2bcabc3f21e53cb2ea40000",
  "cert_seal": "b05401002201564ba55e6160a46451567b55bbb39be686f30ef9294f36c0ca84c9e097d0e6a73d",
  "sw": "6a80"
}
```
