# Open Agent Payment Protocol (OAPP)

[![Spec Version](https://img.shields.io/badge/spec-v1.0--RC-blue)](./RFC%20OAPP%20v1.0-RC.md)
[![Layer](https://img.shields.io/badge/OAP-Layer%202-green)](https://oap.foundation)
[![Status](https://img.shields.io/badge/status-CODE%20FREEZE-snowflake)](./RFC%20OAPP%20v1.0-RC.md)
[![License](https://img.shields.io/badge/license-MIT%2Fwm-green)](LICENSE)

> **⚠️ STATUS ALERT: CODE FREEZE**
>
> This specification is currently a **Release Candidate (v1.0-RC)**.
> We are in **Code Freeze**. No new features will be added. Feedback is strictly limited to security audits, financial compliance (PSD2/eIDAS) checks, and state machine consistency.

## 💳 Introduction

The **Open Agent Payment Protocol (OAPP)** acts as the **Settlement Layer (Layer 2)** of the Open Agent Protocol framework. It translates the commercial intent defined in OACP (the contract) into irrevocable value transfer.

OAPP functions as a universal adapter, decoupling the **Authorization** of a payment (by the user's agent) from the **Technical Execution** (by Ledgers, Banks, or Gateways). This allows autonomous agents to operate seamlessly across modern token economies and traditional fiat systems.

### Core Capabilities
*   **Universal Settlement:** Supports Native OAP Credits, SEPA Instant (via PSD2), and Credit Cards (via PSPs).
*   **Trustless Escrow:** Native support for conditional payments ("Money is locked until delivery is confirmed").
*   **Compliance Ready:** Integrates eIDAS requirements for Open Banking.
*   **Defense-in-Depth:** Additional JWE encryption for sensitive banking data (IBANs), even inside the encrypted transport tunnel.

## 🏗 Architecture

OAPP sits on top of the transport layer and serves the commerce layer.

```text
[      OACP (Commerce)     ]  <-- "I want to buy X for 10 EUR"
             |
[   OAPP (Settlement) L2   ]  <-- "I authorize moving 10 EUR to IBAN Y"
             |
[   OATP (Transport)  L1   ]  <-- Encrypted Logistics
             |
[      OAEP (Trust)   L0   ]  <-- Identity & Signatures
```

## 📂 The Specification

The full normative specification is available here:

👉 **[READ THE SPECIFICATION (v1.0-RC)](RFC%20OAPP%20v1.0-RC.md)**

### Supported Scenarios
OAPP v1.0 defines three normative settlement paths:

1.  **[Internal Economy (OAP Ledger)](RFC%20OAPP%20v1.0-RC.md#3-szenario-a-oap-ledger-api-native-token)**
    *   Direct P2P transactions using DIDs.
    *   Instant settlement with 0-conf possibilities.
    *   Native Escrow support.

2.  **[Open Banking (PSD2)](RFC%20OAPP%20v1.0-RC.md#4-szenario-b-psd2--open-banking-der-bank-adapter)**
    *   Bridges to SEPA Instant via certified TPPs.
    *   Uses JWE/ECDH-ES to encrypt banking credentials directly to the bridge (Bypassing relays).

3.  **[Legacy Commerce (PSP)](RFC%20OAPP%20v1.0-RC.md#5-szenario-c-psp-integration-stripe-paypal)**
    *   Wraps Stripe/PayPal flows into agent-readable objects.
    *   Handles webhook security and state synchronization.

## ⚡ Technical Standards

Implementers must adhere to these financial security primitives:

| Component | Standard / Requirement |
| :--- | :--- |
| **Data Format** | JSON-LD (Schema Draft 07) |
| **Signatures** | **Ed25519** (linked to OAEP Identity) |
| **Sensitive Data** | **JWE** (A256GCM) for IBANs/Credentials |
| **Replay Protection** | 5-Minute Window + Nonce Caching |
| **Idempotency** | Strictly required for all financial endpoints |

### The Payment Flow (Simplified)
1.  **Request:** Payee sends `PaymentRequest` (Amount, Currency, Methods).
2.  **Auth:** Payer signs `PaymentAuthorization` (Nonce, Hash).
3.  **Execution:** Payee submits Auth to Ledger/Bank.
4.  **Receipt:** Ledger/Bank issues `PaymentReceipt` (Finality).

## 🛠 Reference Implementations

To ensure financial safety, please use the official libraries for signature verification and state management.

*   **Core Logic (Rust):** [`oap-foundation/oap-ledger-rs`](https://github.com/oap-foundation/oap-ledger-rs)
    *   *Contains the normative state machine and signature validation logic.*
*   **Merchant SDK (PHP):** [`oap-foundation/oapp-php`](https://github.com/oap-foundation/oapp-php)
    *   *For WooCommerce/Magento integration.*
*   **Wallet SDK (Dart):** [`oap-foundation/oapp-dart`](https://github.com/oap-foundation/oapp-dart)
    *   *For building Payer Agents.*

## 🧪 Conformance & Testing

Implementers must validate their JSON structures against the provided schemas.

*   **JSON Schemas:** See [Section 8](RFC%20OAPP%20v1.0-RC.md#8-json-schema-definitionen-normativ).
*   **Test Vectors:** See [Section 9.3](RFC%20OAPP%20v1.0-RC.md#93-test-vektor-für-die-signatur-erstellung) for canonical signature hashing.

## 🤝 Contributing

We are currently in **Code Freeze**.
*   **Accepted:** Security vulnerabilities (e.g., Replay Attacks), PSD2/Regulatory compliance issues, State Machine deadlocks.
*   **Not Accepted:** New payment methods (e.g., Lightning Network) are deferred to v1.1.

Please review `CONTRIBUTING.md` before submitting PRs.

## 📄 License & Legal

### Specification License (Copyleft)
The specification text, architecture definitions, and protocol logic contained in this repository are licensed under the **Creative Commons Attribution-ShareAlike 4.0 International License (CC BY-SA 4.0)**.

[![License: CC BY-SA 4.0](https://img.shields.io/badge/License-CC%20BY--SA%204.0-lightgrey.svg)](https://creativecommons.org/licenses/by-sa/4.0/)

**Intent of this License:**
The goal of using CC BY-SA 4.0 is to permanently protect the open nature of this standard.
*   **ShareAlike:** If you modify, extend, or build upon this specification (e.g., creating a "Layer 2.5"), you **must** distribute your contributions under the same **CC BY-SA 4.0** license.
*   **No Proprietary Forks:** It is legally prohibited to create a proprietary, closed-source version or extension of this specification text. All derivatives must remain free and open to the community.

### Note on Implementation
To facilitate broad adoption, the use of the concepts, data structures (JSON-LD), and logic defined in this specification to create **software implementations** (libraries, applications, agents) is permitted without triggering the ShareAlike clause for the software itself.

However, any changes to the **specification document itself** remain subject to the ShareAlike requirement.

---
**Maintained by the OAP Foundation**
*Building the settlement layer for the agent economy.*
