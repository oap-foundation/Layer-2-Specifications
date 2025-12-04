# RFC: Open Agent Payment Protocol (OAPP)
**Version:** 1.0 (PROPOSED STANDARD)
**Status:** CODE FREEZE
**Date:** 2025-12-02
**Layer:** 2 (Application)
**Dependencies:** OATP v1.0, OAEP v1.0, OACP v1.0

## 1. Introduction

The **Open Agent Payment Protocol (OAPP)** serves as the financial backbone of the OAP ecosystem. It acts as the **Settlement Layer**, translating the contractual intent of two parties (defined by OACP) into an irrevocable transfer of value.

OAPP functions as a universal adapter. It decouples the **authorization** of a payment (by the user and their agent) from the **technical execution** (by ledgers, banks, or gateways). This enables the seamless coexistence of modern token economies (OAP-Credits) and traditional financial systems (SEPA, Credit Cards) under a unified security standard.

### 1.1 Objectives and Maturity Level
This Version 1.0 defines a production-ready standard for secure payment transactions between autonomous agents. It integrates all necessary security mechanisms for live operation:
*   **Legal Certainty:** PSD2-compliant authorization and eIDAS integration.
*   **Escrow Security:** A native escrow mechanism for trustless trading.
*   **Robustness:** Normative error handling, idempotency, and replay protection for financial transactions.

### 1.2 Scope of Application
OAPP covers three primary scenarios:
1.  **Internal Economy:** Direct P2P transactions with OAP-Credits via the Association Ledger.
2.  **Open Banking:** Triggering bank transfers (SEPA Instant) via PSD2 bridges.
3.  **Legacy Commerce:** Integration of classic Payment Service Providers (PSPs) like Stripe or PayPal.

---

## 2. Data Model (Harmonized with OACP v1.0)

OAPP uses JSON-LD. The objects defined here are transported as payloads within the OATP container and are structurally tightly interwoven with OACP business processes.

### 2.1 `PaymentRequest`
This object defines the conditions of the payment. It is typically created by the Payee (Merchant) and is often embedded within an OACP `OrderConfirmation`.

```json
{
  "@context": ["https://w3id.org/oapp/v1"],
  "type": "PaymentRequest",
  "id": "urn:uuid:pay-req-123",
  "threadId": "urn:uuid:oacp-thread-abc", // Link to OACP trade
  
  // Commercial Data (Must match OACP Order)
  "amount": "12999", // Minor Units (Cents) as String/Integer
  "currency": "EUR", // ISO 4217 or "OAP"
  "reference": "ORDER-2025-ABC",
  
  // Recipient Data
  "beneficiary": {
    "did": "did:web:shop.com",
    "name": "Th!nk Store GmbH"
  },

  // Technical Settlement (Method Selection)
  "supportedMethods": [
    {
      "type": "OAP_LEDGER",
      "targetAccount": "did:web:shop.com#wallet-oap"
    },
    {
      "type": "PSD2_SEPA",
      "targetAccount": "IBAN_ENCRYPTED_BLOB..." // JWE Encrypted
    }
  ],

  // Escrow (Optional, adopted from OACP)
  "escrow": {
    "required": true,
    "agent": "did:web:trustee.org",
    "releaseCondition": "DELIVERY_CONFIRMED"
  },

  "expiresAt": "2025-12-02T15:00:00Z"
}
```

### 2.2 `PaymentAuthorization`
The response from the Payer Agent. It contains the selection of the payment method and the irrevocable cryptographic proof of consent (`UserProof`).

#### 2.2.1 Normative Nonce Format & Hashing
To prevent replay attacks and maintain crypto-agility, the following rules apply:

1.  **Nonce Construction:**
    `Base64URL( BigEndian(Timestamp_Unix_Seconds) || Random_16_Bytes )`
    *   *Timestamp:* Enables verification of the 5-minute time window.
    *   *Random:* Guarantees uniqueness within the window.

2.  **Hash Algorithm:**
    The algorithm used for `signedHash` MUST correspond to the hash function of the active **OAEP Cipher Suite** (e.g., **BLAKE3** for `OAEP-v1-2026`). The use of fixed SHA-256 is deprecated.

