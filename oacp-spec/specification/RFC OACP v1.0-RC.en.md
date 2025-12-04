# RFC: Open Agent Commerce Protocol (OACP)
**Version:** 1.0 (PROPOSED STANDARD)
**Status:** CODE FREEZE
**Date:** 2025-12-02
**Layer:** 2 (Application)
**Dependencies:** OATP v1.0, OAEP v1.0, OAPP v1.0

## 1. Introduction

The **Open Agent Commerce Protocol (OACP)** defines the standard for decentralized, automated commerce between AI agents. It replaces proprietary API silos and centralized marketplaces with an open, semantic protocol that standardizes Discovery (Search), Negotiation, and Settlement.

While OAEP secures identity and OATP ensures transport, OACP provides the **business logic**. It transforms e-commerce from a catalog-based model ("Search for what exists") to an intent-based model ("Describe what you need" – **Intent-Centric Commerce**).

### 1.1 Objectives and Maturity Level
This Version 1.0 defines a production-ready standard that meets the following core requirements:
1.  **Semantic Interoperability:** Utilization of Schema.org and JSON-LD so that agents can understand products and services without human intervention.
2.  **Trustless Safety:** Integration of escrow mechanisms and arbitration procedures to make trading between unknown parties safe.
3.  **Verifiable Truth:** Product properties (e.g., "Organic", "Original") are not marketing claims but are cryptographically provable via W3C Verifiable Credentials.

### 1.2 Integration in the OAP Stack
OACP is intrinsically linked to its sister protocols:
*   **Identity & Signature:** All critical actions (ordering, contract conclusion) are signed with OAEP keys (`UserProof`).
*   **Transport:** All messages are sent as encrypted payloads within OATP containers.
*   **Payment:** Payment processing is delegated to **OAPP v1.0**, which is seamlessly embedded into the OACP workflow.

---

## 2. Data Model & Semantics

OACP uses **JSON-LD** and relies strictly on the **Schema.org** vocabulary, extended by protocol-specific control fields for agent dialogue.

### 2.1 The Message Envelope (Base Message)
Every OACP message inherits from this base schema and is packed as a `payload` into an OATP container.

```json
{
  "@context": ["https://schema.org", "https://w3id.org/oacp/v1"],
  "type": "OACPMessage",        // Abstract base type
  "id": "urn:uuid:...",         // Unique ID of the message
  "threadId": "urn:uuid:...",   // UUIDv4, correlates the entire process (Session)
  "sender": "did:key:buyer...", // Sender DID
  "recipient": "did:web:shop.com", // Recipient DID
  "created": "2025-12-02T14:00:00Z"
}
```

### 2.2 Constraints (Normative Operators)
To formulate Intents (purchase intentions) precisely and in a machine-readable way, OACP defines a normative list of operators for constraints. Merchant agents MUST be able to interpret these operators.

#### 2.2.1 MUST Operators (Mandatory)
*   `equals`: Exact match (String, Number, Boolean).
*   `notEquals`: Exclusion of a value.
*   `lessThan`, `lessThanOrEquals`: Numerical comparison (for prices, dimensions, weights).
*   `greaterThan`, `greaterThanOrEquals`: Numerical comparison.
*   `contains`: Checks if an array contains a specific value (e.g., "colors contains Red").
*   `exists`: Checks if an attribute exists at all (regardless of value).

#### 2.2.2 MAY Operators (Optional)
*   `regex`: Comparison using Regular Expressions (for complex string patterns).
*   `inRange`: Checks if a value lies within a defined interval (`[min, max]`).

**Example of a Constraint Object:**
```json
{
  "property": "schema:memory", // JSON-LD path to attribute
  "operator": "greaterThanOrEquals",
  "value": "16 GB",
  "required": true // Hard Constraint (Must be met)
}
```

### 2.3 Verifiable Credentials Validation (Trust Check)
If an offer contains VCs (e.g., a sustainability certificate), the Buyer Agent MUST validate the following steps before presenting the offer to the user as "verified":

1.  **Trust Root:** Is the `issuer` of the VC listed in a *Trusted Registry* accepted by the user or the OAP Association?
2.  **Integrity:** Is the cryptographic signature (`proof`) of the VC mathematically valid and does it belong to the `issuer`?
3.  **Freshness:** Has the `expirationDate` of the VC not yet passed, and is the status in the *Revocation List* (StatusList2021) "valid"?
4.  **Binding:** Does the `credentialSubject.id` of the VC (e.g., GTIN/SKU) uniquely match the offered product?

