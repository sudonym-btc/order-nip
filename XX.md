NIP-XX
======

Orders
------------

`draft` `optional`

This NIP defines a protocol for creating, negotiating, funding, acknowledging, settling, cancelling, and reviewing orders against NIP-99 listings on Nostr. It introduces `kind:32122` order events, `kind:32123` payment events, `kind:32124` payment ack events, `kind:32125` payment settlement events, `kind:32126` order cancel events, `kind:32127` payment nack events, `kind:1327` private structured-message rumors, `kind:1328` commit authorization helper events, `kind:1329` temporary trade key (temp-key) authorization helper events, `kind:1330` encrypted marketplace seed events, and `kind:31555` reviews.

Negotiation is private. Signed order and escrow-selection events are sent as child events inside encrypted structured-message rumors and delivered with NIP-59 gift wraps. Public lifecycle events are published to relays chosen by the implementation after funding or cancellation. Payment evidence is not embedded in the order event; it is carried by linked payment lifecycle events.

## Terms

- **Buyer** — Nostr user requesting an order.
- **Seller** — Nostr user who owns the listing receiving the order.
- **Escrow** — Optional service participant that verifies funding and can arbitrate disputes.
- **Trade** — A single order negotiation and lifecycle, identified by a stable `trade` tag (trade ID). Private trade messages use this trade ID as their default `conversation` tag.
- **Order Group** — The public replaceable order-state slot for one immutable role-tagged participant tuple inside a trade.
- **Order Group ID** — A deterministic SHA-256 hash derived from the trade ID and the role-tagged public participant pubkeys in the order. `kind:32122` uses this value as its `d` tag.
- **Listing Anchor** — A NIP-99 listing address in the format `<listing-kind>:<seller-pubkey>:<listing-d-tag>`.
- **Temporary Trade Key (Temp-Key)** — A per-trade Nostr key that can publish buyer-side order events without revealing the buyer's long-lived account key on public relays. The buyer's account identity is bound to this temporary key through encrypted `participant_proof` tags, preserving buyer privacy while still allowing counterparties, escrows, and review verifiers to prove participation when needed.

## Event Kinds

| Kind    | Name                    | Type                      | Description |
| ------- | ----------------------- | ------------------------- | ----------- |
| `32122` | Order                   | Parameterized replaceable | Terms, listing anchor, trade id, and immutable public participant tuple for an order group. |
| `32123` | Payment                 | Parameterized replaceable | Payment proof or payment lock linked to an order. |
| `32124` | Payment Ack             | Parameterized replaceable | Buyer, seller, or escrow acceptance of a payment event. |
| `32125` | Payment Settlement      | Parameterized replaceable | Settlement, release, refund, split, or claim packet linked to a payment event. |
| `32126` | Order Cancel            | Parameterized replaceable | Cancellation request or cancellation notice linked to an order or payment event. |
| `32127` | Payment Nack            | Parameterized replaceable | Buyer, seller, or escrow rejection of a payment event. |
| `1327`  | Structured Message      | Regular private rumor     | Private structured-message rumor whose content is a signed child event JSON string. |
| `1328`  | Commit Authorization    | Regular helper event      | Seller authorization over exact negotiated commit terms. |
| `1329`  | Temp-Key Authorization  | Regular helper event      | Identity-key authorization binding a real participant pubkey to a temporary trade key (temp-key) participant pubkey. |
| `1330`  | Marketplace Seed        | Regular encrypted event   | Encrypted recovery seed used to derive trade ids and temporary trade keys. |
| `31555` | Review                  | Parameterized replaceable | Post-trade marketplace review. |

<!-- Order Transition kind 1326 event kind intentionally commented out. -->

## Order (`kind:32122`)

An order event represents the terms and public participant tuple for one order group. It does not prove funding, acceptance, rejection, settlement, or cancellation by itself. Funding is represented by a linked `kind:32123` Payment event, acceptances by linked `kind:32124` Payment Ack events, rejections by linked `kind:32127` Payment Nack events, settlement by linked `kind:32125` Payment Settlement events, and cancellation by a linked `kind:32126` Order Cancel event.