```json
{
  "type": "PaymentAuthorization",
  "paymentRequestId": "urn:uuid:pay-req-123",
  "selectedMethod": "OAP_LEDGER",
  
  // The UserProof (Signed with OAEP Hardware Key)
  "authorizationProof": {
    "type": "OaepSignature2025",
    "created": "2025-12-02T14:10:00Z",
    "algorithm": "Ed25519", // According to OAEP Suite
    "digest": "BLAKE3",      // According to OAEP Suite
    
    // Hash over: paymentRequestId + amount + currency + beneficiaryDID + nonce
    "signedHash": "HASH_VALUE_BASE64...",
    "nonce": "Base64URL(Timestamp + Random)", 
    "signatureValue": "..." 
  },

  // Method-specific Data (e.g., Token)
  "methodData": {
    "payerWallet": "did:key:user#main"
  }
}
```

### 2.3 `PaymentReceipt`
The conclusion of the transaction. Signed by the Payee (Merchant) or the Gateway once funds have been finally moved.

```json
{
  "type": "PaymentReceipt",
  "paymentRequestId": "urn:uuid:pay-req-123",
  "status": "SUCCESS",
  "transactionId": "tx_ledger_998877", // External ID (Bank/Ledger)
  "settledAt": "2025-12-02T14:10:05Z",
  "fee": "0",
  // Optional: Reference for refunds
  "refundId": "ref_tx_998877_refund" 
}
```

---

## 3. Scenario A: OAP Ledger API (Native Tokens)

This is the standard scenario for transactions within the OAP Association ("Circular Economy"). The **OAP Ledger Service** is the central authority that maintains accounts based on DIDs and finalizes transactions.

### 3.1 Architecture
*   **Payer Agent:** Sends `PaymentAuthorization` via OATP to the Payee.
*   **Payee Agent:** Forwards the Authorization to the Ledger API (Settlement).
*   **Ledger:** Verifies signature and funds, then executes the booking atomically.

### 3.2 Ledger API Specification (REST over HTTPS)
The Ledger offers a public but authenticated API. All requests must be signed via OAEP (Bearer Token).

#### 3.2.1 `POST /v1/transfer` (Execution)
This endpoint executes the actual value transfer.
*   **Request Body:** The complete `PaymentAuthorization` object + the original `PaymentRequest` (to validate conditions).
*   **Response (Success):** `200 OK` with `{"txId": "oap:tx:...", "status": "SETTLED"}`.

#### 3.2.2 Normative Error Codes (Error Handling)
The Ledger MUST return the following HTTP status codes and OAPP-specific error codes:

| HTTP | OAPP Code | Meaning | Client Action |
| :--- | :--- | :--- | :--- |
| **400** | `INVALID_SIGNATURE` | OAEP signature invalid or Nonce format incorrect. | Abort (Fatal). |
| **402** | `INSUFFICIENT_FUNDS` | Payer account balance too low. | Notify user. |
| **409** | `DUPLICATE_TRANSACTION` | `paymentRequestId` has already been processed. | **Treat as Success** (Idempotency). Return old `txId`. |
| **422** | `EXPIRED_REQUEST` | Timestamp too old (outside Replay Window). | Request new payment request. |
| **500** | `LEDGER_ERROR` | Internal error (Consensus, DB). | Retry with backoff. |

### 3.3 Escrow Settlement

If an `escrow` object was defined in the `PaymentRequest`, funds do not flow directly to the merchant but to a technical escrow account held by the Ledger.

#### 3.3.1 The Lock and Release Process
1.  **Locking:** Upon successful `POST /transfer`, the Ledger books funds from the Payer to the `EscrowAccount`. Transaction status becomes `ESCROW_LOCKED`.
2.  **Trigger:** OACP reports `DELIVERY_CONFIRMED` or an arbitration decision (`DisputeRuling`).
3.  **Instruction:** The Escrow Agent (a dedicated OAEP agent) sends a signed instruction to the Ledger.

