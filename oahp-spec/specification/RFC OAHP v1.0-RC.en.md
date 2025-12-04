# RFC: Open Agent Health Protocol (OAHP)
**Version:** 1.0 (PROPOSED STANDARD)
**Status:** CODE FREEZE
**Date:** 2025-12-03
**Layer:** 2 (Application)
**Dependencies:** OATP v1.0, OAEP v1.0, OADP v1.0

## 1. Introduction

The **Open Agent Health Protocol (OAHP)** defines the standard for the sovereign, secure, and patient-centric exchange of health data. It builds upon the generic data protocol OADP, extending it with specific medical semantics (HL7 FHIR), emergency mechanisms, and strict consent management rules.

### 1.1 Objectives and Maturity Level
This Version 1.0 defines a production-ready standard that meets the extreme security requirements of health data:
1.  **Patient-Held Records:** The patient (represented by their Agent) is the primary aggregator and sole holder of the keys to their health record.
2.  **Emergency Resilience:** A cryptographically secured "Break-Glass" procedure guarantees emergency access without central backdoors.
3.  **Privacy Preservation:** Utilization of Zero-Knowledge Proofs (ZKP) and Local-AI analysis to minimize data leakage.

### 1.2 Integration into the OAP Stack
*   **Identity:** Physicians and emergency responders must identify themselves via OAEP-DIDs with verified `MedicalLicenseCredentials`.
*   **Transport:** All medical data is transported as encrypted OADP blobs via OATP.
*   **Audit:** Accesses are immutably logged in the OADP Ledger.

---

## 2. Data Model (FHIR & Semantics)

OAHP wraps medical standards into OAP-compatible structures.

### 2.1 `HealthRecordManifest`
A specialization of the OADP Manifest. References an encrypted FHIR Bundle blob.

```json
{
  "@context": ["https://w3id.org/oahp/v1", "http://hl7.org/fhir/json-ld"],
  "type": "HealthRecordManifest",
  "id": "urn:uuid:record-123",
  "author": "did:web:hospital.example.com",
  "patient": "did:key:patient...",
  
  // Medical metadata (LOINC/SNOMED)
  "coding": {
    "system": "http://loinc.org",
    "code": "11502-2", 
    "display": "Laboratory report"
  },
  
  // Reference to OADP Blob (OATP/IPFS)
  "content": {
    "uri": "ipfs://QmHash...",
    "digest": "SHA-256=...",
    "encryption": "A256GCM" 
  }
}
```

### 2.2 `InsightSummary` (AI Analysis)
Condensed information instead of a flood of raw data. Enables physicians to make quick decisions based on local AI analyses performed by the Patient Agent.

```json
{
  "type": "InsightSummary",
  "topic": "Cardiology",
  "summary": "Resting heart rate increased by 15% over last 4 weeks.",
  "evidence": ["urn:uuid:raw-data-1"],
  "aiModel": "Th!nkHealth-v4 (MDR Class IIa)"
}
```

### 2.3 `ConsentRevocation`
The message used to invalidate a previously granted access key.

---

## 3. Architecture & Roles

### 3.1 Patient Agent
Manages the private key and the policy database. It acts as a "Gatekeeper" and performs local AI analyses.

### 3.2 Practitioner Agent
Represents medical personnel. Must authenticate using a `MedicalLicenseCredential` (issued by medical boards/authorities).

### 3.3 Emergency Responder
An emergency physician with a special `EmergencyDID`. This role authorizes the initiation of the Break-Glass protocol.

---

## 4. Protocol Flow (Consent & Access)

### 4.1 Standard Access (Doctor Visit)
1.  **Request:** Physician sends `AccessRequest` (OADP) with purpose "Treatment".
2.  **Check:** Patient Agent verifies the physician's credentials.
3.  **Grant:** Patient confirms. Agent sends `KeyDelivery` (OADP) containing the decryption key for the requested manifests.

### 4.2 Zero-Knowledge Proof (The Verification)
For employers or travel requirements (e.g., "Measles vaccine exists?"), without disclosing the full medical record.
1.  **Request:** Verifier sends `ProofRequest` (e.g., "Present predicate: MeaslesVaccine exists").
2.  **Generate:** Patient Agent locally calculates a zk-SNARK proof over its FHIR data.
3.  **Verify:** Agent sends only the result ("True") and the cryptographic proof. No health data leaves the device.

### 4.3 Consent Revocation
The patient can revoke access at any time.
1.  **Action:** Patient selects "Revoke access for Dr. X".
2.  **Message:** Agent sends `ConsentRevocation` to Dr. X.
3.  **Effect:**
    *   **Technical:** Since the key was already delivered, this is a "Soft Revoke". Dr. X is *legally* obligated to delete the data.
    *   **Forward Secrecy:** The Patient Agent internally rotates keys (Re-Encryption) for future data. Dr. X has no access to new data.

---

## 5. Emergency "Break-Glass" Protocol (Normative)

This procedure allows access to vital data ("Emergency Set") without the patient's active consent, but protects against abuse through extreme transparency and cryptographic hurdles.