Negotiation orders are usually sent privately as structured-message child events. A public order becomes meaningful for marketplace availability only when the order group also contains valid payment lifecycle events.

### Tags

```json
[
  ["d", "<order-group-id>"],
  ["trade", "<trade-id>"],
  ["a", "<listing-anchor>"],
  ["p", "<participant-pubkey>", "<relay-hint>", "<role>"],
  ["participant_proof", "<role>", "<participant-pubkey>", "<recipient-pubkey>", "nip44", "<payload-sha256>", "<encrypted-payload>"],
  ["published_at", "<unix-seconds>"]
]
```

| Tag | Required | Description |
| --- | -------- | ----------- |
| `d` | Yes | Order group ID. This is the parameterized replaceable key for a participant's current public order state. It MUST be derived from the `trade` tag and the order's role-tagged participant `p` tags as described in the Order Group section. |
| `trade` | Yes | Trade identifier. Stable across the negotiation and lifecycle. Private trade messages SHOULD repeat this value in a `conversation` tag on the enclosing rumor. |
| `a` | Yes | Listing anchor (`<listing-kind>:<seller-pubkey>:<listing-d-tag>`). The referenced event MUST be a NIP-99 listing. |
| `p` | Yes | Role-tagged participant pubkey. Use `["p", pubkey, relayHint, role]` where role is `buyer`, `seller`, or `escrow`. Every order MUST include exactly one `buyer` and exactly one `seller` participant. Escrow-backed public order groups MUST also include exactly one `escrow` participant. Private negotiation orders before escrow selection MAY omit the `escrow` participant. The event author MUST appear in exactly one participant `p` tag, and that tag's role is the author's order role. Use privacy-preserving temporary trade keys (temp-keys) as participant pubkeys when possible; bind temp-keys to real pubkeys with `participant_proof` tags. The escrow service pubkey MUST be included as the escrow participant for escrow-backed public orders so escrow daemons can subscribe to their own trades. |
| `participant_proof` | No | Encrypted proof binding a temporary trade key (temp-key) participant pubkey to a real identity pubkey. Required when `participantPubkey != identityPubkey`. |
| `published_at` | No | First publication timestamp. Publishers SHOULD preserve this across replacements. |

### Participant Proofs

Participant proofs bind temporary trade key (temp-key) participant pubkeys to real participant identity pubkeys without forcing the buyer to disclose their long-lived account key publicly.

`participant_proof` tag format:

```json
[
  "participant_proof",
  "<role>",
  "<participant-pubkey>",
  "<recipient-pubkey>",
  "nip44",
  "<sha256-of-plaintext-authorization>",
  "<nip44-encrypted-authorization-payload>"
]
```

The plaintext authorization payload is a JSON-encoded signed `kind:1329` Temp-Key Authorization event. It is encrypted with NIP-44 for each trade participant recipient.

#### Temp-Key Authorization (`kind:1329`)

The `kind:1329` event is signed by the real identity pubkey and authorizes one participant pubkey for a trade role.

Tags:

```json
[
  ["a", "<listing-anchor>"],
  ["trade", "<trade-id>"],
  ["d", "<order-group-id>"]
]
```

Content:

```jsonc
{
  "version": 1,
  "role": "buyer",
  "participantPubkey": "<temp-key-or-real-participant-pubkey>"
}
```

A verifier accepts a participant proof only when:

1. the `participant_proof` hash matches the decrypted authorization payload;
2. the authorization event is validly signed by the identity pubkey;
3. the authorization `a` tag matches the listing anchor;
4. the authorization `trade` tag matches the order trade id;
5. the authorization `d` tag matches the order group ID when the order group ID is known;
6. the authorization content role and participant pubkey match the `participant_proof` tag.

### Content

Order content is JSON:

```jsonc
{
  "start": "2026-05-01T00:00:00.000Z",
  "end": "2026-05-05T00:00:00.000Z",
  "quantity": 1,
  "amount": {
    "value": "0.00500000",
    "denomination": "BTC",
    "decimals": 8
  },
  "recipient": "<recipient-or-trade-pubkey>",
  "commitAuthorization": null
}
```