#### 3.3.2 `EscrowReleaseInstruction`
```json
{
  "type": "EscrowReleaseInstruction",
  "escrowTxId": "tx_ledger_lock_123",
  "action": "RELEASE",                // "RELEASE" (to Payee) or "REFUND" (to Payer)
  "beneficiary": "did:web:shop.com",  // Target DID
  "amount": "12999",
  "reason": "Delivery confirmed by buyer",
  "signature": "..."                  // Signature of the Escrow Agent
}
```

### 3.4 Consensus Models (Governance)
For Version 1.0, the OAP Ledger operates as a **Single Trusted Validator**, technically operated by the OAP Association.
*   **Roadmap:** Version 2.0 plans a transition to a federated model (Consortium Ledger) where cooperative members validate via PBFT (Practical Byzantine Fault Tolerance) consensus.

### 3.5 Fee Distribution (Fee Model)
To maintain price transparency for the buyer ("What you see is what you pay"), the following is normative:
*   **Payee Pays:** Transaction fees are deducted from the *received amount*.
*   *Example:* Buyer sends 100. Fee is 1. Ledger books: Payer -100, Payee +99, FeeAccount +1.

### 3.6 Refund API
Enables the Merchant (Payee) to return money from a completed transaction easily and securely.

**Endpoint:** `POST /v1/refund`
*   **Body:**
    ```json
    {
      "originalTxId": "tx_ledger_123",
      "amount": "12999", // Can be <= Original Amount (Partial Refund)
      "reason": "ITEM_RETURNED"
    }
    ```
*   **Auth:** The request MUST be signed by the Payee (Recipient of the original Tx).
*   **Logic:** The Ledger checks if `originalTxId` exists, if the requester was the original recipient, and if funds are available.

---

## 4. Scenario B: PSD2 / Open Banking (The Bank Adapter)

In this scenario, OAPP acts as a bridge to the classical banking system (e.g., SEPA Instant in Europe). Since direct access to bank APIs (XS2A) is reserved for regulated Third Party Providers (TPPs), OAPP uses a certified intermediary: the **PSD2 Bridge**.

### 4.1 Architecture and Trust Model

*   **The Payer Agent:** Initiates the payment but holds no banking certificate itself.
*   **The PSD2 Bridge:** A service provider (e.g., a licensed FinTech) that speaks OATP and possesses the necessary eIDAS certificates (QWAC/QSealC) to communicate with banks.

#### 4.1.1 Bridge Authentication (Security)
Since the Bridge processes sensitive payment data, its trustworthiness must be absolute.

1.  **eIDAS Requirement:** The Bridge **MUST** possess a valid QWAC (Qualified Website Authentication Certificate) or QSealC verifying its role as a PISP (Payment Initiation Service Provider).
2.  **Trusted Registry:** The Payer Agent **MUST** verify, before sending data, whether the Bridge's DID is listed in the *OAP Trusted Service Registry* and if the stored certificate is valid.
3.  **Signed Initiation:** The Bridge signs the API call to the bank with its eIDAS key, not the User Key.

### 4.2 Payload & Encryption (Privacy)

Sensitive banking data (IBAN, account logins, TANs) in the `methodData` field **MUST NEVER** be sent in cleartext through the OATP network, even if the transport channel is encrypted. The principle of Defense-in-Depth applies.

#### 4.2.1 Normative JWE Encryption
The `accountData` field within the `PaymentAuthorization` **MUST** be encrypted as a **JWE Compact String** (according to RFC 7516).

*   **Algorithm (`enc`):** **A256GCM** (AES-256-GCM).
*   **Key Management (`alg`):** **ECDH-ES** (Ephemeral Static Diffie-Hellman). The sender uses the PSD2 Bridge's public key (from its DID Document Property `keyAgreement`) to derive the Content Encryption Key (CEK).
*   **Effect:** Only the Bridge (holder of the private key) can decrypt the IBAN. No relay or other intermediary has access.

```json
// Example methodData for PSD2_SEPA
"methodData": {
  "bankId": "BIC_OF_BANK", // Unencrypted for routing
  "remittanceInformation": "Order #123",
  // Encrypted container for IBAN/Login
  "accountData": "eyJhbGciOiJFQ0RILUVTIiwiZW5jIjoiQTI1NkdDTSJ9..." 
}
```