---

## 3. Protocol Flow (The Commerce Flow)

The trading process is modeled as an asynchronous state machine. The `threadId` links all messages of a transaction.

### Phase 1: Discovery & Negotiation

**Step 1.1: `NegotiateRequest` (Buyer -> Merchant)**
The Buyer Agent sends an Intent. It describes *what* is being sought and under which conditions.

```json
{
  "type": "NegotiateRequest",
  "threadId": "urn:uuid:thread-123",
  "intent": {
    "@type": "Product",
    "category": "RunningShoes"
  },
  "constraints": [
    { "property": "schema:size", "operator": "equals", "value": "42" },
    { "property": "schema:price", "operator": "lessThanOrEquals", "value": "150" }
  ],
  "requiredCredentials": ["EcoFriendlyCertified"] 
}
```

**Step 1.2: `OfferResponse` (Merchant -> Buyer)**
The merchant responds with a concrete, binding offer.
*   **Validity:** The `validUntil` field defines how long the offer is reserved or guaranteed.
*   **Credentials:** The requested proofs (VCs) are delivered here.

### Phase 2: Commitment (Contract)

**Step 2.1: `OrderRequest` (Buyer -> Merchant)**
The buyer accepts the offer. This step creates a legal obligation. To prevent "AI hallucination purchases," this requires a cryptographic signature by the human.

#### 3.2.1 Normative Specification of `UserProof`
The `UserProof` binds the buyer's identity to the specific conditions of the offer.

1.  **Canonicalization:** Creation of a canonical JSON representation (JCS according to RFC 8785) of the fields: `threadId`, `offerId`, `price`, `currency`, `itemSku`, `timestamp`.
2.  **Hashing:** Calculation of the **BLAKE3** hash of this canonical data.
3.  **Signature:** Signing the hash with the private OAEP key (`Ed25519`).

```json
{
  "type": "OrderRequest",
  "acceptedOfferId": "urn:uuid:offer-987",
  "shippingAddress": { ... }, // OATP-Encrypted
  "userProof": {
    "type": "OaepSignature2025",
    "created": "2025-12-02T14:05:00Z",
    "signedHash": "HASH_VALUE...",
    "signatureValue": "..."
  }
}
```

**Step 2.2: `OrderConfirmation` (Merchant -> Buyer)**
The merchant accepts the contract, reserves the goods, and requests payment.

```json
{
  "type": "OrderConfirmation",
  "orderId": "ORDER-2025-ABC",
  "status": "WaitingForPayment",
  "paymentRequest": { 
    // Structure according to OAPP v1.0 Standard
    "type": "PaymentRequest",
    "amount": "12999",
    "currency": "EUR",
    "escrow": { ... } 
  }
}
```

### Phase 3: Payment & Trust

Here, the handover to **OAPP v1.0** takes place. The merchant waits for the `PaymentReceipt` (Status `SETTLED` or `ESCROW_LOCKED`).

#### 3.3.1 Escrow Release
In escrow payments, funds are locked on the ledger. Release is triggered by the buyer.
*   **Message:** `DeliveryAcknowledgement` (Buyer -> Escrow Agent & Merchant).
*   **Content:** Confirmation of receipt and integrity.
*   **Effect:** The Escrow Agent then sends the `EscrowReleaseInstruction` to the Ledger (see OAPP v1.0 Chap. 3.3).

### Phase 4: Fulfillment (Delivery)

**Step 4.1: `DeliveryUpdate` (Merchant -> Buyer)**
Status updates regarding delivery (shipping, tracking).

#### 3.4.1 Partial Fulfillment
If the merchant cannot deliver the full quantity, they send an update with `status: PARTIALLY_FULFILLED`.

*   **Refund Calculation:**
    If payment has already been made, the merchant MUST refund the difference via OAPP (`POST /refund`).
    *   *Formula:* `Refund = (MissingQuantity * UnitPrice)`.
*   **Protocol:**
    ```json
    {
      "type": "DeliveryUpdate",
      "status": "PARTIALLY_FULFILLED",
      "fulfilledItems": [{"sku": "A", "qty": 1}],
      "pendingItems": [{"sku": "B", "qty": 1, "status": "CANCELLED"}],
      "refundData": {
        "amount": "2999",
        "currency": "EUR",
        "reason": "OUT_OF_STOCK"
      }
    }
    ```