| Field | Type | Required | Description |
| ----- | ---- | -------- | ----------- |
| `start` | string | No | Order start date/time in ISO 8601 UTC. This field MUST be omitted when no start is set. Date-only order UIs SHOULD encode calendar dates as midnight UTC. |
| `end` | string | No | Order end date/time in ISO 8601 UTC. This field MUST be omitted when no end is set. |
| `quantity` | integer | No | Number of units. Default `1`. |
| `amount` | object | No | Agreed or proposed price. `value` is a decimal string, `denomination` is the unit of account, and `decimals` is precision. |
| `recipient` | string | No | Intended payment/trade recipient pubkey. |
| `commitAuthorization` | object | No | Full signed `kind:1328` event JSON authorizing negotiated terms. |

### Commit Terms

The order commitment hash locks exactly these fields:

```json
["start", "end", "quantity", "amount", "recipient"]
```

Implementations MUST canonicalize those fields before hashing. Payment lifecycle events and participant proof tags are not part of the commit terms.

#### Commit Authorization (`kind:1328`)

When the seller accepts off-list or negotiated terms, the seller signs a `kind:1328` event authorizing the order commit hash.

Tags:

```json
[
  ["a", "<listing-anchor>"],
  ["trade", "<trade-id>"],
  ["d", "<order-group-id>"]
]
```

Content:

```jsonc
{
  "version": 1,
  "commitHash": "<order-commit-hash>",
  "role": "seller"
}
```

The order is authorized only if the commit authorization was signed by the listing owner, references the same listing anchor, trade id, and order group ID, and contains the order's commit hash. If the authorization is produced during private negotiation before the escrow participant is chosen, it MAY omit the `d` order group tag; in that case it authorizes only the trade ID and listing anchor and MUST be bound to the final order group by the committed order that embeds it.

## Negotiation Semantics

Negotiation is an append-only private thread of valid order and cancel child
events sharing the same trade id in the `trade` tag. The enclosing
structured-message rumor SHOULD use
`["conversation", "<trade-id>"]` so the buyer and seller can continue the same
private thread before and after escrow selection.

The current negotiation state is the newest valid order child event in the
private thread, unless the newest valid child event is an Order Cancel event for
that thread. Clients SHOULD order private child events the same way
NIP-17 direct-message clients order messages in a chat room after unwrapping and
decrypting them: use the decrypted inner event's `created_at`, not the
randomized seal or gift-wrap `created_at` values. If two valid items have the
same timestamp, clients SHOULD use a deterministic tie-break such as event id.

A valid negotiation item MUST:

1. reference the same listing anchor and trade id;
2. contain exactly one role-tagged buyer `p` tag and exactly one role-tagged
   seller `p` tag;
3. be authored by one of the participant pubkeys listed in the order's `p`
   tags, using that tag's role as the author role;
4. be delivered in the private thread for the resolved participants of that
   order;
5. have valid order content and valid participant proofs when temporary trade
   keys are used.

When the seller responds to an offer, the seller MUST send a signed order child
event containing a signed `commitAuthorization` that authorizes the response
order's commit hash. A seller counteroffer without `commitAuthorization` is only
an unsigned proposal and MUST NOT be treated as accepted.

In the negotiation phase, an accept action is indicated by either:

- the seller sending back a valid signed order containing a valid
  `commitAuthorization`; or
- the buyer executing payment and publishing a public order anchor plus linked
  Payment event.

## Accepted Live Orders

A trade is an accepted live order when one of the following is true after
validating the relevant private thread and public order group:

1. the latest payable terms have a seller-signed `commitAuthorization`;
2. the seller has accepted the linked Payment event with a Payment Ack;
3. the listing has `autoAccept=true` and the buyer has published a Payment event
   with valid payment proof for the full required amount, even without
   explicit seller acknowledgement.

An accepted live order is not necessarily final financial settlement. Escrow
confirmation, dispute resolution, payment release, or cancellation policies may
still apply.

## Private Structured Messages (`kind:1327`)

Private structured trade messages use `kind:1327` as the inner rumor kind. The rumor `content` is the JSON string of a signed child event, usually a `kind:32122` order or `kind:30302` escrow-service selection.