### 4.3 Process Flow

1.  **Init:** The Payer Agent sends the `PaymentAuthorization` via OATP to the PSD2 Bridge.
2.  **Setup:** The Bridge decrypts the `accountData` and initiates a *Payment Initiation Request* with the API of the user's bank.
3.  **SCA (Strong Customer Authentication):** The bank requests Two-Factor Authentication (e.g., Push notification on the user's banking app). This step happens "Out-of-Band".
4.  **Execution:** Upon successful SCA, the bank executes the SEPA transfer.
5.  **Receipt:** The bank confirms execution to the Bridge. The Bridge generates the OAPP `PaymentReceipt`, signs it, and sends it via OATP to the Payer and the Payee (Merchant).

### 4.4 Error Mapping
The Bridge MUST map Bank errors to OAPP statuses:
*   *SCA cancelled/timeout* $\rightarrow$ `FAILED`
*   *Insufficient Funds* $\rightarrow$ `FAILED` (with Reason Code)
*   *Technical Bank Issue* $\rightarrow$ `PENDING` (Retry possible)

---

## 5. Scenario C: PSP Integration (Stripe, PayPal)

To accept Credit Cards, Wallets (Apple Pay/Google Pay), or PayPal, OAPP utilizes the APIs of established Payment Service Providers (PSPs). The Agent acts as an intelligent orchestrator here, encapsulating the legacy payment into the OAPP format.

### 5.1 Architecture: "Agent-Mediated Checkout"

Since PSPs often require JavaScript SDKs or Web Redirects (and AI Agents are not browsers), OAPP supports two integration modes:

1.  **Headless (Server-Side):** If the Agent already has a tokenized payment method (e.g., Stripe Customer ID `cus_...`), it can call the API directly.
2.  **Handoff (Interactive):** The Payer Agent presents the user with a native UI element or a secure Webview to enter card data.

**The Flow (Example: Stripe PaymentIntents):**
1.  **Request:** The Merchant Agent creates a `PaymentIntent` server-side and sends the `client_secret` in the `PaymentRequest` (Field `gatewayData`).
2.  **Authorization:** The Payer Agent uses the Secret to confirm the payment with the PSP (via SDK or API).
3.  **Receipt Generation:** The Merchant Agent waits for confirmation from the PSP (Webhook) and only then issues the OAPP `PaymentReceipt`.

### 5.2 PSP Webhook Security (Normative)

Since the final confirmation ("Money received") occurs asynchronously via a PSP Webhook to the Merchant Agent, this endpoint is a critical attack vector. An attacker could send fake "Success" webhooks to receive goods without payment.

Implementations **MUST** perform the following security checks:

#### 5.2.1 Signature Validation
The Merchant Agent **MUST** validate the cryptographic signature of the webhook.
*   **Mechanism:** Calculation of an HMAC (usually SHA-256) over the Raw Body of the webhook using the *Webhook Secret* provided by the PSP (e.g., `whsec_...`).
*   **Check:** The calculated hash must match the Header (e.g., `Stripe-Signature` or `Paypal-Transmission-Sig`).
*   **Failure:** If there is a mismatch, the request **MUST** be rejected with HTTP 400 and ignored.

#### 5.2.2 Idempotency & Replay Protection
Webhooks can be sent multiple times (PSP Retry) or recorded and replayed by attackers.
1.  **Event-ID Cache:** The Agent **MUST** store the unique ID of the webhook event (e.g., `evt_123...`).
2.  **Check:** Is the ID already known?
    *   *Yes:* Reply immediately with `200 OK`, but do NOT execute business logic (Idempotency).
    *   *No:* Process the event.
3.  **Time Window:** The Agent **SHOULD** check the timestamp in the webhook header. Events older than **5 minutes** must be discarded as potential replay attacks (unless the PSP covers this via the signature).

#### 5.2.3 State Consistency (Race Conditions)
A `payment.succeeded` event may only be processed if the corresponding OACP Order is still in a valid state (e.g., `LOCKED`).
*   If the Order is already `CANCELLED` or `TIMED_OUT`, the Agent must automatically initiate a **Refund** at the PSP, as the goods are no longer reserved.

### 5.3 Mapping to OAPP Receipt
After successful validation of the webhook, the Merchant Agent generates the `PaymentReceipt`:

```json
{
  "type": "PaymentReceipt",
  "status": "SUCCESS",
  "transactionId": "pi_3Mem...", // The PSP PaymentIntent ID
  "gateway": "STRIPE",
  "fee": "299" // The fee charged by the PSP (Minor Units)
}
```

---

## 6. State Machine

The states of an OAPP transaction model the asynchrony of banks and blockchains. To avoid race conditions, transitions are strictly defined.

### 6.1 OAPP States

| Status | Description | Next Step / Trigger |
| :--- | :--- | :--- |
| **CREATED** | Request created locally, not yet sent. | Send via OATP $\rightarrow$ `PENDING` |
| **PENDING** | Sent to Payer. Waiting for authorization. | User Signature $\rightarrow$ `AUTHORIZED` |
| **AUTHORIZED** | `PaymentAuthorization` is present. | Submit to Ledger/Bank $\rightarrow$ `PROCESSING` |
| **SCA_REQUIRED** | (PSD2 only) Bank waiting for 2FA. | User confirms in Bank App $\rightarrow$ `PROCESSING` |
| **PROCESSING** | Technical execution in progress (Mining, Batch). | Success $\rightarrow$ `SETTLED`<br>Error $\rightarrow$ `FAILED`<br>Escrow $\rightarrow$ `ESCROW_LOCKED` |
| **ESCROW_LOCKED**| Funds held by Trustee. | Release Command $\rightarrow$ `SETTLED`<br>Refund Command $\rightarrow$ `REFUNDED` |
| **SETTLED** | Funds have finally arrived at recipient. | (Final State) |
| **FAILED** | Error (Insufficient funds, Tech error, Timeout). | Retry or Abort $\rightarrow$ (Final State) |
| **REFUNDED** | Refund successfully executed. | (Final State) |

### 6.2 Mapping to OACP (Order Status)

To inform the trade workflow (OACP) when to proceed, OAPP defines normative mappings to OACP states.

| OAPP Status | Recommended OACP Order Status | Meaning for the Trade |
| :--- | :--- | :--- |
| `PENDING` | `LOCKED` | Order is reserved, payment pending. |
| `AUTHORIZED` | `LOCKED` (Payment Processing) | Payment is en route, goods remain reserved. |
| `ESCROW_LOCKED` | `PAID` (or `ESCROWED`) | Funds are secured. **Shipping can proceed.** |
| `SETTLED` | `PAID` | Funds are with the merchant. **Shipping can proceed.** |
| `FAILED` | `PAYMENT_FAILED` | Merchant cancels reservation (after Retry). |
| `REFUNDED` | `CANCELLED` | Order has been reversed. |

**Important:** For the Merchant, the status `ESCROW_LOCKED` is equivalent to `PAID` regarding shipping safety. The funds are not yet in their account, but have left the Payer and are secured by the Smart Contract or Trustee. They can ship safely.

---

## 7. Security Considerations

This section consolidates security mechanisms for all three scenarios and normatively defines **Replay Protection** and **Separation of Duties** to prevent fraud at the protocol level.

### 7.1 Separation of Concerns (Architectural Security)
The OAP framework enforces a strict separation of competencies:
*   The **OACP** protocol manages the *Contract* (Cart, Delivery Terms).
*   The **OAPP** protocol manages the *Money*.
*   **Rule:** A Merchant Agent (OACP) must never have direct access to the user's wallet keys. A payment always requires a dedicated `PaymentAuthorization`, separately signed by the user (or their Payer Agent).

### 7.2 PSD2 Bridge & Data Encryption
For Scenario B (Open Banking), stricter rules apply as IBANs and credentials are processed.
*   **Defense in Depth:** Even if the OATP transport channel is encrypted, sensitive banking data (`methodData`) MUST additionally be encrypted via JWE (see 4.2.1), so only the certified PSD2 Bridge can read it.
*   **Tokenization:** Where possible, Agents SHOULD only store temporary access tokens (AIS/PIS Tokens), never the user's online banking credentials (PIN/TAN).

### 7.3 Replay Protection & Idempotency

#### 7.3.1 Ledger Replay Cache (Normative)
The OAP Ledger MUST implement a **Replay Cache** to prevent Double-Spending via repeated sending of the same signed message.

*   **Cache Key:** `SHA256(paymentRequestId || payerDID)`.
*   **Time Window:** The window is **5 Minutes**.
*   **Validation Logic:**
    1.  **Timestamp Check:** Is `created` in `authorizationProof` older than 5 minutes?
        *   *Yes:* **Reject** (`422 EXPIRED_REQUEST`).
    2.  **Cache Lookup:** Is the Key in the Cache?
        *   *Yes:* **Idempotency Case.** Return the result of the original transaction (`200 OK` with old `txId`), but do NOT execute a new booking.
    3.  **New Transaction:** Execute booking and store Key in Cache (with TTL 5 min).

### 7.4 Separation of Duties (Fraud Prevention)
To prevent collusion in the Escrow case:
*   **Rule:** The **Escrow Agent** MUST NEVER be identical to the **Payee** (Merchant). The DIDs must be different and ideally belong to different legal entities.
*   **Client Check:** The Payer Agent SHOULD warn the user if `beneficiary` and `escrow.agent` use the same domain or root identity.

### 7.5 Hash Agility vs. Banking Standards
While OAEP favors modern algorithms like **BLAKE3**, external banking APIs often require **SHA-256**.
*   **Internal (OAP Ledger):** Signatures MUST follow the active OAEP Cipher Suite (currently BLAKE3).
*   **External (PSD2/PSP):** For communication with legacy systems (e.g., Webhook validation), the standards mandated there (mostly HMAC-SHA256) MUST be used. Implementations must handle this context switch cleanly.

---

## 8. JSON Schema Definitions (Normative)

To ensure interoperability, implementations MUST validate incoming OAPP messages against the following JSON Schemas (Draft 07).

### 8.1 `PaymentRequest` Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "https://w3id.org/oapp/v1/PaymentRequest",
  "type": "object",
  "required": ["type", "id", "amount", "currency", "beneficiary"],
  "properties": {
    "type": { "const": "PaymentRequest" },
    "id": { "type": "string", "format": "uri" },
    "amount": { "type": "string", "pattern": "^[0-9]+$" },
    "currency": { "type": "string", "minLength": 3, "maxLength": 4 },
    "beneficiary": {
      "type": "object",
      "required": ["did"],
      "properties": {
        "did": { "type": "string", "format": "uri" }
      }
    },
    "escrow": {
      "type": "object",
      "required": ["agent"],
      "properties": {
        "agent": { "type": "string", "format": "uri" }
      }
    }
  }
}
```

### 8.2 `PaymentAuthorization` Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "https://w3id.org/oapp/v1/PaymentAuthorization",
  "type": "object",
  "required": ["type", "paymentRequestId", "selectedMethod", "authorizationProof"],
  "properties": {
    "type": { "const": "PaymentAuthorization" },
    "paymentRequestId": { "type": "string", "format": "uri" },
    "selectedMethod": { "enum": ["OAP_LEDGER", "PSD2_SEPA", "PSP_STRIPE"] },
    "authorizationProof": {
      "type": "object",
      "required": ["type", "created", "signedHash", "nonce", "signatureValue"],
      "properties": {
        "type": { "const": "OaepSignature2025" },
        "created": { "type": "string", "format": "date-time" },
        "signedHash": { "type": "string" },
        "nonce": { "type": "string" },
        "signatureValue": { "type": "string" }
      }
    }
  }
}
```

