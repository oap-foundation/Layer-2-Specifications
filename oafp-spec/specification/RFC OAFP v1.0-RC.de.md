# RFC: Open Agent Feed Protocol (OAFP)
**Version:** 1.0 (PROPOSED STANDARD)
**Status:** CODE FREEZE
**Date:** 2025-12-03
**Layer:** 2 (Application)
**Dependencies:** OATP v1.0, OAEP v1.0, OAPP v1.0

## 1. Einleitung

Das **Open Agent Feed Protocol (OAFP)** ist der Standard für die dezentrale Verbreitung und Kuration von Inhalten (Social Media, News, Microblogging) im OAP-Ökosystem. Es ersetzt die zentralisierten "Feeds" traditioneller Plattformen durch eine Architektur, die auf **Algorithmischer Souveränität** und **Dateneigentum** basiert.

### 1.1 Kernprinzipien
1.  **Entkopplung:** Der Speicherort (Wo liegen die Daten?) ist vom Index (Wer weist darauf hin?) und von der Kuration (Wer wählt aus?) getrennt.
2.  **Lokale Kuration ("Intention Economy"):** Es gibt keinen globalen Algorithmus. Die Zusammenstellung des Feeds erfolgt exklusiv durch die Personal AI auf dem Endgerät des Nutzers basierend auf dessen expliziten Interessen und Vertrauensnetzwerken.
3.  **Provenienz:** Inhalte sind kryptografisch signiert (OAEP) und optional durch C2PA (Content Credentials) gegen Manipulation gesichert.

### 1.2 Architektur-Überblick
OAFP unterscheidet zwei Räume:
*   **Private Moments:** E2E-verschlüsselte P2P-Kommunikation via OATP (Direct Messaging, Small Groups).
*   **Public Stage:** Veröffentlichung von **Manifesten** in öffentlichen Netzwerken (DHT, PubSub), während die **Blobs** (Medien) in Content Addressable Storage (CAS) wie IPFS liegen.

---

## 2. Datenmodell (Semantik)

Das zentrale Element ist das **Manifest**. Es beschreibt Inhalt, Kontext und Autorschaft. OAFP nutzt JSON-LD und Schema.org.

### 2.1 `ContentManifest` (Der Beitrag)
Der "Katalogzettel", der öffentlich verbreitet wird.

```json
{
  "@context": ["https://schema.org", "https://w3id.org/oafp/v1"],
  "type": "ContentManifest",
  "id": "urn:uuid:manifest-123...",
  "author": "did:web:journalist.example.com",
  "created": "2025-12-03T10:00:00Z",
  
  // Metadaten für die KI-Kuration
  "meta": {
    "type": "schema:NewsArticle", // oder SocialMediaPosting
    "topics": ["Tech", "Privacy"],
    "language": "de",
    "contentWarning": "POLITICAL_DISCOURSE" 
  },

  // Verweis auf den Inhalt (CAS)
  "content": {
    "uri": "ipfs://QmHash...",
    "digest": "SHA256:...",
    "mimeType": "text/markdown",
    "encrypted": false // true bei Pay-to-View
  },

  // Ökonomischer Spam-Schutz (siehe Kap. 4)
  "proofOfBurn": {
    "txId": "oap:tx:burn_transaction_123",
    "amount": "10" // Credits
  },

  // Signatur des Autors
  "proof": {
    "type": "OaepSignature2025",
    "signatureValue": "..."
  }
}
```

### 2.2 `Interaction` (Die Reaktion)
Likes, Kommentare und Shares sind eigenständige Objekte, die auf ein Manifest verweisen. Sie liegen nicht beim Autor, sondern beim Interagierenden.

```json
{
  "type": "Interaction",
  "interactionType": "schema:LikeAction", // oder CommentAction, ShareAction
  "target": "urn:uuid:manifest-123...",
  "author": "did:key:fan...",
  "proof": { ... }
}
```

### 2.3 `Tombstone` (Das Lösch-Signal)
Um das "Recht auf Vergessenwerden" in unveränderlichen Netzwerken sozial durchzusetzen.

