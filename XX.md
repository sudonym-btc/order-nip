NIP-XX
======

Orders
------------

`draft` `optional`

This NIP defines a protocol for creating, negotiating, committing, cancelling, and reviewing orders against NIP-99 listings on Nostr. It introduces `kind:32122` order events, `kind:1327` private structured-message rumors, `kind:1328` commit authorization helper events, `kind:1329` temporary trade key (temp-key) authorization helper events, and `kind:31555` reviews.

Negotiation is private. Signed order and escrow-selection events are sent as child events inside encrypted structured-message rumors and delivered with NIP-59 gift wraps. Public committed/cancelled order snapshots are published to relays chosen by the implementation.

## Terms

- **Buyer** — Nostr user requesting an order.
- **Seller** — Nostr user who owns the listing receiving the order.
- **Escrow** — Optional service participant that verifies funding and can arbitrate disputes.
- **Trade** — A single order negotiation and lifecycle, identified by a stable `d` tag (trade ID).
- **Listing Anchor** — A NIP-99 listing address in the format `<listing-kind>:<seller-pubkey>:<listing-d-tag>`.
- **Temporary Trade Key (Temp-Key)** — A per-trade Nostr key that can publish buyer-side order events without revealing the buyer's long-lived account key on public relays. The buyer's account identity is bound to this temporary key through encrypted `participant_proof` tags, preserving buyer privacy while still allowing counterparties, escrows, and review verifiers to prove participation when needed.

## Event Kinds

| Kind    | Name                    | Type                      | Description |
| ------- | ----------------------- | ------------------------- | ----------- |
| `32122` | Order             | Parameterized replaceable | A participant's order proposal, commitment, or cancellation. |
| `1327`  | Structured Message      | Regular private rumor     | Private structured-message rumor whose content is a signed child event JSON string. |
| `1328`  | Commit Authorization    | Regular helper event      | Seller authorization over exact negotiated commit terms. |
| `1329`  | Temp-Key Authorization  | Regular helper event      | Identity-key authorization binding a real participant pubkey to a temporary trade key (temp-key) participant pubkey. |
| `31555` | Review                  | Parameterized replaceable | Post-trade marketplace review. |

<!-- Order Transition kind 1326 event kind intentionally commented out. -->

## Order (`kind:32122`)

An order event represents one participant's current position in a trade. Multiple participants (buyer, seller, and optionally escrow) may each publish order events sharing the same `d` tag.

### Stages

| Stage       | Description |
| ----------- | ----------- |
| `negotiate` | Mutable private proposal or counteroffer. Negotiate-stage orders are sent only between buyer and seller as private structured-message child events delivered with NIP-59 gift wraps, so they are not observed by the wider relay network. |
| `commit`    | Order commitment. Public commit-stage events affect listing availability unless cancelled. |
| `cancel`    | Cancellation of a negotiation or prior commitment. |

### Tags

```json
[
  ["d", "<trade-id>"],
  ["a", "<listing-anchor>"],
  ["p", "<participant-pubkey>", "<relay-hint>", "<role>"],
  ["participant_proof", "<role>", "<participant-pubkey>", "<recipient-pubkey>", "nip44", "<payload-sha256>", "<encrypted-payload>"],
  ["published_at", "<unix-seconds>"]
]
```

| Tag | Required | Description |
| --- | -------- | ----------- |
| `d` | Yes | Trade identifier. Stable across all events in the trade. Private trade messages SHOULD repeat this value in a `conversation` tag on the enclosing rumor. |
| `a` | Yes | Listing anchor (`<listing-kind>:<seller-pubkey>:<listing-d-tag>`). The referenced event MUST be a NIP-99 listing. |
| `p` | Yes | Participant pubkey. When a role is known, use `["p", pubkey, relayHint, role]` where role is `seller`, `buyer`, or `escrow`. |
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
  ["d", "<trade-id>"]
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
4. the authorization `d` tag matches the order trade id;
5. the authorization content role and participant pubkey match the `participant_proof` tag.

### Content

Order content is JSON:

```jsonc
{
  "start": "2026-05-01T00:00:00.000Z",
  "end": "2026-05-05T00:00:00.000Z",
  "stage": "negotiate",
  "quantity": 1,
  "amount": {
    "value": "0.00500000",
    "denomination": "BTC",
    "decimals": 8
  },
  "recipient": "<recipient-or-trade-pubkey>",
  "proof": null,
  "commitAuthorization": null
}
```