### Phase 5: Dispute Resolution

If a problem occurs, this flow is activated.

**Step 5.1: `DisputeRequest` (Buyer/Merchant -> Arbiter)**
One party calls upon the arbitrator (Arbiter) defined in the escrow contract.
```json
{
  "type": "DisputeRequest",
  "reason": "ITEM_DAMAGED",
  "evidence": ["url:photo1", "url:log"], // OATP-Encrypted Links
  "proposedResolution": "FULL_REFUND"
}
```

**Step 5.2: `DisputeRuling` (Arbiter -> Escrow)**
The Arbiter reviews the evidence and issues a ruling. They send a signed instruction to the Escrow Agent, who technically implements it on the ledger (release to merchant or refund to buyer).

---

## 4. State Machine

Trade integrity is based on synchronized states between buyer and seller. Both agents MUST implement this state machine to prevent race conditions and "hanging states."

| Status | Description | Trigger / Next Step | Timeout / Error Path |
| :--- | :--- | :--- | :--- |
| **INIT** | `NegotiateRequest` sent. | Receive `OfferResponse` $\rightarrow$ `OFFER_RECEIVED` | 1h $\rightarrow$ `CANCELLED` |
| **OFFER_RECEIVED** | Offer is present. | Send `OrderRequest` $\rightarrow$ `COMMITTED` | **24h** $\rightarrow$ `OFFER_EXPIRED` |
| **COMMITTED** | Order sent, waiting for confirmation. | Receive `OrderConfirmation` $\rightarrow$ `LOCKED` | Merchant Reject $\rightarrow$ `CANCELLED` |
| **LOCKED** | Goods reserved. Waiting for payment. | OAPP `PaymentReceipt` $\rightarrow$ `PAID` | **15 Min** $\rightarrow$ `PAYMENT_FAILED` |
| **PAID** | Payment confirmed (Settled or Escrow). | Send `DeliveryUpdate` $\rightarrow$ `FULFILLED` | 30 Days $\rightarrow$ `DISPUTED` (Non-Delivery) |
| **FULFILLED** | Goods shipped / Service rendered. | User Ack / Timer $\rightarrow$ `COMPLETED` | User Report $\rightarrow$ `DISPUTED` |
| **COMPLETED** | Transaction successfully completed. | (End state) | - |
| **DISPUTED** | Conflict reported. Arbitration active. | `DisputeRuling` $\rightarrow$ `RESOLVED` | 7 Days $\rightarrow$ Auto-Resolution |
| **RESOLVED** | Conflict resolved (Refund/Release). | (End state) | - |
| **CANCELLED** | Cancellation before payment. | (End state) | - |

### 4.1 Normative Timeouts & Ownership

To define which party initiates cancellation in case of network issues or inactivity, the following rules apply:

1.  **Payment Timeout (15 Minutes):**
    *   *Responsibility:* The **Merchant Agent**.
    *   *Logic:* If no `PaymentReceipt` (OAPP) or at least a `PaymentAuthorization` (Status `PENDING`) is received 15 minutes after sending the `OrderConfirmation`, the status changes to `PAYMENT_FAILED`. The reservation of goods is released.
    *   *Client View:* The Buyer Agent MUST display a countdown to the user and lock the UI upon expiration.

2.  **Delivery Timeout (Standard: 30 Days):**
    *   *Responsibility:* The **Buyer Agent**.
    *   *Logic:* If no `DeliveryUpdate` occurs after 30 days (or the `deliveryLeadTime` defined in the offer + buffer), the Buyer Agent automatically sends a `DisputeRequest` (Reason: `NOT_DELIVERED`). This ensures that funds do not remain frozen in escrow forever.

3.  **Offer Expiry (24 Hours):**
    *   *Responsibility:* The **Merchant Agent**.
    *   *Logic:* Offers are valid for 24 hours by default, unless defined otherwise in the `validUntil` field. Subsequent `OrderRequests` are rejected with `OACP_OFFER_EXPIRED`.

### 4.2 Error Recovery & Offer Renewal

If an offer expires (`OFFER_EXPIRED`) or a payment fails (`PAYMENT_FAILED`), the transaction initially fails.