```json
{
  "type": "Tombstone",
  "targetManifestId": "urn:uuid:manifest-123...",
  "reason": "AUTHOR_REQUEST", // oder LEGAL_TAKEDOWN, SPAM
  "deletedAt": "2025-12-04T10:00:00Z",
  "author": "did:web:journalist.example.com",
  "proof": { ... }
}
```

### 2.4 `KeyDelivery` (Pay-to-View Unlock)
Das Objekt, das den Entschlüsselungs-Key für Premium-Inhalte überträgt.

```json
{
  "type": "KeyDelivery",
  "targetManifestId": "urn:uuid:manifest-123...",
  "recipient": "did:key:buyer...",
  // JWE verschlüsselter Content Key (nur für Recipient lesbar)
  "keyPayload": "eyJhbGciOiJFQ0RILUVTI..." 
}
```

---

## 3. Protokoll-Ablauf (OATP Transport)

OAFP definiert spezifische Nachrichtentypen, die über OATP transportiert werden.

### 3.1 Subscription (`OAFP_SUBSCRIBE`)
Die Personal AI signalisiert Interesse an Inhalten.
*   **Aktion:** `SUBSCRIBE` oder `UNSUBSCRIBE`.
*   **Empfänger:** Ein PubSub-Relay oder direkt die DID eines Creators.
*   **Payload:**
    ```json
    {
      "type": "Subscription",
      "action": "SUBSCRIBE",
      "topics": ["#Privacy"],
      "authors": ["did:web:news.com"]
    }
    ```

### 3.2 Announcement (`OAFP_ANNOUNCE`)
Ein Creator oder Relay broadcastet ein neues Manifest.
*   **Payload:** Das `ContentManifest` JSON.
*   **Verarbeitung:** Die Empfänger-AI prüft Signatur und `proofOfBurn`. Wenn valide und relevant -> Speichern in lokaler DB.

### 3.3 Fetch (`OAFP_FETCH_BLOB`)
Der Abruf des eigentlichen Inhalts (z.B. Bild) erfolgt **Out-of-Band** via IPFS/Hypercore oder **In-Band** via OATP, falls Privatsphäre kritisch ist.
*   **Privacy-Modus:** Der Client fordert den Blob via OATP-Relay an, um seine IP-Adresse gegenüber dem Storage-Provider zu verbergen.

### 3.4 Unlock (`OAFP_KEY_DELIVERY`)
Der Transport der `KeyDelivery`-Nachricht. Diese MUSS zwingend via OATP (verschlüsselt und authentifiziert) an den Käufer gesendet werden.

---

## 4. Ökonomie: Earn & Burn (OAPP Integration)

OAFP nutzt **OAPP**, um Spam zu bekämpfen und Monetarisierung zu ermöglichen.

### 4.1 Proof of Burn (Anti-Spam)
Um die "Öffentliche Bühne" vor Bot-Netzwerken zu schützen, erfordert das Publizieren von Manifesten eine Gebühr.
*   **Mechanismus:** Der Autor sendet eine OAPP-Zahlung an eine definierte "Burn-Adresse".
*   **Governance:** Die Mindesthöhe des Burn-Betrags (`amount`) wird dynamisch durch die **OAP Network Registry** (Governance-Parameter) festgelegt.
*   **Validierung:** Empfangende Clients prüfen via OAPP-Ledger, ob die `txId` existiert.

### 4.2 Pay-to-View (Monetarisierung)
Inhalte können verschlüsselt sein.
1.  **Manifest:** `content.encrypted = true`. Enthält `unlockCondition` (Preis, Währung).
2.  **Payment:** Konsument sendet OAPP `PaymentAuthorization` an den Autor.
3.  **Unlock:** Nach Erhalt des `PaymentReceipt` sendet der Autor-Agent die `KeyDelivery`-Nachricht via OATP an den Käufer.

---

## 5. Moderation & Trust

### 5.1 User Control Lists (Block/Mute)
Jeder Nutzer führt lokal eine Liste von blockierten DIDs.
*   **Effekt:** Manifeste und Interaktionen von blockierten DIDs werden von der AI vor der Anzeige ausgefiltert.
*   **Sharing:** Nutzer können Blocklisten abonnieren ("Shared Blocklists"), z.B. von Anti-Spam-Initiativen.