| Field | Type | Required | Description |
| ----- | ---- | -------- | ----------- |
| `stage` | string | Yes | One of `negotiate`, `commit`, `cancel`. |
| `start` | string | No | Order start date/time in ISO 8601 UTC. Date-only order UIs SHOULD encode calendar dates as midnight UTC. |
| `end` | string | No | Order end date/time in ISO 8601 UTC. |
| `quantity` | integer | No | Number of units. Default `1`. |
| `amount` | object | No | Agreed or proposed price. `value` is a decimal string, `denomination` is the unit of account, and `decimals` is precision. |
| `recipient` | string | No | Intended payment/trade recipient pubkey. |
| `proof` | object | No | Payment proof. Present on committed self-signed or escrow-backed orders. |
| `commitAuthorization` | object | No | Full signed `kind:1328` event JSON authorizing negotiated terms. |

### Commit Terms

The order commitment hash locks exactly these fields:

```json
["start", "end", "quantity", "amount", "recipient"]
```

Implementations MUST canonicalize those fields before hashing. `stage`, `proof`, and participant proof tags are not part of the commit terms.

#### Commit Authorization (`kind:1328`)

When the seller accepts off-list or negotiated terms, the seller signs a `kind:1328` event authorizing the order commit hash.

Tags:

```json
[
  ["a", "<listing-anchor>"],
  ["d", "<trade-id>"]
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

The order is authorized only if the commit authorization was signed by the listing owner, references the same listing anchor and trade id, and contains the order's commit hash.

## Negotiation Semantics

Negotiation is an append-only private thread of valid `stage=negotiate` and
private `stage=cancel` order child events sharing the same trade id.

The current negotiation state is the newest valid order child event in the
private thread. If that item has `stage=cancel`, the private negotiation is
cancelled; otherwise the current offer is the newest valid `stage=negotiate`
order child event. Clients SHOULD order private order child events the same way
NIP-17 direct-message clients order messages in a chat room after unwrapping and
decrypting them: use the decrypted inner event's `created_at`, not the
randomized seal or gift-wrap `created_at` values. If two valid items have the
same timestamp, clients SHOULD use a deterministic tie-break such as event id.

A valid negotiation item MUST:

1. reference the same listing anchor and trade id;
2. be authored by the buyer participant, seller participant, or a temporary
   trade key authorized by a valid `participant_proof`;
3. be delivered in the private thread for the resolved participants of that
   order;
4. have valid order content and valid participant proofs when temporary trade
   keys are used.

When the seller responds to an offer, the seller MUST send a signed
`stage=negotiate` order child event containing a signed `commitAuthorization`
that authorizes the response order's commit hash. A seller counteroffer without
`commitAuthorization` is only an unsigned proposal and MUST NOT be treated as
accepted.

In the negotiation phase, an accept action is indicated by either:

- the seller sending back a valid signed `stage=negotiate` order containing a
  valid `commitAuthorization`; or
- the buyer executing payment and publishing a public `stage=commit` order with
  payment proof.

## Accepted Live Orders

A trade is an accepted live order when one of the following is true after
validating the relevant private thread and public order group:

1. the latest payable terms have a seller-signed `commitAuthorization`;
2. the seller has published a counterpart `stage=commit` order for the same
   trade id and listing anchor;
3. the listing has `autoAccept=true` and the buyer has published a `stage=commit`
   order with valid payment proof for the full required amount, even without
   explicit seller acknowledgement.

An accepted live order is not necessarily final financial settlement. Escrow
confirmation, dispute resolution, payment release, or cancellation policies may
still apply.

## Private Structured Messages (`kind:1327`)

Private structured trade messages use `kind:1327` as the inner rumor kind. The rumor `content` is the JSON string of a signed child event, usually a `kind:32122` order or `kind:30302` escrow-service selection.

The rumor includes `p` tags for the recipients and SHOULD include `["conversation", "<trade-id>"]` for trade-related messages. The `conversation` tag is the private-message grouping mechanism; order and escrow-selection child events do not carry their own grouping tag. The rumor MAY include `alt` tags. It is sealed and wrapped with NIP-59. The sender broadcasts one `kind:1059` gift wrap for every recipient and one for self.

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

## Payment Proof

A `commit` order SHOULD include a `proof` object unless it is a seller-published blocked order.

The `proof.paymentProof` object is the generic payment evidence. It is keyed by
the payment method enum and carries method-specific `params`. Escrow-specific
verification context, when needed, is attached beside the generic payment proof
under `proof.escrow`.

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
    "escrowService": "<JSON string of the EscrowService kind:30303 event>",
    "sellerEscrowMethod": "<JSON string of the seller's EscrowMethod kind:17388 event>"
  }
}
```

