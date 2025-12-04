# Open Agent Data Protocol (OADP)

[![Spec Version](https://img.shields.io/badge/spec-v1.0--RC-blue)](./RFC%20OADP%20v1.0-RC.md)
[![Layer](https://img.shields.io/badge/OAP-Layer%202-indigo)](https://oap.foundation)
[![Status](https://img.shields.io/badge/status-CODE%20FREEZE-snowflake)](./RFC%20OADP%20v1.0-RC.md)
[![License](https://img.shields.io/badge/license-MIT%2Fwm-green)](LICENSE)

> **⚠️ STATUS ALERT: CODE FREEZE**
>
> This specification is currently a **Release Candidate (v1.0-RC)**.
> We are in **Code Freeze**. No new features will be added. Feedback is limited to ODRL policy logic, JWE encryption vectors, and schema validation.

## 💾 Introduction

The **Open Agent Data Protocol (OADP)** is the **Data Layer (Layer 2)** of the Open Agent Protocol framework. It establishes a standard for the sovereign exchange, licensing, and monetization of structured datasets.

Unlike OAFP (which focuses on social feeds), OADP is designed for **High-Value Data**: Personal Data Vaults (Medical records, Financial history), IoT Telemetry, and AI Training Sets. It enables a "Data Economy" where users retain ownership of their data and grant access only under cryptographic enforcement.

### Core Capabilities
*   **Self-Describing Data:** Uses **Data Manifests** to describe schema, origin, and integrity without revealing the actual content.
*   **Granular Consent:** Integrates **W3C ODRL** (Open Digital Rights Language) to define machine-readable access policies (e.g., "Read-only access for 5 EUR").
*   **Encrypted by Default:** Data blobs are always stored and transported as **JWE** (JSON Web Encryption) containers.
*   **Monetization Native:** Seamless integration with **OAPP** for pay-to-access data streams.

## 🏗 Architecture

OADP manages the metadata and rights, delegating the physical transfer and payment to lower layers.

```text
[       OADP (Data)        ]  <-- "Data Manifest / Access Policy (ODRL)"
             |
[   OAPP (Economics)  L2   ]  <-- "Pay 5 EUR for Decryption Key"
             |
[   OATP (Transport)  L1   ]  <-- Encrypted Blob Transfer (IPFS/Direct)
             |
[      OAEP (Trust)   L0   ]  <-- Owner Identity
```

## 📂 The Specification

The full normative specification is available here:

👉 **[READ THE SPECIFICATION (v1.0-RC)](RFC%20OADP%20v1.0-RC.md)**

### Supported Flows
*   **[Discovery](RFC%20OADP%20v1.0-RC.md#phase-1-discovery-dataannouncement):** announcing datasets via `DataManifests`.
*   **[Negotiation](RFC%20OADP%20v1.0-RC.md#phase-2-negotiation-accessrequest):** Requesting access based on specific claims (VCs) or payments.
*   **[Fulfillment](RFC%20OADP%20v1.0-RC.md#phase-3-fulfillment-keydelivery):** Secure delivery of the Content Encryption Key (CEK) via JWE Key Wrapping.

## ⚡ Technical Standards

Implementers must strictly adhere to these primitives:

| Component | Standard / Requirement |
| :--- | :--- |
| **Vocabulary** | **Schema.org** (Dataset, MedicalRecord, etc.) |
| **Rights Language** | **W3C ODRL** (Offer, Permission, Duty) |
| **Encryption** | **JWE** (A256GCM for content, ECDH-ES for keys) |
| **Integrity** | **SHA-256** Digests linked to Manifests |

### Example: Data Manifest
```json
{
  "type": "DataManifest",
  "id": "urn:uuid:data-123",
  "datasetSchema": "https://schema.org/MedicalRecord",
  "location": {
    "uri": "ipfs://QmHash...",
    "digest": "SHA-256=..."
  },
  "encryption": {
    "algorithm": "A256GCM",
    "keyId": "key-1"
  },
  "accessPolicy": {
    "type": "odrl:Offer",
    "permission": [{
      "action": "odrl:read",
      "duty": [{
        "action": "odrl:compensate",
        "constraint": [{
          "leftOperand": "payAmount",
          "operator": "eq",
          "rightOperand": "5.00 EUR"
        }]
      }]
    }]
  }
}
```

## 🛠 Reference Implementations

*   **Core Logic (Rust):** [`oap-foundation/oadp-core-rs`](https://github.com/oap-foundation/oadp-core-rs)
    *   *Includes the ODRL Policy Engine, JWE Wrapper, and Manifest Builder.*
*   **Data Vault:** `oadp-personal-vault` (Reference Storage Server)
*   **Marketplace Adapter:** `oadp-market-js` (SDK for Web UI)

## 🧪 Conformance & Validation

All messages must be validated against the JSON Schemas defined in **[Section 7](RFC%20OADP%20v1.0-RC.md#7-json-schema-definitionen-normativ)** of the RFC.

**Important:** The **JWE Encryption** requirement (Section 2.2) is normative. Implementations transmitting plain-text data blobs over OADP (even if the transport is TLS) are considered non-compliant and insecure.

## 🤝 Contributing

We are currently in **Code Freeze**.
*   **Accepted:** Fixes for ODRL policy evaluation edge cases, JWE key wrapping interoperability, or schema bugs.
*   **Not Accepted:** New encryption algorithms or alternative policy languages are deferred to v1.1.

Please review `CONTRIBUTING.md` before submitting PRs.

## 📄 License

Specification text: **CC BY 4.0 International**.
Schemas and Code samples: **MIT**.

---
**Maintained by the OAP Foundation**
*Unlock the value of sovereign data.*