### 5.2 Community Moderation
Communities können Moderatoren ernennen.
*   **Verifikation:** Moderatoren müssen durch ein Verifiable Credential (`CommunityModeratorVC`) oder einen Eintrag in der `OAP Trusted Registry` legitimiert sein.
*   **Effekt:** Tombstones von validierten Moderatoren werden von Clients akzeptiert, auch wenn sie nicht vom Autor stammen.

### 5.3 Reputation
OAFP ist ein Konsument von Reputationsdaten. Die lokale AI gewichtet Inhalte höher, wenn der Autor eine hohe Reputation im "Web of Trust" des Nutzers hat (z.B. "Freundes-Freund" oder "Verifizierter Journalist").

---

## 6. Sicherheits-Implikationen

### 6.1 Reader Privacy
Der Abruf von Inhalten aus öffentlichen IPFS-Netzwerken exponiert die IP-Adresse des Lesers.
*   **Mitigation:** Clients SOLLTEN für den Content-Abruf Anonymisierungsnetzwerke (Tor) oder **OATP-Relays als Proxy** nutzen.
*   **Jitter:** Der Abruf des Blobs sollte zeitlich entkoppelt vom Empfang des Manifests erfolgen (Random Delay), um Timing-Analysen zu erschweren.

### 6.2 C2PA Binding (Deepfake Schutz)
Um sicherzustellen, dass ein Bild authentisch ist:
*   Das Medium (Blob) enthält C2PA-Metadaten.
*   Das Manifest enthält die OAEP-Signatur.
*   **Prüfung:** Der Client prüft, ob die C2PA-Identität mit der OAEP-Identität des Manifest-Autors übereinstimmt.

---

## 7. JSON-Schema Definitionen (Normativ)

### 7.1 `ContentManifest` Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "https://w3id.org/oafp/v1/ContentManifest",
  "type": "object",
  "required": ["type", "id", "author", "created", "content", "proof"],
  "properties": {
    "type": { "const": "ContentManifest" },
    "id": { "type": "string", "format": "uri" },
    "author": { "type": "string", "format": "uri" },
    "content": {
      "type": "object",
      "required": ["uri", "digest"],
      "properties": {
        "uri": { "type": "string" },
        "digest": { "type": "string" },
        "encrypted": { "type": "boolean", "default": false }
      }
    },
    "proofOfBurn": {
      "type": "object",
      "required": ["txId", "amount"],
      "properties": {
        "txId": { "type": "string" },
        "amount": { "type": "string" }
      }
    }
  }
}
```

### 7.2 `Interaction` Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "https://w3id.org/oafp/v1/Interaction",
  "type": "object",
  "required": ["type", "interactionType", "target", "author", "proof"],
  "properties": {
    "type": { "const": "Interaction" },
    "interactionType": {
      "enum": ["schema:LikeAction", "schema:CommentAction", "schema:ShareAction"]
    },
    "target": { "type": "string", "format": "uri" },
    "author": { "type": "string", "format": "uri" },
    "proof": { "type": "object" }
  }
}
```

### 7.3 `Tombstone` Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "https://w3id.org/oafp/v1/Tombstone",
  "type": "object",
  "required": ["type", "targetManifestId", "reason", "proof"],
  "properties": {
    "type": { "const": "Tombstone" },
    "targetManifestId": { "type": "string", "format": "uri" },
    "reason": { 
      "enum": ["AUTHOR_REQUEST", "LEGAL_TAKEDOWN", "SPAM", "NSFW"] 
    }
  }
}
```

### 7.4 `KeyDelivery` Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "https://w3id.org/oafp/v1/KeyDelivery",
  "type": "object",
  "required": ["type", "targetManifestId", "recipient", "keyPayload"],
  "properties": {
    "type": { "const": "KeyDelivery" },
    "targetManifestId": { "type": "string", "format": "uri" },
    "recipient": { "type": "string", "format": "uri" },
    "keyPayload": { "type": "string", "description": "JWE Compact String" }
  }
}
```

---

## 8. Anhang (Informativ)

### 8.1 Referenz-Implementierung
Repo: `oafp-core-rs` (Rust)

### 8.2 Storage-Adapter
OAFP ist speicheragnostisch. Die Referenz-Implementierung unterstützt **IPFS** (für öffentliche Inhalte) und **Hypercore** (für Mutable Feeds).