### 8.3 `PaymentReceipt` Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "https://w3id.org/oapp/v1/PaymentReceipt",
  "type": "object",
  "required": ["type", "paymentRequestId", "status", "transactionId"],
  "properties": {
    "type": { "const": "PaymentReceipt" },
    "status": { "enum": ["SUCCESS", "PENDING", "FAILED"] },
    "transactionId": { "type": "string" },
    "fee": { "type": "string", "pattern": "^[0-9]+$" }
  }
}
```

---

## 9. Appendix (Informative)

This section contains examples and assistance for implementers.

### 9.1 Reference Implementations

The OAP Foundation provides open-source libraries implementing this standard:

*   **Core Logic (Rust):** `oap-ledger-rs`
    *   Contains normative logic for signature validation, state machine, and ledger functions.
*   **PHP SDK:** `oapp-php`
    *   A reference implementation for merchant systems (e.g., WooCommerce, Magento Plugins).
*   **Dart/Flutter SDK:** `oapp-dart`
    *   For integration into Wallet Apps and Mobile Clients.

### 9.2 Complete JSON Example (Scenario A: OAP-Ledger)

Below is a complete trace of a transaction ("Purchase of an E-Book for 12.99 OAP").

#### A. PaymentRequest (from Merchant)
```json
{
  "@context": ["https://w3id.org/oapp/v1"],
  "type": "PaymentRequest",
  "id": "urn:uuid:req-550e8400-e29b-41d4-a716-446655440000",
  "threadId": "urn:uuid:thread-98765432-1234-5678-9abc-def012345678",
  "amount": "1299",
  "currency": "OAP",
  "description": "E-Book: The Future of AI",
  "beneficiary": {
    "did": "did:web:books.example.com",
    "name": "Example Books Ltd."
  },
  "supportedMethods": [
    { "type": "OAP_LEDGER", "targetAccount": "did:web:books.example.com" }
  ],
  "expiresAt": "2025-12-02T16:00:00Z"
}
```

#### B. PaymentAuthorization (from Buyer)
```json
{
  "type": "PaymentAuthorization",
  "paymentRequestId": "urn:uuid:req-550e8400-e29b-41d4-a716-446655440000",
  "selectedMethod": "OAP_LEDGER",
  "authorizationProof": {
    "type": "OaepSignature2025",
    "created": "2025-12-02T15:30:00Z",
    "algorithm": "Ed25519",
    "digest": "BLAKE3",
    // Base64URL( Timestamp(4 bytes) || Random(16 bytes) )
    "nonce": "AAABjPz8_30Qh3_z8_30Qh3_z8_30Qh3",
    "signedHash": "K7gNU3kbf...", 
    "signatureValue": "OpP..."
  }
}
```

#### C. PaymentReceipt (from Ledger via Merchant)
```json
{
  "type": "PaymentReceipt",
  "paymentRequestId": "urn:uuid:req-550e8400-e29b-41d4-a716-446655440000",
  "status": "SUCCESS",
  "transactionId": "oap:tx:8374928374928374928374",
  "settledAt": "2025-12-02T15:30:02Z",
  "fee": "0"
}
```

### 9.3 Test Vector for Signature Creation

To ensure different implementations sign the same hash, the following canonical input is defined.

**Input Parameters:**
*   `paymentRequestId`: `urn:uuid:test-123`
*   `amount`: `100`
*   `currency`: `EUR`
*   `beneficiaryDID`: `did:web:test.com`
*   `nonce`: `testnonce`

**Canonical String (JCS):**
`{"amount":"100","beneficiary":"did:web:test.com","currency":"EUR","nonce":"testnonce","requestId":"urn:uuid:test-123"}`

**Expected Hash (BLAKE3):**
*   Hex: `b4c5...` *(Note: This value must be verified in the reference implementation)*.

### 9.4 Mapping Table for External Status

Guidance for implementing adapters (Scenario B and C).

| OAPP Status | PSD2 (Sepa) Status | Stripe PaymentIntent Status |
| :--- | :--- | :--- |
| `PENDING` | `RCVD` (Received) | `requires_payment_method` |
| `AUTHORIZED` | `ACCP` (Accepted) | `processing` |
| `SCA_REQUIRED` | `PDNG` (Pending SCA) | `requires_action` |
| `SETTLED` | `ACSC` (Accepted Settlement Completed) | `succeeded` |
| `FAILED` | `RJCT` (Rejected) | `canceled` / `requires_payment_method` |