For EVM escrow-backed orders, `paymentProof.method` MUST be `"evm"` and
`paymentProof.params.txHash` is the transaction hash to verify. The `escrow`
context is required only to interpret that EVM payment proof as satisfying a
selected escrow method. `sellerEscrowMethod` MUST include the seller's `["i",
"evm:address:<address>", "eip191:<signature>"]` ownership proof. See the Escrow
Services NIP for the exact proof payload and escrow verification requirements.

### Seller-Published Orders

The listing owner MAY publish `commit` orders without a payment proof, for example to block dates. Clients MAY accept proofless orders when `order.pubkey == listing.pubkey`.

## Order Group

All order events with the same trade id and listing anchor form an order group. Role-marked `p` tags are routing and discovery hints; they MUST NOT by themselves add authority to the group. Clients establish participants from signed order authors that satisfy the role rules below, the listing owner derived from the listing anchor, valid participant proofs, valid escrow proofs, and valid escrow-authored orders.

Roles are determined by:

| Role | Determination |
| ---- | ------------- |
| Seller | `pubkey == getPubKeyFromAnchor(listingAnchor)`. A `p` tag role of `seller` is only a hint unless it identifies the listing owner. |
| Buyer | A participant with role `buyer`, including a temporary trade key (temp-key) participant resolved through a valid `participant_proof`. |
| Escrow | A participant with role `escrow` only when it is the pubkey of the escrow service in a valid escrow proof or the author of an escrow-authored order for that trade. |

The group id SHOULD be derived from the listing anchor, trade id, and sorted established participant set. Clients MUST ignore role-marked `p` tags and order events from outsiders that cannot be tied to the listing owner, a valid participant proof, a valid escrow proof, or a valid escrow-authored order.

### Group Validity Ordering

The validity of an order group is ordered by participant authority:

1. buyer order with attached proof;
2. escrow order for the same trade id, which can confirm with `stage=commit` or override the buyer proof with `stage=cancel`;
3. seller order for the same trade id, which overrides buyer and escrow orders.

Clients SHOULD first evaluate the buyer's latest public order and attached proof. A buyer `commit` can make a listing appear tentatively unavailable only if its payment proof validates or is confirmed by an escrow or seller order. An escrow SHOULD immediately publish either `stage=commit` or `stage=cancel` for the same trade id after validating or rejecting a buyer payment proof. This prevents ordinary clients from having to validate against the payment chain for hundreds of public orders on a listing.

Seller orders have highest precedence for listing availability because the seller owns the referenced listing. A seller `commit` confirms the trade or blocks the listed inventory even without buyer proof. A seller `cancel` invalidates buyer or escrow claims for availability purposes, while financial disputes after escrow commitment remain governed by escrow settlement, claim, release, or arbitration rules.

The group stage is derived after applying that ordering:

- `cancel` if the highest-precedence applicable order is cancelled;
- `commit` if the highest-precedence applicable order is committed;
- `negotiate` otherwise.

A group is **confirmed committed** if any of the following hold after validation:

- the seller has a `commit` order;
- the escrow has a `commit` order;
- the buyer's escrow-backed payment proof validates on-chain.

Later buyer or seller cancellation MUST NOT by itself erase the fact that a trade reached confirmed commitment.

## Review (`kind:31555`)

After a completed or confirmed committed trade, a participant MAY publish a review. Reviews use the GammaMarkets product review shape: the review body is raw human-readable text in `content`, and the primary rating is a `rating` tag with a normalized score from 0 to 1 and the `thumb` label.

### Tags

```json
[
  ["d", "<trade-id>"],
  ["a", "<listing-anchor>"],
  ["rating", "<0-to-1-score>", "thumb"],
  ["r", "<order-anchor>"],
  ["review_proof", "<role>", "<participant-or-temp-key-pubkey>", "<plaintext signed kind:1329 event JSON>"]
]
```

The `d` tag contains the order trade id. Reviews are parameterized replaceable events, so a participant replaces their review for the same trade by publishing a newer review with the same `d` tag.

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

1. the `d` tag identifies a trade and the `a` tag references an existing listing;
2. the primary `rating` tag is present as `["rating", "<0-to-1-score>", "thumb"]`;
3. the referenced order group validates structurally;
4. the order group is confirmed committed;
5. either:
   - the review pubkey appears as an order participant pubkey in the referenced order group; or
   - the `review_proof` tag's authorization payload hashes to a `participant_proof` payload hash in the referenced order group, and the decoded `kind:1329` authorization event is valid and matches the claimed role, participant pubkey, listing anchor, and trade id.

This NIP does not define a canonical on-chain or block-time proof that the review was written after the order ended. Clients MAY additionally require the order end time to be in the past or a terminal payment event to exist, but those timing rules are application policy unless standardized separately.

## Protocol Flow

### 1. Buyer Initiates Negotiation

1. Buyer discovers a NIP-99 listing.
2. Buyer allocates a trade id and temporary trade key (temp-key). Deterministic derivation is optional and described in the addendum below.
3. Buyer creates a `kind:32122` order with `stage=negotiate`, signed by the temporary trade key (temp-key).
4. Buyer includes role-marked participant `p` tags and encrypted `participant_proof` tags as needed.
5. Buyer sends the order as a child event inside a private `kind:1327` rumor tagged `["conversation", "<trade-id>"]`, delivered with NIP-59 gift wraps to the resolved seller participant and self.

### 2. Negotiation

6. Seller reviews the request. If counter-offering, seller creates a new `stage=negotiate` order with changed terms and embeds a signed `kind:1328` commit authorization for those terms.
7. If the seller accepts the current terms, the seller replies with a `stage=negotiate` order for those terms and embeds a signed `kind:1328` commit authorization.
8. Counteroffers continue privately until terms are accepted or cancelled.

### 3. Escrow Selection and Payment

9. If escrow-backed, buyer sends a private `kind:30302` Escrow Service Selected child event inside a `kind:1327` rumor tagged `["conversation", "<trade-id>"]`.
10. Buyer funds escrow directly or via a Lightning-to-on-chain swap.

### 4. Commitment

11. Once payment proof exists, the buyer or temporary trade key (temp-key) publishes a public `stage=commit` `kind:32122` order with `proof`.
<!-- Order Transition publication step intentionally commented out. -->
12. Seller or escrow MAY publish their own `stage=commit` order/confirmation.

### 5. Cancellation

Either party MAY cancel a private negotiation with a private `stage=cancel` order child event. After public commitment, cancellation publishes a public `stage=cancel` order. Escrow may only cancel before it has committed.

## Availability

Clients MUST only consider public order groups after applying the Order Group validity ordering when computing listing availability. Negotiation orders are exchanged only between buyer and seller via gift-wrapped private structured messages, so they are not part of public listing availability calculations.

Order clients SHOULD verify that the referenced listing is an active NIP-99 listing before displaying or accepting an order.

## Addendum: How to Choose Temporary Keys

Temporary trade keys (temp-keys) exist to preserve privacy by separating a buyer's public account identity from public order events. This NIP does not require deterministic values for trade ids or temporary keys. A client MAY generate a fresh random trade id and random temporary key for each trade and store the resulting private state locally.

A Nostr user MAY instead publish a single encrypted SEED event (`kind:17389`) globally. The SEED event content is a NIP-44 encrypted payload from the user's identity key to itself, for example:

```jsonc
{
  "v": 1,
  "seed": "<32-byte-hex-seed>"
}
```

Clients that can decrypt this SEED event can derive arbitrary trade ids and temporary trade keys from the seed, avoiding dependence on one device's local storage of encrypted gift wraps for each individual trade. This is a convenience and recovery mechanism, not a consensus requirement: clients MUST accept valid orders and participant proofs regardless of whether their trade ids and temporary keys were random, locally stored, or derived from a published SEED event.

Clients that implement deterministic derivation SHOULD domain-separate trade ids from temporary keys and SHOULD include an account-controlled index or nonce so the same seed never produces the same public key for two unrelated trades.

## Related NIPs

- [NIP-01](01.md) — Event structure and parameterized replaceable events.
- [NIP-99](99.md) — Classified listings.
- [NIP-17](17.md) — Private message rumor kind `14`.
- [NIP-19](19.md) — `naddr` encoding for anchors.
- [NIP-44](44.md) — Encryption scheme.
- [NIP-57](57.md) — Zaps.
- [NIP-59](59.md) — Gift wrap.
