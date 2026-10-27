# Assisted settlement design proposal

Status: design review before wire freeze. This proposal describes the next
capability; existing 2.1 cards retain their current wire and recovery path.
Production APDU assignments, version numbers, storage limits and shared vectors
follow resource measurements and review. Implementation tracking:
[authenticated export](https://github.com/cashucard/nut/issues/5),
[local mint proof](https://github.com/cashucard/cashucard/pull/188),
[merchant durability](https://github.com/lnflash/flash-pos/pull/96),
[merchant recovery](https://github.com/lnflash/flash-pos/pull/97), and
[resource probe](https://github.com/cashucard/cashucard/pull/189).

## Purpose and initial profile

An online holder wallet relays the offline till's original signed swap and
verifies recoverable merchant outputs. It can then authorize reclamation of that
payment's spent slots while the till remains offline. The till later restores
and unblinds its own outputs using its retained private preparation. A live till
acknowledgement would prevent this use case.

The initial profile is sat, format-4 SIG_ALL, date zero, exact payments with no
card change or refund path, frozen request metadata, and independently accessible
swap, state and restore endpoints. Restore must retain matching blinded DLEQ.
Unsupported profiles are refused before consumption. Merchant intent and private
openings must be durable before SIGN, with a verified witness durable before
acceptance. Merchant inventory must survive repeated recovery and payout replay.
Merchant device loss requires its own secret-backup policy.

## Authentication and keyset provenance

[NUT-11 SIG_ALL](https://github.com/cashubtc/nuts/blob/main/11.md)
binds ordered input secret/C values and output amount/B_ values. Mint URL, unit,
input amount/keyset ID, output keyset ID and output roles need a separate
commitment. The card signs an authenticated descriptor of the complete packet
under a distinct domain. The card derives its configuration and input fields
from its own state and validates the remaining metadata before signing.

Current slots store 8-byte keyset prefixes, while modern full IDs are 33 bytes.
The proposed extension keeps a bounded immutable full-ID registry established
during holder-authorized loading. Each proof slot references its exact registry
entry. A merchant may select output IDs from registered entries; it cannot add or
replace a mapping during payment. Registry entries referenced by a proof or
pending record remain pinned. Loading, mint lookup and DLEQ verification must
establish the full mapping before funding is committed. An ambiguous prefix
match blocks loading or payment.

The registry binds the descriptor to card state. Wallets and tills still verify
mint public keys, amounts, input DLEQ, fees and supported keyset policy. The card
cannot discover current mint activation or fee policy while offline. A later
inactive keyset, expiry or incompatible policy stops reconciliation with the
original request retained. Generating replacement output IDs under the old
witness is outside this profile.

Three keys have separate roles:

- The card's secp256k1 key signs the payment and authenticated descriptor.
- The holder's owner key authorizes packet reads and retirement using the
  existing owner-proof model. The ops service holds no such authority.
- Merchant secrets and blinding factors stay in the till's preparation. They
  are excluded from the exported packet.

## Canonical packet candidate

This is the proposed canonical order for the codec prototype. Integers are
unsigned big endian. Byte strings have their stated length. Count fields are
uint8; bounded counts and page sizes remain measurement gates. Amounts are
uint32, matching the card's current amount domain; reject sum overflow.

| Field                 | Candidate encoding and validation                                                                                                                      |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Domain and format     | ASCII `CCP-AS`, uint8 packet format 1                                                                                                                  |
| Installation          | Card public key (33 bytes), installation nonce (32 bytes), freshly generated on installation                                                           |
| Record identity       | uint64 monotonic payment ID, uint8 outbox index, uint32 outbox generation                                                                              |
| Profile and mint      | uint8 exact-payment profile, uint16 byte length and exact configured mint URL bytes, existing sat unit encoding                                        |
| Inputs                | Count, then ordered entries: uint8 slot, uint32 slot generation, uint32 amount, uint8 full-ID length (8 or 33), full ID, nonce (32), compressed C (33) |
| Outputs               | Count, then ordered entries: uint32 amount, uint8 full-ID length, full ID, compressed B_ (33); every output belongs to the merchant in this profile    |
| Payment authorization | Original SIG_ALL witness (64 bytes)                                                                                                                    |

The configured mint URL is validated at configuration time and retained byte for
byte. Production supports HTTPS base URLs without credentials, query or fragment.
The local test profile permits loopback HTTP. Input secrets are reconstructed
with the existing exact format-4 date-zero serialization, card key and nonce.
A parser refuses alternative secret forms, duplicate input slots/nonces, duplicate
output B_ values, invalid points, trailing bytes and unsupported fields.

Let `body` be these canonical bytes including the payment witness. The packet
ID is `SHA256(body)`. The descriptor signs
`SHA256(SHA256(tag) || SHA256(tag) || body)`, where the proposed UTF-8 tag is
`CashuCard/assisted-settlement/v1`. Its BIP-340 signature (64 bytes) follows the
body. Verify both signatures against the authenticated card key. The distinct
tag separates descriptor signatures from payments and ordinary AUTH.

On-card storage can retain slot references, registry references and compact
outputs. Export materializes the canonical packet while streaming through
existing scratch space. Cached packets retain the complete authenticated context.
Final bytes and cross-implementation vectors are required before interoperability
claims or production capability advertisement.

## Publication and retirement

Each bounded record follows `FREE → STAGING → COMMITTED → CLEARING → FREE`.
Reserve all required space before signing. A terminal omitting required journal
metadata receives a refusal before any proof is consumed. Pending records cannot
be evicted to admit another payment. Abandoned STAGING records are reclaimable.

Stage the complete payload and both signatures outside the transaction. Publish
COMMITTED with the existing irreversible consumption/burn marker in one atomic
boundary. Before commit, staging signatures remain inaccessible through every
export and last-signature path. After commit, a power loss leaves a complete
exportable record and resumable burn bookkeeping. Resume pending burns before
exposing balances, accepting another payment or exporting a packet. References
pin their proof slots and keyset mappings; proof data cannot be zeroed or reused
while required by any unresolved record.

The holder wallet persists the packet and settlement evidence before requesting
retirement. Its owner proof binds the command label/nonce, installation identity,
payment ID, record generation, packet ID and input slot generations. A stale or
wrong-card request cannot clear a replacement record. Retirement clears only the
selected verified packet's slots. CLEARING retains sufficient identity and
payload to resume after a cut. Slot reuse follows completed cleanup, with a new
generation. Counters cannot wrap silently.

Bounded completion evidence lets recent repeated requests report completion.
After that evidence expires, the card returns stale and the wallet re-reads actual
state. It cannot infer free capacity from an old acknowledgement. Configuration
or ownership changes that invalidate pending context are blocked until retirement.

The owner-authorized acknowledgement trusts the wallet's mint verification over
HTTPS. The card authenticates the packet and owner command; it does not verify
ordinary NUT-07/NUT-09 network responses as mint-signed receipts.

## Helper and merchant convergence

1. Read/authenticate the same card, export and persist complete packets.
2. Check every original input and restore exact original outputs. Verify exact
   coverage, amount, keyset and blinded DLEQ, matching response pairs by B_.
3. Submit the frozen original swap only when the input state permits it.
   PENDING waits. Lost replies and duplicate races return to restore. Mixed,
   unavailable, contradictory or incomplete evidence retains the packet.
4. Once original outputs are issued and recoverable, the holder wallet may
   authorize retirement without waiting for the till to reconnect.
5. The till restores its original outputs, unblinds with its saved openings,
   and inserts each merchant proof once. Already-spent restored outputs require
   durable payout accounting; unknown consumption remains unresolved. NUT-09
   issuance alone cannot create another merchant credit.

For top-up, plan capacity from currently free slots plus verified eligible
retirements. Persist paid funding and its resulting proofs. On the second NFC
session, revalidate installation, packet and slot generations, retire, confirm
actual space, then load. Changed state preserves the paid top-up journal for
later loading. The first inspection reserves no capacity.

## Remaining gates

The physical resource spike measures installation, available EEPROM/RAM,
publication overhead, the second signature and NFC latency before selecting
record, registry or output bounds. Full power-loss, stale-ACK, abandoned-staging,
slot-reuse, mint-rotation and helper/till-race cases follow the frozen contract.
Existing cards cannot export complete historical swaps. Migration requires
settlement, returned change, an empty reconciled card and an approved fresh
installation. The merchant recovery release precedes relay enablement.