### 5.1 Preparation (Key Escrow)
The patient defines an emergency dataset. The decryption key ($K_{em}$) for this set is split into $N$ parts (shares) using **Shamir's Secret Sharing (SSS)**.
*   **Distribution:** Shares are distributed to $N$ Trustees (e.g., OAP Foundation, GP, Notary, Partner).
*   **Threshold:** $M$ shares are required to reconstruct the key (e.g., 3 of 5).

### 5.2 Emergency Access
1.  **Identification:** Responder scans the public QR code (Health Card/Smartphone Lockscreen) -> retrieves DIDs of the Trustees.
2.  **BreakGlassRequest:** Responder sends a signed `BreakGlassRequest` to all Trustees.
    *   *Payload:* Location, Time, Reason, Signature of the `EmergencyDID`.
3.  **Validation & Audit:**
    *   Trustees verify the responder's signature against a government Root-CA.
    *   Trustees immutably log the access in the **OADP Audit Ledger**.
4.  **Release:** Upon success, Trustees send their share to the Responder Agent.
5.  **Reconstruction:** Once the Responder has $M$ shares, their Agent reconstructs $K_{em}$ and decrypts the emergency set.

### 5.3 Post-Mortem (Notification)
As soon as the Patient Agent is back online, it detects the access in the Audit Log. It displays a prominent, non-dismissible warning: **"EMERGENCY ACCESS OCCURRED BY [Name] ON [Date]"**.

---

## 6. Security Implications

### 6.1 Granularity
OAHP forbids "All or Nothing" sharing. Access is granted based on **FHIR Resource Types** (e.g., `Patient/*.read`) or **Tags** (e.g., `category=mental-health`).

### 6.2 ZK-SNARK Specification (Normative)
Implementations MUST use the **Groth16** or **Plonk** proof systems.
*   **Trusted Setup:** For Groth16, the *Common Reference Strings (CRS)* are provided by the OAP Foundation as trusted Common Goods.
*   **Circuits:** Standard circuits (Vaccination status, Age, BMI Range) are part of the reference implementation.

### 6.3 Legal Compliance (GDPR & HIPAA)
OAHP implements regulatory requirements technically:
*   **Art. 9 GDPR:** Explicit consent and encryption for special categories of data.
*   **Right to be Forgotten:** Implemented via `ConsentRevocation` and cryptographic key rotation (Crypto-Shredding).
*   **Data Portability:** FHIR export is natively supported.

---

## 7. JSON-Schema Definitions (Normative)

### 7.1 `HealthRecordManifest` Schema
*(Extends OADP DataManifest with `coding`)*

### 7.2 `InsightSummary` Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["type", "topic", "summary", "aiModel"],
  "properties": {
    "type": { "const": "InsightSummary" },
    "topic": { "type": "string" },
    "summary": { "type": "string" },
    "evidence": { "type": "array", "items": { "type": "string", "format": "uri" } },
    "aiModel": { "type": "string" }
  }
}
```

### 7.3 `BreakGlassRequest` Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["type", "patientDid", "reason", "responderProof"],
  "properties": {
    "type": { "const": "BreakGlassRequest" },
    "patientDid": { "type": "string", "format": "uri" },
    "reason": { "type": "string" }, // "UNCONSCIOUS_PATIENT"
    "location": { "type": "string" }, // GPS Lat/Lon
    "responderProof": { "$ref": "#/definitions/OaepSignature" }
  }
}
```

### 7.4 `ConsentRevocation` Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["type", "targetManifestId", "revocationDate"],
  "properties": {
    "type": { "const": "ConsentRevocation" },
    "targetManifestId": { "type": "string", "format": "uri" },
    "revocationDate": { "type": "string", "format": "date-time" }
  }
}
```

---

## 8. Error Handling

Normative Error Codes:

| Code | Meaning | Client Action |
| :--- | :--- | :--- |
| **`OAHP_CONSENT_DENIED`** | Patient denied access. | Abort. |
| **`OAHP_EMERGENCY_ACCESS_DENIED`** | Responder credentials invalid. | Use manual procedure. |
| **`OAHP_INVALID_FHIR`** | Data does not conform to FHIR standard. | Report parsing error. |
| **`OAHP_REVOKED_KEY`** | Key has been revoked. | Delete data. |

---

## 9. Appendix (Informative)

### 9.1 Reference Implementation
Repo: `oahp-core-rs` (Rust) including FHIR parser and Shamir's Secret Sharing module.

### 9.2 FHIR Version
OAHP v1.0 recommends **FHIR R4** or **R5** as the payload format.

### 9.3 Legal Compliance Mapping

| Requirement | OAHP Feature |
| :--- | :--- |
| **GDPR Art. 6 (Consent)** | OAHP `AccessRequest` + Signature |
| **GDPR Art. 9 (Health Data)** | E2E-Encryption + JWE |
| **GDPR Art. 17 (Deletion)** | `ConsentRevocation` + Crypto-Shredding |
| **HIPAA Security Rule** | Audit Trail in Ledger, Encryption at Rest/Transit |
| **Emergency Access** | Break-Glass Protocol (SSS) |