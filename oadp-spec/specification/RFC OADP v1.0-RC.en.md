# RFC: Open Agent Data Protocol (OADP)
**Version:** 1.0 (PROPOSED STANDARD)
**Status:** CODE FREEZE
**Date:** 2025-12-03
**Layer:** 2 (Application)
**Dependencies:** OATP v1.0, OAEP v1.0, OAPP v1.0

## 1. Introduction

The **Open Agent Data Protocol (OADP)** is the standard for the sovereign exchange of structured data between agents. It enables a "Data Economy" where users retain sovereignty over their data while being able to share, sell, or donate it for research in a controlled manner.

In contrast to OAFP (which focuses on feeds and content), OADP specializes in **Datasets**, **Personal Data Vaults** (health, finance), and **IoT Telemetry**, where granular access control and encryption are of the highest priority.

### 1.1 Objectives and Maturity Level
This Version 1.0 defines a production-ready standard that fulfills the following core requirements:
1.  **Self-Describing Data:** Every piece of data is accompanied by a manifest describing the schema, origin, and access conditions.
2.  **Granular Consent:** Access is controlled via **W3C ODRL** (Open Digital Rights Language).
3.  **Encrypted by Default:** By default, data leaves the owner's storage location only as encrypted JWE containers.

### 1.2 Integration in the OAP Stack
*   **Identity:** Data owners and consumers use OAEP-DIDs.
*   **Transport:** Metadata (Manifests) and encrypted Blobs are transported via OATP.
*   **Economy:** Paid data access is processed via OAPP.

---

## 2. Data Model & Encryption

OADP utilizes JSON-LD. Data is divided into two components: The public **Manifest** (metadata) and the (usually encrypted) **Blob** (content).

### 2.1 `DataManifest`
The "catalog card" that describes what data is available.

```json
{
  "@context": ["https://schema.org", "https://w3id.org/oadp/v1"],
  "type": "DataManifest",
  "id": "urn:uuid:data-123",
  "author": "did:web:patient.example.com",
  "created": "2025-12-03T10:00:00Z",
  
  // What kind of data is this?
  "datasetSchema": "https://schema.org/MedicalRecord",
  "description": "Blood values Q3/2025",
  
  // Where is the Blob located? (OATP Out-of-Band or In-Band)
  "location": {
    "uri": "ipfs://QmHash...",
    "digest": "SHA-256=..."
  },

  // Encryption Metadata (for the Blob)
  "encryption": {
    "algorithm": "A256GCM", // AES-256-GCM
    "keyId": "key-1" 
  },

  // Access Policy (ODRL)
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
  },

  // Optional: Anti-Spam for public marketplaces
  "proofOfBurn": {
    "txId": "oap:tx:burn_123",
    "amount": "1"
  }
}
```

### 2.2 Normative Encryption (JWE)
To guarantee the security of "Private Vaults," data blobs MUST be encrypted according to **RFC 7516 (JSON Web Encryption)**.

1.  **Content Encryption (Symmetric):**
    *   Algorithm: **A256GCM** (AES GCM with 256-bit Key).
    *   This key (CEK) encrypts the actual data blob.
2.  **Key Wrapping (Asymmetric):**
    *   The CEK is encrypted for the recipient once access has been granted (see 4.3).
    *   Algorithm: **ECDH-ES+A256KW** (Elliptic Curve Diffie-Hellman Ephemeral Static with AES Key Wrap).

---

## 3. Access Control (ODRL Integration)

OADP uses the W3C **Open Digital Rights Language (ODRL)** to establish machine-readable contracts regarding data.

### 3.1 `AccessOffer` (Policy)
The data owner's offer (embedded in the Manifest). Defines conditions:
*   **Payment:** "Pay 5 EUR via OAPP."
*   **Attribution:** "Cite the author."
*   **Verification:** "Prove that you are a certified doctor (Verifiable Credential)."

### 3.2 `AccessRequest`
The consumer's request.
```json
{
  "type": "AccessRequest",
  "targetManifestId": "urn:uuid:data-123",
  "assignee": "did:web:research-institute.org",
  "purpose": "MedicalResearch", // Purpose of use
  // Optional: Credentials (e.g., Researcher ID)
  "claims": ["vc_jwt_string..."] 
}
```

---

## 4. Protocol Flow (Discovery & Exchange)

### Phase 1: Discovery (`DataAnnouncement`)
Agents make data available (e.g., in a Data Marketplace or P2P).
*   **Message:** `OADP_ANNOUNCE` via OATP.
*   **Payload:** The `DataManifest` (see 7.4).
*   **Privacy:** Sensitive manifests are only sent encrypted to specific groups.

### Phase 2: Negotiation (`AccessRequest`)
An interested party requests access.
*   **Validation:** The owner agent (Personal AI) checks the request against the policy:
    *   Is the requester authorized?
    *   Is payment required? (If yes -> Trigger OAPP Flow).