*   **Offer Renewal:** The Merchant MAY send an `OfferExpiredNotification` containing an updated offer (`newOffer`) (e.g., with updated price or new validity).
*   **Re-Negotiation:** The Buyer MAY then respond with a new `OrderRequest` (referencing the new Offer ID) without restarting the entire Negotiation process. The `threadId` is preserved.

---

## 5. Error Handling

If an OACP message was transmitted technically correctly (OATP) but cannot be processed logically, the recipient MUST respond with an `OACPError` message.

### 5.1 The Error Object
Error messages are JSON-LD objects referring to the `threadId` of the process.

```json
{
  "@context": ["https://w3id.org/oacp/v1"],
  "type": "OACPError",
  "id": "urn:uuid:error-...",
  "threadId": "urn:uuid:original-thread-id",
  "code": "OACP_OFFER_EXPIRED",
  "message": "The offer 123 is no longer valid.",
  "details": { ... } // Optional: Context-specific data (e.g., new offer)
}
```

### 5.2 Normative Error Codes

Implementations MUST support the following codes and react according to the "Client Action":

| Code | Meaning | Trigger | Client Action |
| :--- | :--- | :--- | :--- |
| **`OACP_UNSUPPORTED_CONSTRAINT`** | Criterion unsatisfiable. | `NegotiateRequest` contains conditions the merchant cannot technically check or does not meet. | Relax constraints and request again. |
| **`OACP_OUT_OF_STOCK`** | Goods out of stock. | `OrderRequest` for a product sold in the meantime (Race Condition). | Inform user, search for alternative. |
| **`OACP_OFFER_EXPIRED`** | Offer expired. | `OrderRequest` occurs after expiration of `validUntil` or the 24h standard. | Start Re-Negotiation (see 4.2). |
| **`OACP_INVALID_PROOF`** | Signature invalid. | The `UserProof` in `OrderRequest` is cryptographically incorrect or does not belong to the sender's DID. | Abort. Check Wallet/Key setup. |
| **`OACP_PAYMENT_TIMEOUT`** | Payment too late. | Merchant has not registered payment receipt after 15 min (Status `LOCKED`). | Place new order. |
| **`OACP_PAYMENT_FAILED`** | Payment rejected. | OAPP reports error (e.g., insufficient funds). | Change payment method and retry via OAPP. |
| **`OACP_DISPUTE_REJECTED`** | Arbitration rejected. | `DisputeRequest` is unfounded or spam. | Manual clarification outside the protocol. |

### 5.3 Extension Points
Merchants CAN define their own error codes, but these MUST begin with a vendor prefix (e.g., `EXT_SHIPPING_REGION_LOCKED`) to avoid collisions with future standards. Unknown error codes SHOULD be treated generically by the client as "Transaction Failed."

---

## 6. Security & Privacy Implications

Since OACP is designed to process purchase intentions (Intents) and personal data (addresses), stricter data protection requirements apply than for pure transport protocols.

### 6.1 Privacy of Intent
A `NegotiateRequest` ("I am looking for Medication X") contains highly sensitive data.
*   **Risk:** If these requests are sent unencrypted or with a persistent identity, merchants or network observers can create detailed psychographic profiles.
*   **Mitigation:** User Agents SHOULD use anonymous networks (such as Tor or OATP Mesh Routing) or trusted **Discovery Proxies** for the Discovery Phase (Broadcasting of Intents) to decouple the request from the user.

### 6.2 Anti-Doxing & Data Minimization
*   **Shipping Addresses:** The `shippingAddress` is the most sensitive datum. It must **exclusively** be transmitted in the `OrderRequest` (Phase 2). A merchant demanding the address already in the `NegotiateRequest` MUST be marked as "suspicious" by the agent.
*   **Encryption:** Since OACP is transported via OATP, address data is protected against relays ("Blind Delivery"). However, the merchant receives it in cleartext for processing.

### 6.3 Prevention of Price Discrimination
Merchants might attempt to adjust prices based on purchase history or the perceived wealth of a DID ("Dynamic Pricing").

*   **Requirement (Incognito Mode):** For Phase 1 (`NegotiateRequest`), User Agents SHOULD use rotating, ephemeral identities (`did:key`) by default, which have no linkable history.
*   **Reputation Switch:** Only at the `OrderRequest`, when trust is needed for settlement (e.g., "I am a reliable payer"), CAN the agent switch to a persistent Reputation DID.

