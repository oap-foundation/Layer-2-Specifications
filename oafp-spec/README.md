# Open Agent Feed Protocol (OAFP)

[![Spec Version](https://img.shields.io/badge/spec-v1.0--RC-blue)](./RFC%20OAFP%20v1.0-RC.md)
[![Layer](https://img.shields.io/badge/OAP-Layer%202-orange)](https://oap.foundation)
[![Status](https://img.shields.io/badge/status-CODE%20FREEZE-snowflake)](./RFC%20OAFP%20v1.0-RC.md)
[![License](https://img.shields.io/badge/license-MIT%2Fwm-green)](LICENSE)

> **⚠️ STATUS ALERT: CODE FREEZE**
>
> This specification is currently a **Release Candidate (v1.0-RC)**.
> We are in **Code Freeze**. No new features will be added. Feedback is limited to CAS integration (IPFS/Hypercore), C2PA binding, and Anti-Spam mechanisms.

## 📰 Introduction

The **Open Agent Feed Protocol (OAFP)** is the **Social Layer (Layer 2)** of the Open Agent Protocol framework. It re-imagines social media not as a centralized platform, but as a protocol for sovereign content distribution.

OAFP shifts power from the platform ("The Algorithm decides what you see") to the user ("Your Personal AI curates for you"). It enables a censorship-resistant, verifiable public stage where creators own their audience.

### Core Philosophy
*   **Algorithmic Sovereignty:** Your feed is compiled locally by your agent, based on *your* interests and trust graph, not optimized for engagement or ad-revenue.
*   **Content Provenance:** Every post is cryptographically signed (OAEP) and optionally bound to C2PA credentials to fight deepfakes.
*   **Earn & Burn:** Uses economic stakes (OAPP) to prevent spam and enable direct monetization (Pay-to-View).
*   **Storage Agnostic:** Content lives on IPFS, Arweave, or Hypercore. OAFP handles the indexing and distribution.

## 🏗 Architecture

OAFP separates storage, indexing, and curation.

```text
[      OAFP (Social)       ]  <-- "Post Manifest / Like / Share"
             |
[   OAPP (Economics)  L2   ]  <-- "Burn Credits to Post" / "Unlock Content"
             |
[   OATP (Transport)  L1   ]  <-- PubSub / Encrypted Delivery
             |
[      OAEP (Trust)   L0   ]  <-- Author Identity
```

## 📂 The Specification

The full normative specification is available here:

👉 **[READ THE SPECIFICATION (v1.0-RC)](RFC%20OAFP%20v1.0-RC.md)**

### Supported Objects
*   **[ContentManifest](RFC%20OAFP%20v1.0-RC.md#21-contentmanifest-der-beitrag):** The core post object linking to CAS blobs.
*   **[Interaction](RFC%20OAFP%20v1.0-RC.md#22-interaction-die-reaktion):** Likes, Comments, Shares (stored in user's repo).
*   **[Tombstone](RFC%20OAFP%20v1.0-RC.md#23-tombstone-das-lösch-signal):** Signal for deletion ("Right to be Forgotten").
*   **[KeyDelivery](RFC%20OAFP%20v1.0-RC.md#24-keydelivery-pay-to-view-unlock):** Secure transport for premium content keys.

## ⚡ Technical Standards

Implementers must strictly adhere to these primitives:

| Component | Standard / Requirement |
| :--- | :--- |
| **Vocabulary** | **Schema.org** (SocialMediaPosting, Article) |
| **Data Format** | JSON-LD |
| **Storage** | **IPFS** (Public) / **OATP** (Private) |
| **Provenance** | **C2PA** (Content Credentials) + OAEP Sig |
| **Anti-Spam** | **Proof-of-Burn** (OAPP Transaction) |

### Example: Content Manifest
```json
{
  "type": "ContentManifest",
  "author": "did:web:journalist.example.com",
  "content": {
    "uri": "ipfs://QmHash...",
    "mimeType": "text/markdown"
  },
  "proofOfBurn": {
    "amount": "10",
    "txId": "oap:tx:..."
  },
  "proof": { ... }
}
```

## 🛠 Reference Implementations

*   **Core Logic (Rust):** [`oap-foundation/oafp-core-rs`](https://github.com/oap-foundation/oafp-core-rs)
    *   *Includes Manifest Builders, Validation Logic, and Spam Filters.*
*   **Storage Adapter:** `oafp-ipfs-adapter` (Beta)
*   **Client SDK:** `oafp-feed-js` (Beta)

## 🧪 Conformance & Validation

All messages must be validated against the JSON Schemas defined in **[Section 7](RFC%20OAFP%20v1.0-RC.md#7-json-schema-definitionen-normativ)** of the RFC.

**Important:** The **Proof of Burn** validation is mandatory for public relays. Messages without a valid OAPP transaction reference must be dropped to prevent network flooding.

## 🤝 Contributing

We are currently in **Code Freeze**.
*   **Accepted:** Fixes for CAS resolution logic, C2PA integration bugs, or schema validation errors.
*   **Not Accepted:** New interaction types or storage backends are deferred to v1.1.

Please review `CONTRIBUTING.md` before submitting PRs.

## 📄 License

Specification text: **CC BY 4.0 International**.
Schemas and Code samples: **MIT**.

---
**Maintained by the OAP Foundation**
*Reclaiming the public square for sovereign agents.*