# RFC: Open Agent Data Protocol (OADP)
**Version:** 1.0 (PROPOSED STANDARD)
**Status:** CODE FREEZE
**Date:** 2025-12-03
**Layer:** 2 (Application)
**Dependencies:** OATP v1.0, OAEP v1.0, OAPP v1.0

## 1. Einleitung

Das **Open Agent Data Protocol (OADP)** ist der Standard für den souveränen Austausch von strukturierten Daten zwischen Agenten. Es ermöglicht eine "Data Economy", in der Nutzer die Hoheit über ihre Daten behalten, diese aber kontrolliert teilen, verkaufen oder für Forschung spenden können.

Im Gegensatz zu OAFP (das auf Feeds und Inhalte fokussiert ist) spezialisiert sich OADP auf **Datasets**, **Personal Data Vaults** (Gesundheit, Finanzen) und **IoT-Telemetrie**, bei denen granulare Zugriffskontrolle und Verschlüsselung oberste Priorität haben.

### 1.1 Zielsetzung und Reifegrad
Diese Version 1.0 definiert einen produktionsreifen Standard, der folgende Kernanforderungen erfüllt:
1.  **Self-Describing Data:** Jedes Datum wird von einem Manifest begleitet, das Schema, Herkunft und Zugriffsbedingungen beschreibt.
2.  **Granular Consent:** Zugriff wird über **W3C ODRL** (Open Digital Rights Language) gesteuert.
3.  **Encrypted by Default:** Daten verlassen den Speicherort des Besitzers standardmäßig nur als verschlüsselte JWE-Container.

### 1.2 Integration im OAP-Stack
*   **Identität:** Datenbesitzer und Konsumenten nutzen OAEP-DIDs.
*   **Transport:** Metadaten (Manifeste) und verschlüsselte Blobs werden via OATP transportiert.
*   **Ökonomie:** Bezahlte Datenzugriffe werden via OAPP abgewickelt.

---

## 2. Datenmodell & Verschlüsselung

OADP nutzt JSON-LD. Daten werden in zwei Komponenten geteilt: Das öffentliche **Manifest** (Metadaten) und den (meist verschlüsselten) **Blob** (Inhalt).

### 2.1 `DataManifest`
Der "Katalogzettel", der beschreibt, welche Daten verfügbar sind.

```json
{
  "@context": ["https://schema.org", "https://w3id.org/oadp/v1"],
  "type": "DataManifest",
  "id": "urn:uuid:data-123",
  "author": "did:web:patient.example.com",
  "created": "2025-12-03T10:00:00Z",
  
  // Was ist das für ein Datum?
  "datasetSchema": "https://schema.org/MedicalRecord",
  "description": "Blutwerte Q3/2025",
  
  // Wo liegt der Blob? (OATP Out-of-Band oder In-Band)
  "location": {
    "uri": "ipfs://QmHash...",
    "digest": "SHA-256=..."
  },

  // Verschlüsselungs-Metadaten (für den Blob)
  "encryption": {
    "algorithm": "A256GCM", // AES-256-GCM
    "keyId": "key-1" 
  },

  // Zugriffs-Policy (ODRL)
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

  // Optional: Anti-Spam für öffentliche Marktplätze
  "proofOfBurn": {
    "txId": "oap:tx:burn_123",
    "amount": "1"
  }
}
```

### 2.2 Normative Verschlüsselung (JWE)
Um die Sicherheit der "Private Vaults" zu garantieren, MÜSSEN Daten-Blobs gemäß **RFC 7516 (JSON Web Encryption)** verschlüsselt werden.

1.  **Content Encryption (Symmetrisch):**
    *   Algorithmus: **A256GCM** (AES GCM mit 256-bit Key).
    *   Dieser Schlüssel (CEK) verschlüsselt den eigentlichen Daten-Blob.
2.  **Key Wrapping (Asymmetrisch):**
    *   Der CEK wird für den Empfänger verschlüsselt, sobald Zugriff gewährt wurde (siehe 4.3).
    *   Algorithmus: **ECDH-ES+A256KW** (Elliptic Curve Diffie-Hellman Ephemeral Static mit AES Key Wrap).

---

## 3. Access Control (ODRL Integration)

OADP nutzt die **Open Digital Rights Language (ODRL)** des W3C, um maschinenlesbare Verträge über Daten zu schließen.

