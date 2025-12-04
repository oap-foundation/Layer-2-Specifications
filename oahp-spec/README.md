# Open Agent Health Protocol (OAHP)

[![Spec Version](https://img.shields.io/badge/spec-v1.0--RC-blue)](./RFC%20OAHP%20v1.0-RC.md)
[![Layer](https://img.shields.io/badge/OAP-Layer%202-red)](https://oap.foundation)
[![Status](https://img.shields.io/badge/status-CODE%20FREEZE-snowflake)](./RFC%20OAHP%20v1.0-RC.md)
[![License](https://img.shields.io/badge/license-MIT%2Fwm-green)](LICENSE)

> **⚠️ STATUS ALERT: CODE FREEZE**
>
> This specification is currently a **Release Candidate (v1.0-RC)**.
> We are in **Code Freeze**. No new features will be added. Feedback is limited to FHIR R4 mapping, Shamir's Secret Sharing vectors, and ZK Circuit definitions.

## 🏥 Introduction

The **Open Agent Health Protocol (OAHP)** is the **Health Layer (Layer 2)** of the Open Agent Protocol framework. It provides the standards for sovereign, secure, and patient-controlled exchange of medical data.

OAHP flips the traditional E-Health model upside down: Instead of data living in hospital silos, the **Patient Agent** holds the master record. It grants granular access to doctors, hospitals, or research institutes strictly on a need-to-know basis.

### Core Capabilities
*   **Patient-Held Records:** Data is aggregated locally by the patient's agent, encrypted with keys only the patient controls.
*   **Emergency "Break-Glass":** A cryptographic protocol (Shamir's Secret Sharing) allows emergency responders to access vital data if the patient is unconscious, without relying on a central backdoor.
*   **Zero-Knowledge Privacy:** Prove health facts (e.g., "Vaccinated against Measles") without revealing the underlying medical record.
*   **Semantic Interoperability:** Native wrapping of **HL7 FHIR** resources.

## 🏗 Architecture

OAHP acts as a secure envelope for medical standards.

```text
[      OAHP (Health)       ]  <-- "Access Grant" / "Break-Glass Request"
             |
[       OADP (Data)   L2   ]  <-- Encrypted FHIR Blobs & ODRL Policies
             |
[   OATP (Transport)  L1   ]  <-- Secure Delivery
             |
[      OAEP (Trust)   L0   ]  <-- Medical License Verification
```

## 📂 The Specification

The full normative specification is available here:

👉 **[READ THE SPECIFICATION (v1.0-RC)](RFC%20OAHP%20v1.0-RC.md)**

### Supported Flows
*   **[Standard Access](RFC%20OAHP%20v1.0-RC.md#41-standard-access-der-arztbesuch):** Doctor requests access -> Patient approves -> Keys delivered.
*   **[Break-Glass Protocol](RFC%20OAHP%20v1.0-RC.md#5-emergency-break-glass-protokoll-normativ):** Emergency access via distributed trust anchors (Shamir's Secret Sharing).
*   **[Zero-Knowledge Proofs](RFC%20OAHP%20v1.0-RC.md#42-zero-knowledge-proof-der-nachweis):** Generating zk-SNARKs for predicates (e.g., Age > 18, BMI < 30).

## ⚡ Technical Standards

Implementers must strictly adhere to these primitives:

| Component | Standard / Requirement |
| :--- | :--- |
| **Data Standard** | **HL7 FHIR R4/R5** (JSON) |
| **Encryption** | **JWE** (A256GCM) |
| **Emergency Auth** | **Shamir's Secret Sharing (SSS)** |
| **Privacy Proofs** | **zk-SNARKs** (Groth16 / Plonk) |
| **Consent** | **W3C ODRL** |

### Example: Break-Glass Request
```json
{
  "type": "BreakGlassRequest",
  "patientDid": "did:key:patient...",
  "reason": "UNCONSCIOUS_PATIENT",
  "location": "48.2082,16.3738",
  "responderProof": {
    "type": "OaepSignature",
    "signer": "did:web:emergency-services.at",
    "signature": "..." 
  }
}
```

## 🛠 Reference Implementations

*   **Core Logic (Rust):** [`oap-foundation/oahp-core-rs`](https://github.com/oap-foundation/oahp-core-rs)
    *   *Includes FHIR Parser, SSS Implementation, and ZK Prover.*
*   **Patient Wallet:** `oahp-patient-app` (Reference App)
*   **Doctor Dashboard:** `oahp-clinic-connector` (FHIR Bridge)

## 🧪 Conformance & Validation

All messages must be validated against the JSON Schemas defined in **[Section 7](RFC%20OAHP%20v1.0-RC.md#7-json-schema-definitionen-normativ)** of the RFC.

**Important:** The **Break-Glass** mechanism is critical infrastructure. Implementations must ensure that Trustee Servers for Shamir's Secret Sharing are geographically distributed and independently operated to prevent collusion.

## 🤝 Contributing

We are currently in **Code Freeze**.
*   **Accepted:** Security audits of the SSS implementation, FHIR mapping errors, or legal compliance (GDPR/HIPAA) gaps.
*   **Not Accepted:** New medical ontologies (outside LOINC/SNOMED) or alternative ZK schemes are deferred to v1.1.

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
*Healing the data fragmentation.*