The rumor includes `p` tags for the recipients and SHOULD include `["conversation", "<trade-id>"]` for trade-related messages. The `conversation` tag is the private-message grouping mechanism and remains the stable trade id before and after escrow selection. Local inbox implementations MAY group gift wraps by the sorted set of rumor author plus rumor `p` tags, combined with the `conversation` tag; that local conversation id is not the order group id and MUST NOT be used for order validity. The rumor MAY include `alt` tags. It is sealed and wrapped with NIP-59. The sender broadcasts one `kind:1059` gift wrap for every recipient and one for self.

Private trade DMs MUST be sent between the resolved participant pubkeys of the
order. A resolved participant pubkey is the participant identity pubkey when a
temporary trade key is authorized by `participant_proof`, otherwise it is the
participant order pubkey. Buyer/seller negotiation messages SHOULD include the
resolved buyer and seller participants. If a committed order is disputed, the
participants SHOULD add the escrow service pubkey to the same participant
thread, keep the same `conversation` trade id, and message the buyer, seller,
and escrow together. Implementations SHOULD NOT create an escrow-only side
conversation for disputes about a committed order.

Plain text private messages use standard private message rumor kind `14`.

<!-- Order Transition kind 1326 section intentionally commented out. -->

## Verification

Clients verify orders for two primary reasons:

1. to determine availability for the referenced NIP-99 listing;
2. to verify that reviews are attached to a real trade.

Availability verification is based on public order groups after applying the validity and precedence rules in the Order Group section. Review verification proves that the reviewer participated in a structurally valid order group that reached confirmed commitment.

## Payment Lifecycle Events

The order group lifecycle is represented by events that all carry the same `d`
tag as the order group ID. They MUST also carry the same `trade` and `a` tags as
the order. Implementations SHOULD repeat the role-tagged `p` tags from the order
anchor on every lifecycle event so clients can subscribe by participant pubkey.

After the initial order anchor defines the order group participant tuple, every
payment, payment ack, payment nack, payment settlement, and cancel event for
that order group MUST be authored by one of those participant pubkeys. Clients
MUST ignore lifecycle events whose author is not in the order group's
role-tagged participant set, or whose `d`, `trade`, or `a` tag does not match
the order anchor.

Lifecycle events link to the event they evaluate with `e` tags:

```json
["e", "<event-id>", "<relay-hint>", "<marker>"]
```

Defined markers are:

| Marker | Description |
| ------ | ----------- |
| `order` | Referenced order anchor. Required on Payment events. |
| `payment` | Referenced Payment event. Required on Payment Ack, Payment Nack, and Payment Settlement events. |
| `payment-ack` | Referenced Payment Ack event. Optional on Payment Settlement events. |
| `payment-nack` | Referenced Payment Nack event. Optional on later lifecycle events. |
| `settlement` | Referenced Payment Settlement event. Optional on later lifecycle events. |
| `cancel` | Referenced Order Cancel event. Optional on later lifecycle events. |

### Payment (`kind:32123`)

A Payment event contains the generic payment evidence or payment lock for an
order. It MUST include `["e", "<order-event-id>", "<relay-hint>", "order"]`.

Payment content is JSON:

```jsonc
{
  "proof": {
    "listing": { "...": "NIP-99 listing event JSON" },
    "paymentProof": {
      "method": "evm",
      "params": {
        "txHash": "<evm-transaction-hash>"
      }
    },
    "escrow": {
      "escrowService": { "...": "EscrowService kind:30303 event JSON" },
      "sellerEscrowMethod": { "...": "seller EscrowMethod kind:17388 event JSON" }
    }
  },
  "purpose": "order.payment"
}
```

The `proof.paymentProof` object is keyed by the payment method enum and carries
method-specific `params`. Escrow-specific verification context, when needed, is
attached beside the generic payment proof under `proof.escrow`.

Defined payment methods are `"zap"` for NIP-57 zap receipts and `"evm"` for EVM
transaction proofs. Escrow service types MAY map to payment methods; for
example, escrow service type `"EVM"` uses payment method `"evm"`.