### 6.4 Escrow Security
The Escrow model relies on the independence of the trustee.
*   **Separation of Duties:** The Escrow account MUST NOT be controlled by the merchant.
*   **Warning Duty:** The Buyer Agent MUST warn the user if:
    1.  The DID of the `escrow.agent` is identical to the DID of the `recipient` (Merchant).
    2.  The Escrow Agent has a low reputation in the OAP Trust Network.

### 6.5 Protection against "Rogue Agents" (AI Safety)
Since AI models can hallucinate or be manipulated by "Prompt Injection," the agent must never spend money autonomously.
*   **UserProof Constraint:** Merchants MUST reject orders (`OACP_INVALID_PROOF`) that do not contain a valid cryptographic proof generated by human interaction (Biometrics/PIN). A pure AI signature is not sufficient for `OrderRequest`.

---

## 7. JSON Schema (Normative)

To ensure interoperability and stability of the network, OACP implementations MUST validate incoming messages against the following JSON Schemas (Draft 07). Messages violating the schema MUST be rejected with a transport error or `OACP_UNSUPPORTED_CONSTRAINT`.

### 7.1 `NegotiateRequest` Schema
Defines the structure for search requests and constraints.

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "https://w3id.org/oacp/v1/NegotiateRequest",
  "type": "object",
  "required": ["type", "threadId", "intent"],
  "properties": {
    "type": { "const": "NegotiateRequest" },
    "threadId": { 
      "type": "string", 
      "pattern": "^urn:uuid:[0-9a-fA-F-]{36}$" 
    },
    "intent": {
      "type": "object",
      "required": ["@type"],
      "properties": {
        "@type": { "type": "string" }, // e.g., "Product"
        "category": { "type": "string" }
      }
    },
    "constraints": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["property", "operator", "value"],
        "properties": {
          "property": { "type": "string" },
          "operator": { 
            "enum": [
              "equals", "notEquals", 
              "lessThan", "lessThanOrEquals", 
              "greaterThan", "greaterThanOrEquals", 
              "contains", "exists", 
              "regex", "inRange"
            ] 
          },
          "required": { "type": "boolean", "default": true }
        }
      }
    },
    "requiredCredentials": {
      "type": "array",
      "items": { "type": "string" } // e.g., "EcoFriendlyCertified"
    }
  }
}
```

### 7.2 `OfferResponse` Schema
Ensures that offers contain binding validity dates and prices.

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "https://w3id.org/oacp/v1/OfferResponse",
  "type": "object",
  "required": ["type", "threadId", "offer"],
  "properties": {
    "type": { "const": "OfferResponse" },
    "threadId": { "type": "string", "pattern": "^urn:uuid:.*$" },
    "offer": {
      "type": "object",
      "required": ["id", "price", "priceCurrency", "validUntil", "itemOffered"],
      "properties": {
        "id": { "type": "string", "format": "uri" },
        "price": { "type": "number", "minimum": 0 },
        "priceCurrency": { "type": "string", "minLength": 3, "maxLength": 3 },
        "validUntil": { "type": "string", "format": "date-time" },
        "itemOffered": {
          "type": "object",
          "required": ["name"],
          "properties": {
            "sku": { "type": "string" }
          }
        }
      }
    },
    "verifiableCredentials": {
      "type": "array",
      "items": { "type": "object" } // W3C VC Structure (externally defined)
    }
  }
}
```

