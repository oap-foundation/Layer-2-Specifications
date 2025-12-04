# Open Agent Voting Protocol (OAVP)

[![Spec Version](https://img.shields.io/badge/spec-v1.0--RC-blue)](./RFC%20OAVP%20v1.0-RC.md)
[![Layer](https://img.shields.io/badge/OAP-Layer%202-magenta)](https://oap.foundation)
[![Status](https://img.shields.io/badge/status-CODE%20FREEZE-snowflake)](./RFC%20OAVP%20v1.0-RC.md)
[![License](https://img.shields.io/badge/license-MIT%2Fwm-green)](LICENSE)

> **⚠️ STATUS ALERT: CODE FREEZE**
>
> This specification is currently a **Release Candidate (v1.0-RC)**.
> We are in **Code Freeze**. No new features will be added. Feedback is limited to BLS12-381 curve parameters, Mixnet timing attack vectors, and Coercion Resistance logic.

## 🗳️ Introduction

The **Open Agent Voting Protocol (OAVP)** is the **Governance Layer (Layer 2)** of the Open Agent Protocol framework. It provides the cryptographic foundation for secure, anonymous, and universally verifiable digital elections.

OAVP solves the fundamental tension of digital democracy: How to prove *eligibility* (Identity) while guaranteeing absolute *secrecy* (Anonymity). It replaces trust in a central "black box" server with trust in mathematics.

### Core Capabilities
*   **Perfect Unlinkability:** Using **Blind Signatures**, the authority that validates your right to vote *cannot* see how you voted. The ballot box *cannot* see who you are.
*   **Coercion Resistance:** Supports "Panic Credentials" that allow coerced voters to cast plausible but invalid votes to satisfy an attacker, protecting the integrity of the true count.
*   **Universal Verifiability:** The final tally is mathematically provable by any observer using the public ledger (DBB), without revealing individual votes.
*   **Mixnet Transport:** Votes are routed through an OATP Mixnet to strip network-layer metadata (IP addresses).

## 🏗 Architecture

OAVP implements a cryptographic Separation of Duties.

```text
[      OAVP (Voting)       ]  <-- "Blind Sign Request" / "Cast Ballot"
             |
[   OATP (Mixnet)     L1   ]  <-- Anonymous Routing (Onion Layering)
             |
[      OAEP (Trust)   L0   ]  <-- Eligibility Check (Voter ID)
```

### The Actors
1.  **Registration Authority (RA):** Knows WHO you are. Issues tokens.
2.  **Validation Authority (VA):** Signs your ballot BLINDLY. Does not know what you voted.
3.  **Digital Ballot Box (DBB):** Receives the signed ballot. Knows WHAT you voted, but not who you are.

## 📂 The Specification

The full normative specification is available here:

👉 **[READ THE SPECIFICATION (v1.0-RC)](RFC%20OAVP%20v1.0-RC.md)**

### Supported Flows
*   **[Registration Phase](RFC%20OAVP%20v1.0-RC.md#phase-1-registrierung-setup):** Authentication via OAEP credentials.
*   **[Blinding Phase](RFC%20OAVP%20v1.0-RC.md#phase-2-vorbereitung--blinding-lokal):** Local generation of the vote and blinding factors.
*   **[Casting Phase](RFC%20OAVP%20v1.0-RC.md#phase-4-casting-das-mixnet):** Anonymous delivery via the Mixnet.

## ⚡ Technical Standards

Implementers must strictly adhere to these primitives:

| Component | Standard / Requirement |
| :--- | :--- |
| **Signatures** | **BLS12-381** (Boneh-Lynn-Shacham) |
| **Blinding** | Chaumian Blind Signatures over BLS |
| **Transport** | **OATP Mixnet** (Stratified Topology) |
| **Manifest** | JSON-LD Election Definition |

### Example: Cast Ballot Payload
```json
{
  "type": "CastBallot",
  "electionId": "urn:uuid:election-2025",
  "selection": "Candidate_A",
  "signature": "HexEncodedUnblindedBLSSignature...",
  "nonce": "RandomNonceForVerification"
}
```

## 🛠 Reference Implementations

*   **Core Logic (Rust):** [`oap-foundation/oavp-core-rs`](https://github.com/oap-foundation/oavp-core-rs)
    *   *Includes the BLS12-381 wrapper (blst), Blinding Logic, and Tallying Engine.*
*   **Mixnet Client:** `oavp-mix-client` (Beta)
*   **Tally Server:** `oavp-tally-node` (Auditable counting server)

## 🧪 Conformance & Validation

All messages must be validated against the JSON Schemas defined in **[Section 7](RFC%20OAVP%20v1.0-RC.md#7-json-schema-definitionen-normativ)** of the RFC.

**Important:** The **Mixnet Usage** (Section 6.2) is normative. Implementations that cast votes directly to the DBB (exposing IPs) are non-compliant and considered insecure.

## 🤝 Contributing

We are currently in **Code Freeze**.
*   **Accepted:** Fixes for side-channel leaks in the BLS library, Mixnet routing inefficiencies, or schema validation bugs.
*   **Not Accepted:** New voting schemes (e.g., Ranked Choice or Quadratic Voting logic) are deferred to v1.1 (the protocol is agnostic to the tallying logic).

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
*Democracy needs math, not trust.*
