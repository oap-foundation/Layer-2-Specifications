# RFC: Open Agent Health Protocol (OAHP)
**Version:** 1.0 (PROPOSED STANDARD)
**Status:** CODE FREEZE
**Date:** 2025-12-03
**Layer:** 2 (Application)
**Dependencies:** OATP v1.0, OAEP v1.0, OADP v1.0

## 1. Einleitung

Das **Open Agent Health Protocol (OAHP)** definiert den Standard für den souveränen, sicheren und patientenzentrierten Austausch von Gesundheitsdaten. Es baut auf dem generischen Datenprotokoll OADP auf, erweitert dieses aber um spezifische medizinische Semantik (HL7 FHIR), Notfallmechanismen und strenge Consent-Management-Regeln.

### 1.1 Zielsetzung und Reifegrad
Diese Version 1.0 definiert einen produktionsreifen Standard, der die extremen Sicherheitsanforderungen von Gesundheitsdaten erfüllt:
1.  **Patient-Held Records:** Der Patient (vertreten durch seinen Agenten) ist der primäre Aggregator und alleinige Schlüsselinhaber seiner Gesundheitsakte.
2.  **Emergency Resilience:** Ein kryptografisch gesichertes "Break-Glass"-Verfahren garantiert Notfallzugriff ohne zentrale Hintertüren.
3.  **Privacy Preservation:** Einsatz von Zero-Knowledge Proofs (ZKP) und Local-AI-Analyse zur Minimierung des Datenabflusses.

### 1.2 Integration im OAP-Stack
*   **Identität:** Ärzte und Notärzte müssen sich über OAEP-DIDs mit verifizierten `MedicalLicenseCredentials` ausweisen.
*   **Transport:** Alle medizinischen Daten werden als verschlüsselte OADP-Blobs über OATP transportiert.
*   **Audit:** Zugriffe werden unveränderlich im OADP-Ledger protokolliert.

---

## 2. Datenmodell (FHIR & Semantik)

OAHP verpackt medizinische Standards in OAP-kompatible Strukturen.

### 2.1 `HealthRecordManifest`
Spezialisierung des OADP-Manifests. Referenziert einen verschlüsselten FHIR-Bundle-Blob.

```json
{
  "@context": ["https://w3id.org/oahp/v1", "http://hl7.org/fhir/json-ld"],
  "type": "HealthRecordManifest",
  "id": "urn:uuid:record-123",
  "author": "did:web:hospital.example.com",
  "patient": "did:key:patient...",
  
  // Medizinische Metadaten (LOINC/SNOMED)
  "coding": {
    "system": "http://loinc.org",
    "code": "11502-2", 
    "display": "Laboratory report"
  },
  
  // Verweis auf OADP Blob (OATP/IPFS)
  "content": {
    "uri": "ipfs://QmHash...",
    "digest": "SHA-256=...",
    "encryption": "A256GCM" 
  }
}
```

### 2.2 `InsightSummary` (KI-Analyse)
Verdichtete Informationen statt Rohdatenflut. Ermöglicht Ärzten schnelle Entscheidungen basierend auf lokalen KI-Analysen des Patienten-Agenten.

```json
{
  "type": "InsightSummary",
  "topic": "Cardiology",
  "summary": "Resting heart rate increased by 15% over last 4 weeks.",
  "evidence": ["urn:uuid:raw-data-1"],
  "aiModel": "Th!nkHealth-v4 (MDR Class IIa)"
}
```

### 2.3 `ConsentRevocation` (Widerruf)
Die Nachricht, um einen erteilten Zugriffsschlüssel ungültig zu machen.

---

## 3. Architektur & Rollen

### 3.1 Patient Agent
Verwaltet den privaten Schlüssel und die Policy-Datenbank. Er agiert als "Gatekeeper" und führt lokale KI-Analysen durch.

### 3.2 Practitioner Agent
Repräsentiert medizinisches Personal. Muss sich durch ein `MedicalLicenseCredential` (ausgestellt von Ärztekammern/Behörden) ausweisen.

### 3.3 Emergency Responder
Ein Notarzt mit einer speziellen `EmergencyDID`. Diese Rolle berechtigt zum Starten des Break-Glass-Protokolls.

---

## 4. Protokoll-Ablauf (Consent & Access)

### 4.1 Standard Access (Der Arztbesuch)
1.  **Request:** Arzt sendet `AccessRequest` (OADP) mit Zweck "Behandlung".
2.  **Check:** Patient-Agent prüft Credentials des Arztes.
3.  **Grant:** Patient bestätigt. Agent sendet `KeyDelivery` (OADP) mit dem Entschlüsselungs-Key für die angeforderten Manifeste.

### 4.2 Zero-Knowledge Proof (Der Nachweis)
Für Arbeitgeber oder Reiseanforderungen ("Masern-Impfung vorhanden?"), ohne die medizinische Akte offenzulegen.
1.  **Request:** Verifier sendet `ProofRequest` (z.B. "Present predicate: MeaslesVaccine exists").
2.  **Generate:** Patient-Agent berechnet lokal einen zk-SNARK Beweis über seine FHIR-Daten.
3.  **Verify:** Agent sendet nur das Ergebnis ("True") und den kryptografischen Beweis. Keine Gesundheitsdaten verlassen das Gerät.