### 3.1 `AccessOffer` (Policy)
Das Angebot des Datenbesitzers (im Manifest eingebettet). Definiert Bedingungen:
*   **Payment:** "Zahle 5 EUR via OAPP."
*   **Attribution:** "Nenne den Autor."
*   **Verification:** "Beweise, dass du ein zertifizierter Arzt bist (Verifiable Credential)."

### 3.2 `AccessRequest`
Die Anfrage eines Konsumenten.
```json
{
  "type": "AccessRequest",
  "targetManifestId": "urn:uuid:data-123",
  "assignee": "did:web:research-institute.org",
  "purpose": "MedicalResearch", // Verwendungszweck
  // Optional: Credentials (z.B. Forscher-Ausweis)
  "claims": ["vc_jwt_string..."] 
}
```

---

## 4. Protokoll-Ablauf (Discovery & Exchange)

### Phase 1: Discovery (`DataAnnouncement`)
Agenten machen Daten verfügbar (z.B. in einem Data Marketplace oder P2P).
*   **Message:** `OADP_ANNOUNCE` via OATP.
*   **Payload:** Das `DataManifest` (siehe 7.4).
*   **Privacy:** Sensible Manifeste werden nur verschlüsselt an spezifische Gruppen gesendet.

### Phase 2: Negotiation (`AccessRequest`)
Ein Interessent fordert Zugriff an.
*   **Prüfung:** Der Owner-Agent (Personal AI) prüft die Anfrage gegen die Policy:
    *   Ist der Requester berechtigt?
    *   Ist eine Zahlung nötig? (Falls ja -> Trigger OAPP Flow).

### Phase 3: Fulfillment (`KeyDelivery`)
Nach erfolgreicher Prüfung (und ggf. OAPP `PaymentReceipt`) sendet der Owner den Schlüssel.

```json
{
  "type": "KeyDelivery",
  "targetManifestId": "urn:uuid:data-123",
  "recipient": "did:web:research-institute.org",
  // JWE: CEK verschlüsselt mit Public Key des Empfängers
  "jweKey": "eyJhbGciOiJFQ0RILUVTK0EyNTZLVyIs..." 
}
```

### Phase 4: Access (Decryption)
Der Empfänger lädt den Blob (via IPFS/OATP) und entschlüsselt ihn mit dem erhaltenen CEK.

---

## 5. Fehlerbehandlung (Error Handling)

Normative Fehlercodes für Layer 2 (OADP):

| Code | Bedeutung | Client-Aktion |
| :--- | :--- | :--- |
| **`OADP_INVALID_MANIFEST`** | Schema-Verletzung. | Manifest verwerfen. |
| **`OADP_ACCESS_DENIED`** | Policy nicht erfüllt (z.B. fehlendes VC). | Credentials prüfen oder Anfrage abbrechen. |
| **`OADP_PAYMENT_REQUIRED`** | Daten kostenpflichtig. | OAPP-Zahlung initiieren. |
| **`OADP_ENCRYPTION_FAILED`** | JWE ungültig/nicht entschlüsselbar. | KeyDelivery neu anfordern. |
| **`OADP_NOT_FOUND`** | Blob nicht mehr verfügbar (404). | Link prüfen. |

---

## 6. Sicherheits-Implikationen

### 6.1 Data Lineage & Provenance
*   Jedes Manifest MUSS kryptografisch signiert sein (OAEP `proof`).
*   Der `digest` im Manifest garantiert die Integrität des Blobs.
*   **Empfehlung:** Für kritische Daten (z.B. Sensoren) sollte der Blob selbst C2PA-Signaturen enthalten.

### 6.2 Privacy of Queries
Suchanfragen (`AccessRequest`) verraten Interessen.
*   **Mitigation:** Nutzung von **OATP Blind Relays** und Anonymisierungsnetzwerken, um Anfragen vom Requester zu entkoppeln, bis der Vertrag zustande kommt.

---

## 7. JSON-Schema Definitionen (Normativ)

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

## 8. Anhang (Informativ)

### 8.1 Chunking (Große Dateien)
Für Dateien > 10MB empfiehlt OADP das Aufteilen in Chunks. Das `DataManifest` referenziert dann eine **Manifest-Liste** (ähnlich Docker Images oder HLS Playlists), die die Digests der einzelnen Chunks enthält.

### 8.2 Referenz-Implementierung
Repo: `oap-foundation/oadp-core-rs` (Rust)