### Phase 3: Fulfillment (`KeyDelivery`)
After successful validation (and optionally OAPP `PaymentReceipt`), the owner sends the key.

```json
{
  "type": "KeyDelivery",
  "targetManifestId": "urn:uuid:data-123",
  "recipient": "did:web:research-institute.org",
  // JWE: CEK encrypted with the recipient's Public Key
  "jweKey": "eyJhbGciOiJFQ0RILUVTK0EyNTZLVyIs..." 
}
```

### Phase 4: Access (Decryption)
The recipient downloads the blob (via IPFS/OATP) and decrypts it using the received CEK.

---

## 5. Error Handling

Normative error codes for Layer 2 (OADP):

| Code | Meaning | Client Action |
| :--- | :--- | :--- |
| **`OADP_INVALID_MANIFEST`** | Schema violation. | Discard manifest. |
| **`OADP_ACCESS_DENIED`** | Policy not met (e.g., missing VC). | Check credentials or abort request. |
| **`OADP_PAYMENT_REQUIRED`** | Data requires payment. | Initiate OAPP payment. |
| **`OADP_ENCRYPTION_FAILED`** | JWE invalid/undecryptable. | Request KeyDelivery again. |
| **`OADP_NOT_FOUND`** | Blob no longer available (404). | Check link. |

---

## 6. Security Implications

### 6.1 Data Lineage & Provenance
*   Every manifest MUST be cryptographically signed (OAEP `proof`).
*   The `digest` in the manifest guarantees the integrity of the blob.
*   **Recommendation:** For critical data (e.g., sensors), the blob itself should contain C2PA signatures.

### 6.2 Privacy of Queries
Search queries (`AccessRequest`) reveal interests.
*   **Mitigation:** Use **OATP Blind Relays** and anonymization networks to decouple requests from the requester until the contract is formed.

---

## 7. JSON-Schema Definitions (Normative)

### 7.1 `DataManifest`

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["type", "id", "author", "datasetSchema", "location", "encryption"],
  "properties": {
    "type": { "const": "DataManifest" },
    "id": { "type": "string", "format": "uri" },
    "author": { "type": "string", "format": "uri" },
    "datasetSchema": { "type": "string", "format": "uri" },
    "location": {
      "type": "object",
      "required": ["uri", "digest"],
      "properties": {
        "uri": { "type": "string" },
        "digest": { "type": "string", "pattern": "^SHA-256=.*$" }
      }
    },
    "encryption": {
      "type": "object",
      "required": ["algorithm"],
      "properties": {
        "algorithm": { "const": "A256GCM" },
        "keyId": { "type": "string" }
      }
    },
    "accessPolicy": { "$ref": "#/definitions/AccessOffer" },
    "proofOfBurn": {
      "type": "object",
      "properties": {
        "txId": { "type": "string" },
        "amount": { "type": "string" }
      }
    }
  }
}
```

### 7.2 `AccessOffer` (ODRL Subset)

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "type": { "const": "odrl:Offer" },
    "permission": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["action"],
        "properties": {
          "action": { "type": "string" }, // e.g. "odrl:read"
          "duty": { "type": "array" },
          "constraint": { "type": "array" }
        }
      }
    }
  }
}
```

### 7.3 `AccessRequest`

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["type", "targetManifestId", "assignee", "purpose"],
  "properties": {
    "type": { "const": "AccessRequest" },
    "targetManifestId": { "type": "string", "format": "uri" },
    "assignee": { "type": "string", "format": "uri" },
    "purpose": { "type": "string" },
    "claims": {
      "type": "array",
      "items": { "type": "string" } // VC JWTs
    }
  }
}
```

### 7.4 `DataAnnouncement`

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["type", "manifest"],
  "properties": {
    "type": { "const": "DataAnnouncement" },
    "manifest": { "$ref": "#/definitions/DataManifest" },
    "tags": { "type": "array", "items": { "type": "string" } }
  }
}
```

### 7.5 `KeyDelivery`

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["type", "targetManifestId", "recipient", "jweKey"],
  "properties": {
    "type": { "const": "KeyDelivery" },
    "targetManifestId": { "type": "string", "format": "uri" },
    "recipient": { "type": "string", "format": "uri" },
    "jweKey": { "type": "string", "description": "JWE Compact String (Encrypted CEK)" }
  }
}
```

---

## 8. Appendix (Informative)

### 8.1 Chunking (Large Files)
For files > 10MB, OADP recommends splitting them into chunks. The `DataManifest` then references a **Manifest List** (similar to Docker Images or HLS Playlists) containing the digests of the individual chunks.

### 8.2 Reference Implementation
Repo: `oap-foundation/oadp-core-rs` (Rust)