### 4.3 Consent Revocation (Der Widerruf)
Der Patient kann Zugriff jederzeit entziehen.
1.  **Action:** Patient wählt "Zugriff für Dr. X widerrufen".
2.  **Message:** Agent sendet `ConsentRevocation` an Dr. X.
3.  **Effect:**
    *   **Technisch:** Da der Key bereits geliefert wurde, ist dies ein "Soft Revoke". Dr. X ist *rechtlich* verpflichtet, die Daten zu löschen.
    *   **Forward Secrecy:** Der Patient-Agent rotiert intern die Schlüssel (Re-Encryption) für zukünftige Daten. Dr. X hat keinen Zugriff auf neue Daten.

---

## 5. Emergency "Break-Glass" Protokoll (Normativ)

Dieses Verfahren ermöglicht den Zugriff auf lebenswichtige Daten ("Emergency Set") ohne aktive Zustimmung des Patienten, schützt aber vor Missbrauch durch extreme Transparenz und kryptografische Hürden.

### 5.1 Vorbereitung (Key Escrow)
Der Patient definiert ein Notfall-Datenset. Der Entschlüsselungs-Key ($K_{em}$) für dieses Set wird mittels **Shamir's Secret Sharing (SSS)** in $N$ Teile (Shares) zerlegt.
*   **Verteilung:** Die Shares werden an $N$ Treuhänder (Trustees) verteilt (z.B. OAP Foundation, Hausarzt, Notariat, Partner).
*   **Threshold:** Es werden $M$ Shares benötigt, um den Key zu rekonstruieren (z.B. 3 von 5).

### 5.2 Der Notfall-Zugriff
1.  **Identifikation:** Notarzt scannt den öffentlichen QR-Code (Gesundheitskarte/Smartphone-Lockscreen) -> erhält DIDs der Trustees.
2.  **BreakGlassRequest:** Notarzt sendet einen signierten `BreakGlassRequest` an alle Trustees.
    *   *Payload:* Standort, Zeit, Begründung, Signatur der `EmergencyDID`.
3.  **Validierung & Audit:**
    *   Trustees prüfen die Signatur des Notarztes gegen eine staatliche Root-CA.
    *   Trustees protokollieren den Zugriff unveränderlich im **OADP Audit Ledger**.
4.  **Release:** Bei Erfolg senden die Trustees ihren Share an den Notarzt-Agenten.
5.  **Rekonstruktion:** Sobald der Notarzt $M$ Shares hat, rekonstruiert sein Agent $K_{em}$ und entschlüsselt das Notfall-Set.

### 5.3 Post-Mortem (Benachrichtigung)
Sobald der Patient-Agent wieder online ist, erkennt er den Zugriff im Audit-Log. Er zeigt eine prominente, nicht löschbare Warnung an: **"NOTFALL-ZUGRIFF ERFOLGT DURCH [Name] AM [Datum]"**.

---

## 6. Sicherheits-Implikationen

### 6.1 Granularität
OAHP verbietet "Alles oder Nichts"-Freigaben. Zugriff wird auf **FHIR-Ressourcentypen** (z.B. `Patient/*.read`) oder **Tags** (z.B. `category=mental-health`) gewährt.

### 6.2 ZK-SNARK Spezifikation (Normativ)
Implementierungen MÜSSEN das **Groth16** oder **Plonk** Beweissystem nutzen.
*   **Trusted Setup:** Für Groth16 werden die *Common Reference Strings (CRS)* von der OAP Foundation als vertrauenswürdige Common Goods bereitgestellt.
*   **Circuits:** Standard-Circuits (Impfstatus, Alter, BMI-Range) sind Teil der Referenzimplementierung.

### 6.3 Legal Compliance (DSGVO/GDPR & HIPAA)
OAHP setzt regulatorische Anforderungen technisch um:
*   **Art. 9 DSGVO:** Explizite Einwilligung und Verschlüsselung für besondere Datenkategorien.
*   **Right to be Forgotten:** Umgesetzt durch `ConsentRevocation` und kryptografische Schlüssel-Rotation (Crypto-Shredding).
*   **Data Portability:** FHIR-Export ist nativ unterstützt.

---

## 7. JSON-Schema Definitionen (Normativ)

### 7.1 `HealthRecordManifest` Schema
*(Erweitert OADP DataManifest um `coding`)*

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

## 8. Fehlerbehandlung (Error Handling)

Normative Fehlercodes:

| Code | Bedeutung | Client-Aktion |
| :--- | :--- | :--- |
| **`OAHP_CONSENT_DENIED`** | Patient lehnt Zugriff ab. | Abbruch. |
| **`OAHP_EMERGENCY_ACCESS_DENIED`** | Notarzt-Credentials ungültig. | Manuelle Prozedur. |
| **`OAHP_INVALID_FHIR`** | Daten entsprechen nicht FHIR-Standard. | Parsing-Fehler melden. |
| **`OAHP_REVOKED_KEY`** | Schlüssel wurde widerrufen. | Daten löschen. |

---

## 9. Anhang (Informativ)

### 9.1 Referenz-Implementierung
Repo: `oahp-core-rs` (Rust) mit FHIR-Parser und Shamir-Secret-Sharing Modul.

### 9.2 FHIR Version
OAHP v1.0 empfiehlt **FHIR R4** oder **R5** als Payload-Format.

### 9.3 Legal Compliance Mapping

| Anforderung | OAHP Feature |
| :--- | :--- |
| **GDPR Art. 6 (Consent)** | OAHP `AccessRequest` + Signatur |
| **GDPR Art. 9 (Health Data)** | E2E-Encryption + JWE |
| **GDPR Art. 17 (Deletion)** | `ConsentRevocation` + Crypto-Shredding |
| **HIPAA Security Rule** | Audit Trail im Ledger, Encryption at Rest/Transit |
| **Emergency Access** | Break-Glass Protocol (SSS) |