### 7.3 `OrderRequest` Schema
Validates the cryptographic binding of the `UserProof`.

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "https://w3id.org/oacp/v1/OrderRequest",
  "type": "object",
  "required": ["type", "threadId", "acceptedOfferId", "shippingAddress", "userProof"],
  "properties": {
    "type": { "const": "OrderRequest" },
    "threadId": { "type": "string" },
    "acceptedOfferId": { "type": "string", "format": "uri" },
    "shippingAddress": {
      "type": "object",
      "required": ["@type", "streetAddress", "addressCountry"],
      "properties": {
        "@type": { "const": "PostalAddress" }
      }
    },
    "userProof": {
      "type": "object",
      "required": ["type", "created", "signedHash", "signatureValue"],
      "properties": {
        "type": { "const": "OaepSignature2025" },
        "created": { "type": "string", "format": "date-time" },
        "signedHash": { "type": "string", "description": "BLAKE3 Hash of canonical order data" },
        "signatureValue": { "type": "string", "description": "Ed25519 Signature" }
      }
    }
  }
}
```

---

## 8. Appendix (Informative)

### 8.1 Reference Implementation

The OAP Foundation provides an official Open Source library that encapsulates the normative logic (State Machine, Validation):

*   **Repository:** `oap-foundation/oacp-core-rs` (Rust)
*   **Modules:**
    *   `oacp::intents`: Builder for Negotiate/Order Requests.
    *   `oacp::validation`: Logic for Constraint Matching and VC checking.
    *   `oacp::state`: The state machine for transactions.

### 8.2 Migration from OACP v0.1 (Legacy / GitHub)

For developers who have already worked with the original specification (`oacp-spec` on GitHub), these are the most important changes in v1.0:

1.  **Transport Layer:** OACP no longer uses direct HTTP/WebSockets but strictly requires **OATP v1.0**. All messages must be packed into OATP containers.
2.  **Asynchrony:** The `threadId` is now mandatory to correlate messages in an asynchronous environment.
3.  **UserProof:** The signature in `OrderRequest` is no longer optional or proprietary but strictly follows the **OAEP JCS/BLAKE3/Ed25519** standard.
4.  **Payment:** The embedding of payment information is no longer generic but strictly according to **OAPP v1.0** (`PaymentRequest` object).

### 8.3 Complete Example Scenario

**Scenario:** Buyer (DID: `did:key:buyer`) searches for a sustainable laptop. Merchant (DID: `did:web:tech-store.com`) offers one.

#### A. NegotiateRequest (Buyer)
```json
{
  "@context": ["https://schema.org", "https://w3id.org/oacp/v1"],
  "type": "NegotiateRequest",
  "id": "urn:uuid:msg-1",
  "threadId": "urn:uuid:thread-abc-123",
  "sender": "did:key:buyer",
  "recipient": "did:web:tech-store.com",
  "intent": {
    "@type": "Product",
    "category": "Laptop",
    "description": "High performance, eco friendly"
  },
  "constraints": [
    { "property": "schema:memory", "operator": "greaterThanOrEquals", "value": "16 GB" },
    { "property": "schema:price", "operator": "lessThan", "value": 2000 }
  ],
  "requiredCredentials": ["EcoLabelEU"]
}
```

#### B. OfferResponse (Merchant)
```json
{
  "type": "OfferResponse",
  "id": "urn:uuid:msg-2",
  "threadId": "urn:uuid:thread-abc-123",
  "offer": {
    "id": "urn:uuid:offer-999",
    "validUntil": "2025-12-03T18:00:00Z",
    "price": 1899.00,
    "priceCurrency": "EUR",
    "itemOffered": {
      "@type": "Product",
      "name": "GreenBook Pro 14",
      "sku": "GBP-14-16GB"
    }
  },
  "verifiableCredentials": [
    {
      "type": ["VerifiableCredential", "EcoLabelEU"],
      "issuer": "did:web:eu-label.org",
      "proof": { "..." }
    }
  ]
}
```

#### C. OrderRequest (Buyer)
```json
{
  "type": "OrderRequest",
  "id": "urn:uuid:msg-3",
  "threadId": "urn:uuid:thread-abc-123",
  "acceptedOfferId": "urn:uuid:offer-999",
  "shippingAddress": {
    "@type": "PostalAddress",
    "streetAddress": "Innovationsstraße 1",
    "addressCountry": "AT"
  },
  "userProof": {
    "type": "OaepSignature2025",
    "signedHash": "abc...", // Hash(offer-999 + price + sku)
    "signatureValue": "xyz..."
  }
}
```

#### D. OrderConfirmation (Merchant)
```json
{
  "type": "OrderConfirmation",
  "id": "urn:uuid:msg-4",
  "threadId": "urn:uuid:thread-abc-123",
  "orderId": "ORD-2025-7788",
  "status": "WaitingForPayment",
  "paymentRequest": {
    "type": "PaymentRequest",
    "amount": "189900", // Minor Units
    "currency": "EUR",
    "beneficiary": { "did": "did:web:tech-store.com" },
    "escrow": {
      "agent": "did:web:trustee.org",
      "releaseCondition": "DELIVERY_CONFIRMED"
    }
  }
}
```