### Zap Proof

```jsonc
{
  "listing": { "...": "NIP-99 listing event JSON" },
  "paymentProof": {
    "method": "zap",
    "params": {
      "receipt": { "...": "zap receipt event JSON" },
      "recipientProfile": { "...": "seller profile metadata event JSON" }
    }
  }
}
```

For zap proofs, `recipientProfile` is required so clients can verify that the
zap receipt LNURL matches the seller's signed current payment address.

### Escrow Proof

```jsonc
{
  "listing": { "...": "NIP-99 listing event JSON" },
  "paymentProof": {
    "method": "evm",
    "params": {
      "txHash": "<evm-transaction-hash>"
    }
  },
  "escrow": {
    "escrowService": { "...": "EscrowService kind:30303 event JSON" },
    "sellerEscrowMethod": { "...": "seller EscrowMethod kind:17388 event JSON" }
  }
}
```

For EVM escrow-backed orders, `paymentProof.method` MUST be `"evm"` and
`paymentProof.params.txHash` is the transaction hash to verify. The `escrow`
context is required only to interpret that EVM payment proof as satisfying a
selected escrow method. `sellerEscrowMethod` MUST include the seller's `["i",
"evm:address:<address>", "eip191:<signature>"]` ownership proof. See the Escrow
Services NIP for the exact proof payload and escrow verification requirements.
The on-chain escrow `tradeId` proved by the transaction MUST equal the order
group ID in the payment event's `d` tag. EVM escrow proof now requires on-chain
validation because the order event itself no longer carries authoritative
payment state; this lets lightweight clients subscribe to order groups cheaply
while payment-aware clients or escrow daemons validate the actual transaction.

### Payment Ack (`kind:32124`)

A Payment Ack event records a participant's acceptance of a Payment event. It
MUST include `["e", "<payment-event-id>", "<relay-hint>", "payment"]`.

Content:

```jsonc
{
  "status": "accepted",
  "message": "Payment proof verified"
}
```

`status` MUST be `accepted`. Buyer and seller acknowledgements are used to
represent both parties' agreement that the payment event funds the order.
Escrow services MAY also publish Payment Ack events when they validate a payment
proof.

### Payment Nack (`kind:32127`)

A Payment Nack event records a participant's rejection of a Payment event. It
MUST include `["e", "<payment-event-id>", "<relay-hint>", "payment"]`.

Content:

```jsonc
{
  "status": "rejected",
  "message": "Payment proof did not satisfy the selected policy"
}
```

`status` MUST be `rejected`. Payment Nack events do not cancel an order by
themselves; they explain why the referenced payment proof should not be treated
as committed. If the trade is cancelled, an Order Cancel event is still
required.

### Payment Settlement (`kind:32125`)

A Payment Settlement event records the post-deposit outcome for escrowed
payments and bids. It MUST include `["e", "<payment-event-id>", "<relay-hint>",
"payment"]` and SHOULD include `payment-ack` references for the acknowledgements
or authorizations it consumes.

Content:

```jsonc
{
  "method": "cashu",
  "action": "split",
  "inputs": [
    { "y": "<cashu-proof-y>", "amount": "1000" }
  ],
  "outputs": [
    { "role": "buyer", "pubkey": "<buyer-pubkey>", "amount": "500" },
    { "role": "seller", "pubkey": "<seller-pubkey>", "amount": "500" }
  ],
  "data": {
    "witnesses": { "...": "method-specific settlement data" }
  }
}
```

`method` and `action` are extensible. A Payment Settlement event is atomic: if
it exists, it MUST contain proof that settlement already happened or enough
witness data for the relevant participants to claim their settlement outputs
without any further event. For Cashu, settlement events can carry a signed claim
packet, deterministic restore metadata, or a fully authorized collaborative swap
template. For EVM, settlement events usually reference on-chain settlement
transactions and are advisory because refunds, releases, and losing bids can be
independently verified on chain.

### Order Cancel (`kind:32126`)

An Order Cancel event records cancellation intent for an order group. It MUST
include either an `order` reference or a `payment` reference.

Content:

```jsonc
{
  "reason": "cancelled by buyer"
}
```

Cancellation does not prove that funds moved. If a funded order is cancelled,
the refund/release/split MUST be represented by a Payment Settlement event.

### Seller-Published Proofless Orders

Seller proofless orders no longer rely on `order.pubkey == listing.pubkey`.
Clients MUST use the role-tagged seller participant and, when a temporary trade
key is used, the encrypted `participant_proof` authorization to reconcile the
seller participant with the listing owner. A seller-authored order can block
inventory only after that role proof is valid or application policy explicitly
allows an unproven seller reservation.

## Order Group

All public order lifecycle events with the same order group ID form an order
group. The order group ID is the `d` tag on `kind:32122`, `kind:32123`,
`kind:32124`, `kind:32125`, `kind:32126`, and `kind:32127`. It is derived from
the stable trade ID plus the public, non-decrypted, role-tagged participant
pubkeys in the order anchor's `p` tags.

The participant tuple is role-aware. It is not sufficient to hash only a sorted
list of pubkeys because the same pubkeys in different roles would otherwise
produce the same replaceable event slot.

The order group ID MUST be derived from the trade ID and canonical participant
entries:

```text
sha256(json([
  tradeId,
  sorted([
    ["buyer", "<buyer-participant-pubkey>"],
    ["seller", "<seller-participant-pubkey>"],
    ["escrow", "<escrow-participant-pubkey>"]
  ])
]))
```

`json(...)` means deterministic JSON with no insignificant whitespace and stable
array ordering. The participant entries are sorted by role name and then pubkey.
The resulting 32-byte hash is lowercase hex. Implementations MAY use a different
canonical binary encoding if it is explicitly profiled by a future version of
this NIP, but all participants in a marketplace MUST use the same encoding to
produce the same `d` tag.

Private negotiation orders before escrow selection use the same derivation with
the available buyer and seller participant entries. When an escrow is selected,
the escrow-backed order group has a different order group ID because the
participant tuple now includes the `escrow` entry. The `trade` tag remains the
same, so private messages can continue in the same trade conversation.

Public escrow-backed order groups MUST include exactly one `buyer`, exactly one
`seller`, and exactly one `escrow` participant `p` tag on the order anchor.
Public non-escrow order groups MUST include exactly one `buyer` and exactly one
`seller` participant `p` tag. Events with missing required roles, duplicate
roles, or a computed order group ID that does not equal the `d` tag are invalid
for the normal order group pipeline.

The author role is determined only from the author pubkey's matching
role-tagged `p` tag. Clients MUST NOT infer the author role from the listing
anchor or from an undecrypted `participant_proof`. The listing anchor remains a
validation constraint: the seller participant SHOULD be the listing owner, or a
temporary trade key whose real identity is authorized by a valid
`participant_proof` signed by the listing owner. Implementations MAY reject
public order groups whose seller participant cannot be reconciled with the
listing owner.

Role-marked `p` tags define the public participant tuple and routing hints. They
do not reveal real identities when temporary trade keys are used. Participant
pubkeys SHOULD be temporary trade keys when privacy can be preserved; real
pubkeys are carried in encrypted `participant_proof` payloads when needed.

Clients MUST ignore lifecycle events from outsiders whose author pubkey is not
one of the role-tagged participant pubkeys in the initial order anchor. Clients
MUST also ignore events whose listing anchor, trade ID, or order group ID does
not match the order group they are reducing. Such events form another order
group if their computed order group ID differs; they do not create ambiguity
inside an existing order group.

### Group Reduction

Clients query and subscribe to all public order group event kinds:

```json
{
  "kinds": [32122, 32123, 32124, 32125, 32126, 32127],
  "#d": ["<order-group-id>"]
}
```

When multiple events of the same role and kind are available, clients SHOULD use
the latest valid event by highest `created_at`, with lexicographic event id as a
deterministic tie-break.

The group stage is derived from valid linked lifecycle events:

- `cancel` if a valid Order Cancel event exists and applies to the order or payment;
- `settled` if a valid Payment Settlement event exists and applies to the payment;
- `commit` if the latest Payment event validates by the relevant payment policy, or if both buyer and seller have accepted it with Payment Ack events;
- `negotiate` otherwise.

A group is **confirmed committed** if any of the following hold after validation:

- the payment event validates by the relevant payment policy;
- both buyer and seller have accepted the payment event;
- a valid payment settlement exists.

Later cancellation MUST NOT by itself erase the fact that a trade reached
confirmed commitment. Cancellation changes marketplace availability only when it
is valid for the current order group and application policy says the order is
still cancellable; funded cancellations still require a Payment Settlement event
to explain the refund/release/split.

## Review (`kind:31555`)

After a completed or confirmed committed trade, a participant MAY publish a review. Reviews use the GammaMarkets product review shape: the review body is raw human-readable text in `content`, and the primary rating is a `rating` tag with a normalized score from 0 to 1 and the `thumb` label.

### Tags

```json
[
  ["d", "<order-group-id>"],
  ["trade", "<trade-id>"],
  ["a", "<listing-anchor>"],
  ["rating", "<0-to-1-score>", "thumb"],
  ["r", "<order-anchor>"],
  ["review_proof", "<role>", "<participant-or-temp-key-pubkey>", "<plaintext signed kind:1329 event JSON>"]
]
```

The `d` tag contains the order group ID. Reviews are parameterized replaceable events, so a participant replaces their review for the same order group by publishing a newer review with the same `d` tag. The `trade` tag contains the stable private trade id for clients that want to correlate a review with a private trade thread.

The `a` tag contains the listing anchor (`<kind>:<pubkey>:<d-tag>`) for the listing being reviewed.

The `rating` tag contains the primary rating. The third element MUST be `thumb`. The score is normalized from 0 to 1, where 0 is negative and 1 is positive. Five-star clients SHOULD map stars to this value by dividing by 5.

The `r` tag contains an order anchor (`<kind>:<pubkey>:<d-tag>`) linking the review to a specific trade participant event.

The `review_proof` tag is Hostr-specific and MAY be omitted when the review is signed directly by an order participant pubkey. Clients SHOULD include it when the review is signed by an identity key but the public order participant is a temporary trade key. Generic Gamma-compatible clients can ignore this tag.

### Content

```text
Clear terms, smooth payment, and accurate listing.
```

The content field is raw review text. Structured data belongs in tags.

### Review Validity

A review is valid only if:

1. the `d` tag identifies an order group and the `a` tag references an existing listing;
2. the primary `rating` tag is present as `["rating", "<0-to-1-score>", "thumb"]`;
3. the referenced order group validates structurally;
4. the order group is confirmed committed;
5. either:
   - the review pubkey appears as an order participant pubkey in the referenced order group; or
   - the `review_proof` tag's authorization payload hashes to a `participant_proof` payload hash in the referenced order group, and the decoded `kind:1329` authorization event is valid and matches the claimed role, participant pubkey, listing anchor, trade id, and order group ID when present.

This NIP does not define a canonical on-chain or block-time proof that the review was written after the order ended. Clients MAY additionally require the order end time to be in the past or a terminal payment event to exist, but those timing rules are application policy unless standardized separately.

## Protocol Flow

### 1. Buyer Initiates Negotiation

1. Buyer discovers a NIP-99 listing.
2. Buyer allocates a trade id and temporary trade key (temp-key). Deterministic derivation is optional and described in the addendum below.
3. Buyer creates a `kind:32122` order signed by the temporary trade key (temp-key), carrying `["trade", "<trade-id>"]` and a `d` tag derived from the buyer/seller participant tuple.
4. Buyer includes role-marked `buyer` and `seller` participant `p` tags and encrypted `participant_proof` tags as needed.
5. Buyer sends the order as a child event inside a private `kind:1327` rumor tagged `["conversation", "<trade-id>"]`, delivered with NIP-59 gift wraps to the resolved seller participant and self.

### 2. Negotiation

6. Seller reviews the request. If counter-offering, seller creates a new order with changed terms and embeds a signed `kind:1328` commit authorization for those terms.
7. If the seller accepts the current terms, the seller replies with an order for those terms and embeds a signed `kind:1328` commit authorization.
8. Counteroffers continue privately until terms are accepted or cancelled.

### 3. Escrow Selection and Payment

9. If escrow-backed, buyer selects an escrow service, computes the escrow-backed order group ID from the same trade id plus buyer/seller/escrow participant tuple, and sends a private `kind:30302` Escrow Service Selected child event inside a `kind:1327` rumor tagged `["conversation", "<trade-id>"]`.
10. Buyer funds escrow directly or via a Lightning-to-on-chain swap.

### 4. Commitment

11. Once payment proof exists, the buyer or temporary trade key (temp-key) publishes a public `kind:32122` order anchor with `d=<order-group-id>`, `trade=<trade-id>`, and canonical participant `p` tags for buyer, seller, and escrow when escrow-backed.
12. Buyer publishes a linked `kind:32123` Payment event with `d=<order-group-id>` and `["e", "<order-event-id>", "<relay-hint>", "order"]`.
<!-- Order Transition publication step intentionally commented out. -->
13. Buyer and seller SHOULD publish `kind:32124` Payment Ack events that reference the Payment event. Escrow MAY publish a Payment Ack after validation or a `kind:32127` Payment Nack when the proof is inadequate.
14. Release, refund, split, or claim data is published as a `kind:32125` Payment Settlement event that references the Payment event and any consumed Payment Ack events.

### 5. Cancellation

Either party MAY cancel a private negotiation with a private Order Cancel child
event for the same trade id and participant tuple. Public cancellation publishes
a `kind:32126` Order Cancel event for the same order group ID. A funded
cancellation MUST be followed by a Payment Settlement event that explains the
refund, release, or split.

## Availability

Clients MUST only consider public order groups after applying the Order Group validity ordering when computing listing availability. Negotiation orders are exchanged only between buyer and seller via gift-wrapped private structured messages, so they are not part of public listing availability calculations.

Order clients SHOULD verify that the referenced listing is an active NIP-99 listing before displaying or accepting an order.

## Addendum: How to Choose Temporary Keys

Temporary trade keys (temp-keys) exist to preserve privacy by separating a buyer's public account identity from public order events. This NIP does not require deterministic values for trade ids or temporary keys. A client MAY generate a fresh random trade id and random temporary key for each trade and store the resulting private state locally.

A Nostr user MAY instead publish encrypted SEED events (`kind:1330`) globally. A SEED event content is a NIP-44 encrypted payload from the user's identity key to itself, for example:

```jsonc
{
  "v": 1,
  "seed": "<32-byte-hex-seed>"
}
```

SEED events MUST use a regular, non-replaceable event kind. Clients MUST NOT use a replaceable event kind for seed backups, because replacing a seed can make older deterministic trade ids and temporary trade keys unrecoverable. If a client accidentally publishes another seed, the older event remains available on relays because the kind is not replaceable.

On startup, a client that uses deterministic marketplace seeds SHOULD query the
user's `kind:1330` events with `limit: 1` and use only the first seed event it
receives. If that event cannot be decrypted or parsed, the default behavior is a
startup/recovery failure rather than scanning additional seed events. This NIP
does not define default multi-seed selection, rotation, or merging behavior.

Clients that can decrypt a SEED event can derive arbitrary trade ids and temporary trade keys from the seed, avoiding dependence on one device's local storage of encrypted gift wraps for each individual trade. This is a convenience and recovery mechanism, not a consensus requirement: clients MUST accept valid orders and participant proofs regardless of whether their trade ids and temporary keys were random, locally stored, or derived from a published SEED event.

Clients that implement deterministic derivation SHOULD domain-separate trade ids from temporary keys and SHOULD include an account-controlled index or nonce so the same seed never produces the same public key for two unrelated trades.

## Related NIPs

- [NIP-01](01.md) — Event structure and parameterized replaceable events.
- [NIP-99](99.md) — Classified listings.
- [NIP-17](17.md) — Private message rumor kind `14`.
- [NIP-19](19.md) — `naddr` encoding for anchors.
- [NIP-44](44.md) — Encryption scheme.
- [NIP-57](57.md) — Zaps.
- [NIP-59](59.md) — Gift wrap.
