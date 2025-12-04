# Open Agent Commerce Protocol (OACP)

[![Spec Version](https://img.shields.io/badge/spec-v1.0--RC-blue)](./RFC%20OACP%20v1.0-RC.md)
[![Layer](https://img.shields.io/badge/OAP-Layer%202-purple)](https://oap.foundation)
[![Status](https://img.shields.io/badge/status-CODE%20FREEZE-snowflake)](./RFC%20OACP%20v1.0-RC.md)
[![License](https://img.shields.io/badge/license-MIT%2Fwm-green)](LICENSE)

> **⚠️ STATUS ALERT: CODE FREEZE**
>
> This specification is currently a **Release Candidate (v1.0-RC)**.
> We are in **Code Freeze**. No new features will be added. Feedback is currently limited to semantic correctness (Schema.org mapping), state machine deadlocks, and integration with OAPP v1.0.

## 🛍️ Introduction

The **Open Agent Commerce Protocol (OACP)** represents the **Commerce Layer (Layer 2)** of the Open Agent Protocol framework. It creates a standardized language for autonomous agents to search, negotiate, and contract for goods and services.

OACP shifts e-commerce from a *Catalog-Centric* model ("Search for what is indexed") to an **Intent-Centric** model ("Describe what you need"). It enables a decentralized economy where Buyer Agents and Merchant Agents interact directly, without central marketplaces.

### Core Capabilities
*   **Semantic Discovery:** Uses **Schema.org** and JSON-LD to describe products and needs (Intents) unambiguously.
*   **AI Safety:** Mandatory **UserProof** (cryptographic signatures) for orders to prevent AI agents from hallucinating purchases.
*   **Verifiable Trust:** Integration of W3C Verifiable Credentials to prove product claims (e.g., "Certified Organic").
*   **Escrow Native:** Built-in states for conditional payments and dispute resolution.

## 🏗 Architecture

OACP handles the business logic, delegating transport and settlement to lower layers.

```text
[      OACP (Commerce)     ]  <-- "I need a laptop < 2000 EUR" (Intent)
             |
[   OAPP (Settlement) L2   ]  <-- "Transferring Funds to Escrow"
             |
[   OATP (Transport)  L1   ]  <-- Encrypted / Sharded Delivery
             |
[      OAEP (Trust)   L0   ]  <-- Identity / Signatures
```

## 📂 The Specification

The full normative specification is available here:

👉 **[READ THE SPECIFICATION (v1.0-RC)](RFC%20OACP%20v1.0-RC.md)**

### The Commerce Flow
OACP defines a strict state machine for the trade lifecycle:

1.  **Negotiation:** Buyer broadcasts an `Intent` with constraints (e.g., `price < 100`). Merchant responds with a binding `Offer`.
2.  **Commitment:** Buyer signs an `OrderRequest` including a `UserProof` (protecting against rogue AI).
3.  **Settlement:** The flow hands over to **[OAPP v1.0](https://github.com/oap-foundation/oapp-spec)** for payment processing.
4.  **Fulfillment:** Merchant delivers; Buyer confirms; Escrow releases funds.
5.  **Dispute:** Optional arbitration path if the delivery fails.

## ⚡ Technical Standards

Implementers must strictly adhere to the following primitives:

| Component | Standard / Requirement |
| :--- | :--- |
| **Vocabulary** | **Schema.org** (Product, Offer, Service) |
| **Data Format** | JSON-LD |
| **Constraint Logic** | Normative operators (`equals`, `lessThan`, `inRange`) |
| **Trust** | W3C Verifiable Credentials (VCs) |
| **Signatures** | **Ed25519** (OAEP Suite) over JCS Canonicalization |

### Example: Negotiate Request (Intent)
```json
{
  "type": "NegotiateRequest",
  "intent": {
    "@type": "Product",
    "category": "RunningShoes"
  },
  "constraints": [
    { "property": "schema:size", "operator": "equals", "value": "42" },
    { "property": "schema:price", "operator": "lessThan", "value": "150" }
  ],
  "requiredCredentials": ["EcoFriendlyCertified"] 
}
```

## 🛠 Reference Implementations

To ensure semantic interoperability and correct state management, please use the official core libraries.

*   **Core Logic (Rust):** [`oap-foundation/oacp-core-rs`](https://github.com/oap-foundation/oacp-core-rs)
    *   *Includes the State Machine, Constraint Validator, and Schema.org types.*
*   **Merchant Integration:** `oacp-shopify-adapter` (Coming soon)
*   **Buyer Agent SDK:** `oacp-agent-js` (Coming soon)

## 🧪 Conformance & Validation

All messages must be validated against the JSON Schemas defined in **[Section 7](RFC%20OACP%20v1.0-RC.md#7-json-schema-normativ)** of the RFC.

**Important:** The **UserProof** mechanism (Section 3.2.1) is critical. Implementations that allow orders without a valid human signature will be rejected by compliant Merchant Agents.

## 🤝 Contributing

We are currently in **Code Freeze**.
*   **Accepted:** Fixes for logical loopholes in the Dispute Resolution flow, JSON Schema errors, or inconsistencies with OAPP v1.0.
*   **Not Accepted:** New constraint operators or alternative vocabularies (e.g., GS1) are deferred to v1.1.

Please review `CONTRIBUTING.md` before submitting PRs.

## 📄 License

Specification text: **CC BY 4.0 International**.
Schemas and Code samples: **MIT**.

---
**Maintained by the OAP Foundation**
*Enabling the intent-centric agent economy.*