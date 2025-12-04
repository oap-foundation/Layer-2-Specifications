# RFC: Open Agent Commerce Protocol (OACP)
**Version:** 1.0 (PROPOSED STANDARD)
**Status:** CODE FREEZE
**Date:** 2025-12-02
**Layer:** 2 (Application)
**Dependencies:** OATP v1.0, OAEP v1.0, OAPP v1.0

## 1. Einleitung

Das **Open Agent Commerce Protocol (OACP)** definiert den Standard für den dezentralen, automatisierten Handel zwischen KI-Agenten. Es ersetzt proprietäre API-Silos und zentrale Marktplätze durch ein offenes, semantisches Protokoll, das Discovery (Suche), Negotiation (Verhandlung) und Settlement (Abwicklung) standardisiert.

Während OAEP die Identität sichert und OATP den Transport gewährleistet, liefert OACP die **Geschäftslogik**. Es transformiert den E-Commerce von einem katalogbasierten Modell ("Suche, was da ist") zu einem absichtsbasierten Modell ("Beschreibe, was du brauchst" – **Intent-Centric Commerce**).

### 1.1 Zielsetzung und Reifegrad
Diese Version 1.0 definiert einen produktionsreifen Standard, der folgende Kernanforderungen erfüllt:
1.  **Semantische Interoperabilität:** Nutzung von Schema.org und JSON-LD, damit Agenten Produkte und Dienstleistungen ohne menschliches Zutun verstehen.
2.  **Trustless Safety:** Integration von Treuhand-Mechanismen (Escrow) und Schlichtungsverfahren, um Handel zwischen unbekannten Parteien sicher zu machen.
3.  **Verifiable Truth:** Produkteigenschaften (z.B. "Bio", "Original") sind keine Marketing-Claims, sondern kryptografisch durch W3C Verifiable Credentials belegbar.

### 1.2 Integration im OAP-Stack
OACP ist untrennbar mit den Schwesterprotokollen verbunden:
*   **Identität & Signatur:** Alle kritischen Aktionen (Bestellung, Vertragsabschluss) werden mit OAEP-Schlüsseln signiert (`UserProof`).
*   **Transport:** Alle Nachrichten werden als verschlüsselte Payloads in OATP-Containern versendet.
*   **Zahlung:** Der Zahlungsverkehr wird an das **OAPP v1.0** delegiert, das nahtlos in den OACP-Workflow eingebettet ist.

---

## 2. Datenmodell & Semantik

OACP nutzt **JSON-LD** und basiert strikt auf dem Vokabular von **Schema.org**, erweitert um protokollspezifische Steuerungsfelder für den Agenten-Dialog.

### 2.1 Der Nachrichten-Umschlag (Base Message)
Jede OACP-Nachricht erbt von diesem Basis-Schema und wird als `payload` in einen OATP-Container verpackt.

```json
{
  "@context": ["https://schema.org", "https://w3id.org/oacp/v1"],
  "type": "OACPMessage",        // Abstrakter Basistyp
  "id": "urn:uuid:...",         // Eindeutige ID der Nachricht
  "threadId": "urn:uuid:...",   // UUIDv4, korreliert den gesamten Vorgang (Session)
  "sender": "did:key:buyer...", // Absender-DID
  "recipient": "did:web:shop.com", // Empfänger-DID
  "created": "2025-12-02T14:00:00Z"
}
```

### 2.2 Constraints (Normative Operatoren)
Um Intents (Kaufabsichten) präzise und maschinenlesbar zu formulieren, definiert OACP eine normative Liste von Operatoren für Constraints. Händler-Agenten MÜSSEN diese Operatoren interpretieren können.

#### 2.2.1 MUSS-Operatoren (Mandatory)
*   `equals`: Exakte Übereinstimmung (String, Number, Boolean).
*   `notEquals`: Ausschluss eines Wertes.
*   `lessThan`, `lessThanOrEquals`: Numerischer Vergleich (für Preise, Maße, Gewichte).
*   `greaterThan`, `greaterThanOrEquals`: Numerischer Vergleich.
*   `contains`: Prüft, ob ein Array einen bestimmten Wert enthält (z.B. "Farben enthält Rot").
*   `exists`: Prüft, ob ein Attribut überhaupt vorhanden ist (unabhängig vom Wert).

#### 2.2.2 KANN-Operatoren (Optional)
*   `regex`: Vergleich mittels Regulärer Ausdrücke (für komplexe String-Muster).
*   `inRange`: Prüft, ob ein Wert innerhalb eines definierten Intervalls liegt (`[min, max]`).

**Beispiel eines Constraint-Objekts:**
```json
{
  "property": "schema:memory", // JSON-LD Pfad zum Attribut
  "operator": "greaterThanOrEquals",
  "value": "16 GB",
  "required": true // Hard Constraint (Muss erfüllt sein)
}
```

### 2.3 Verifiable Credentials Validation (Vertrauensprüfung)
Wenn ein Angebot VCs enthält (z.B. ein Nachhaltigkeitszertifikat), MUSS der Buyer-Agent folgende Schritte validieren, bevor er das Angebot dem Nutzer als "verifiziert" präsentiert:

1.  **Trust Root:** Ist der `issuer` des VCs in einer vom Nutzer oder dem OAP-Verein akzeptierten *Trusted Registry* gelistet?
2.  **Integrity:** Ist die kryptografische Signatur (`proof`) des VCs mathematisch valide und gehört sie zum `issuer`?
3.  **Freshness:** Ist das `expirationDate` des VCs noch nicht überschritten und ist der Status in der *Revocation List* (StatusList2021) "gültig"?
4.  **Binding:** Matcht das `credentialSubject.id` des VCs (z.B. GTIN/SKU) eindeutig mit dem angebotenen Produkt?

---

## 3. Protokoll-Ablauf (The Commerce Flow)

Der Handelsprozess ist als asynchrone Zustandsmaschine modelliert. Der `threadId` verknüpft alle Nachrichten einer Transaktion.

### Phase 1: Discovery & Negotiation (Anbahnung)

**Schritt 1.1: `NegotiateRequest` (Buyer -> Merchant)**
Der Käufer-Agent sendet einen Intent. Er beschreibt *was* gesucht wird, und unter welchen Bedingungen.

```json
{
  "type": "NegotiateRequest",
  "threadId": "urn:uuid:thread-123",
  "intent": {
    "@type": "Product",
    "category": "RunningShoes"
  },
  "constraints": [
    { "property": "schema:size", "operator": "equals", "value": "42" },
    { "property": "schema:price", "operator": "lessThanOrEquals", "value": "150" }
  ],
  "requiredCredentials": ["EcoFriendlyCertified"] 
}
```

**Schritt 1.2: `OfferResponse` (Merchant -> Buyer)**
Der Händler antwortet mit einem konkreten, verbindlichen Angebot.
*   **Gültigkeit:** Das Feld `validUntil` definiert, wie lange das Angebot reserviert oder garantiert ist.
*   **Credentials:** Hier werden die geforderten Nachweise (VCs) geliefert.

### Phase 2: Commitment (Vertrag)

**Schritt 2.1: `OrderRequest` (Buyer -> Merchant)**
Der Käufer akzeptiert das Angebot. Dieser Schritt erzeugt eine rechtliche Verbindlichkeit. Um "AI-Halluzinations-Käufe" zu verhindern, erfordert dies eine kryptografische Signatur des Menschen.

#### 3.2.1 Normative Spezifikation des `UserProof`
Der `UserProof` bindet die Identität des Käufers an die spezifischen Konditionen des Angebots.

1.  **Canonicalization:** Erstellung einer kanonischen JSON-Repräsentation (JCS gemäß RFC 8785) der Felder: `threadId`, `offerId`, `price`, `currency`, `itemSku`, `timestamp`.
2.  **Hashing:** Berechnung des **BLAKE3**-Hashs dieser kanonischen Daten.
3.  **Signatur:** Signierung des Hashs mit dem privaten OAEP-Schlüssel (`Ed25519`).

```json
{
  "type": "OrderRequest",
  "acceptedOfferId": "urn:uuid:offer-987",
  "shippingAddress": { ... }, // OATP-Verschlüsselt
  "userProof": {
    "type": "OaepSignature2025",
    "created": "2025-12-02T14:05:00Z",
    "signedHash": "HASH_VALUE...",
    "signatureValue": "..."
  }
}
```

**Schritt 2.2: `OrderConfirmation` (Merchant -> Buyer)**
Der Händler akzeptiert den Vertrag, reserviert die Ware und fordert die Zahlung an.

```json
{
  "type": "OrderConfirmation",
  "orderId": "ORDER-2025-ABC",
  "status": "WaitingForPayment",
  "paymentRequest": { 
    // Struktur gemäß OAPP v1.0 Standard
    "type": "PaymentRequest",
    "amount": "12999",
    "currency": "EUR",
    "escrow": { ... } 
  }
}
```

### Phase 3: Payment & Trust

Hier erfolgt der Handover an das **OAPP v1.0**. Der Händler wartet auf den `PaymentReceipt` (Status `SETTLED` oder `ESCROW_LOCKED`).

#### 3.3.1 Escrow Release (Freigabe)
Bei Treuhand-Zahlungen ist das Geld beim Ledger gesperrt. Die Freigabe erfolgt durch den Käufer.
*   **Nachricht:** `DeliveryAcknowledgement` (Buyer -> Escrow Agent & Merchant).
*   **Inhalt:** Bestätigung des Erhalts und der Unversehrtheit.
*   **Effekt:** Der Escrow-Agent sendet daraufhin die `EscrowReleaseInstruction` an den Ledger (siehe OAPP v1.0 Kap. 3.3).

### Phase 4: Fulfillment (Lieferung)

**Schritt 4.1: `DeliveryUpdate` (Merchant -> Buyer)**
Status-Updates zur Lieferung (Versand, Tracking).

#### 3.4.1 Partial Fulfillment (Teillieferung)
Kann der Händler nicht die gesamte Menge liefern, sendet er ein Update mit `status: PARTIALLY_FULFILLED`.

*   **Berechnung der Rückerstattung (Refund):**
    Wenn die Zahlung bereits erfolgt ist, MUSS der Händler die Differenz via OAPP (`POST /refund`) zurückerstatten.
    *   *Formel:* `Refund = (MissingQuantity * UnitPrice)`.
*   **Protokoll:**
    ```json
    {
      "type": "DeliveryUpdate",
      "status": "PARTIALLY_FULFILLED",
      "fulfilledItems": [{"sku": "A", "qty": 1}],
      "pendingItems": [{"sku": "B", "qty": 1, "status": "CANCELLED"}],
      "refundData": {
        "amount": "2999",
        "currency": "EUR",
        "reason": "OUT_OF_STOCK"
      }
    }
    ```

### Phase 5: Dispute Resolution (Konfliktlösung)

Tritt ein Problem auf, wird dieser Flow aktiviert.

**Schritt 5.1: `DisputeRequest` (Buyer/Merchant -> Arbiter)**
Eine Partei ruft den im Escrow-Vertrag definierten Schlichter (Arbiter) an.
```json
{
  "type": "DisputeRequest",
  "reason": "ITEM_DAMAGED",
  "evidence": ["url:photo1", "url:log"], // OATP-Verschlüsselte Links
  "proposedResolution": "FULL_REFUND"
}
```

**Schritt 5.2: `DisputeRuling` (Arbiter -> Escrow)**
Der Arbiter prüft die Beweise und fällt ein Urteil. Er sendet eine signierte Instruktion an den Escrow-Agenten, der diese technisch beim Ledger umsetzt (Freigabe an Händler oder Rückzahlung an Käufer).

---

## 4. State Machine (Zustandsautomat)

Die Integrität des Handels basiert auf synchronisierten Zuständen zwischen Käufer und Verkäufer. Beide Agenten MÜSSEN diesen Zustandsautomaten implementieren, um Race Conditions und "Hanging States" zu verhindern.

| Status | Beschreibung | Trigger / Nächster Schritt | Timeout / Fehlerpfad |
| :--- | :--- | :--- | :--- |
| **INIT** | `NegotiateRequest` gesendet. | Empfang `OfferResponse` $\rightarrow$ `OFFER_RECEIVED` | 1h $\rightarrow$ `CANCELLED` |
| **OFFER_RECEIVED** | Angebot liegt vor. | Senden `OrderRequest` $\rightarrow$ `COMMITTED` | **24h** $\rightarrow$ `OFFER_EXPIRED` |
| **COMMITTED** | Bestellung gesendet, warte auf Bestätigung. | Empfang `OrderConfirmation` $\rightarrow$ `LOCKED` | Merchant Reject $\rightarrow$ `CANCELLED` |
| **LOCKED** | Ware reserviert. Warte auf Zahlung. | OAPP `PaymentReceipt` $\rightarrow$ `PAID` | **15 Min** $\rightarrow$ `PAYMENT_FAILED` |
| **PAID** | Zahlung bestätigt (Settled oder Escrow). | Senden `DeliveryUpdate` $\rightarrow$ `FULFILLED` | 30 Tage $\rightarrow$ `DISPUTED` (Nicht-Lieferung) |
| **FULFILLED** | Ware versendet / Service erbracht. | User Ack / Timer $\rightarrow$ `COMPLETED` | User Report $\rightarrow$ `DISPUTED` |
| **COMPLETED** | Transaktion erfolgreich abgeschlossen. | (Endzustand) | - |
| **DISPUTED** | Konflikt gemeldet. Schlichtung läuft. | `DisputeRuling` $\rightarrow$ `RESOLVED` | 7 Tage $\rightarrow$ Auto-Resolution |
| **RESOLVED** | Konflikt gelöst (Refund/Release). | (Endzustand) | - |
| **CANCELLED** | Abbruch vor Zahlung. | (Endzustand) | - |

### 4.1 Normative Timeouts & Ownership

Um zu definieren, welche Partei bei Netzwerkproblemen oder Inaktivität den Abbruch initiiert, gelten folgende Regeln:

1.  **Payment Timeout (15 Minuten):**
    *   *Verantwortung:* Der **Merchant-Agent**.
    *   *Logik:* Wenn 15 Minuten nach Senden der `OrderConfirmation` kein `PaymentReceipt` (OAPP) oder zumindest eine `PaymentAuthorization` (Status `PENDING`) eingegangen ist, wechselt der Status auf `PAYMENT_FAILED`. Die Reservierung der Ware wird aufgehoben.
    *   *Client-Sicht:* Der Buyer-Agent MUSS dem Nutzer einen Countdown anzeigen und bei Ablauf die UI sperren.

2.  **Delivery Timeout (Standard: 30 Tage):**
    *   *Verantwortung:* Der **Buyer-Agent**.
    *   *Logik:* Wenn nach 30 Tagen (oder dem im Angebot definierten `deliveryLeadTime` + Puffer) kein `DeliveryUpdate` erfolgt, sendet der Buyer-Agent automatisch einen `DisputeRequest` (Reason: `NOT_DELIVERED`). Dies sichert ab, dass Gelder im Escrow nicht ewig eingefroren bleiben.

3.  **Offer Expiry (24 Stunden):**
    *   *Verantwortung:* Der **Merchant-Agent**.
    *   *Logik:* Angebote sind standardmäßig 24 Stunden gültig, sofern im `validUntil`-Feld nicht anders definiert. Spätere `OrderRequests` werden mit `OACP_OFFER_EXPIRED` abgelehnt.

### 4.2 Error Recovery & Offer Renewal

Wenn ein Angebot abläuft (`OFFER_EXPIRED`) oder eine Zahlung fehlschlägt (`PAYMENT_FAILED`), ist die Transaktion zunächst gescheitert.

*   **Offer Renewal:** Der Merchant KANN eine `OfferExpiredNotification` senden, die ein aktualisiertes Angebot (`newOffer`) enthält (z.B. mit aktualisiertem Preis oder neuer Gültigkeit).
*   **Re-Negotiation:** Der Buyer KANN daraufhin mit einer neuen `OrderRequest` (unter Bezug auf die neue Offer-ID) reagieren, ohne den gesamten Negotiation-Prozess neu zu starten. Der `threadId` bleibt erhalten.

---

## 5. Fehlerbehandlung (Error Handling)

Wenn eine OACP-Nachricht technisch korrekt übertragen wurde (OATP), aber logisch nicht verarbeitet werden kann, MUSS der Empfänger mit einer `OACPError`-Nachricht antworten.

### 5.1 Das Fehler-Objekt
Fehlermeldungen sind JSON-LD Objekte, die sich auf den `threadId` des Vorgangs beziehen.

```json
{
  "@context": ["https://w3id.org/oacp/v1"],
  "type": "OACPError",
  "id": "urn:uuid:error-...",
  "threadId": "urn:uuid:original-thread-id",
  "code": "OACP_OFFER_EXPIRED",
  "message": "The offer 123 is no longer valid.",
  "details": { ... } // Optional: Kontextspezifische Daten (z.B. neues Angebot)
}
```

### 5.2 Normative Fehlercodes

Implementierungen MÜSSEN folgende Codes unterstützen und entsprechend der "Client-Aktion" reagieren:

| Code | Bedeutung | Auslöser | Client-Aktion |
| :--- | :--- | :--- | :--- |
| **`OACP_UNSUPPORTED_CONSTRAINT`** | Kriterium nicht erfüllbar. | `NegotiateRequest` enthält Bedingungen, die der Händler technisch nicht prüfen kann oder nicht erfüllt. | Constraints lockern und neu anfragen. |
| **`OACP_OUT_OF_STOCK`** | Ware nicht mehr verfügbar. | `OrderRequest` für ein Produkt, das zwischenzeitlich verkauft wurde (Race Condition). | Nutzer informieren, Alternative suchen. |
| **`OACP_OFFER_EXPIRED`** | Angebot abgelaufen. | `OrderRequest` erfolgt nach Ablauf von `validUntil` oder dem 24h-Standard. | Re-Negotiation starten (siehe 4.2). |
| **`OACP_INVALID_PROOF`** | Signatur ungültig. | Der `UserProof` im `OrderRequest` ist kryptografisch falsch oder gehört nicht zur DID des Absenders. | Abbruch. Wallet/Key-Setup prüfen. |
| **`OACP_PAYMENT_TIMEOUT`** | Zahlung zu spät. | Händler hat nach 15 Min (Status `LOCKED`) keinen Zahlungseingang registriert. | Neue Bestellung aufgeben. |
| **`OACP_PAYMENT_FAILED`** | Zahlung abgelehnt. | OAPP meldet Fehler (z.B. mangelnde Deckung). | Zahlungsmethode ändern und Retry via OAPP. |
| **`OACP_DISPUTE_REJECTED`** | Schlichtung abgelehnt. | `DisputeRequest` ist unbegründet oder Spam. | Manuelle Klärung außerhalb des Protokolls. |

### 5.3 Extension Points
Händler KÖNNEN eigene Fehlercodes definieren, diese MÜSSEN jedoch mit einem Vendor-Präfix beginnen (z.B. `EXT_SHIPPING_REGION_LOCKED`), um Kollisionen mit zukünftigen Standards zu vermeiden. Unbekannte Fehlercodes SOLLTEN vom Client generisch als "Transaktion fehlgeschlagen" behandelt werden.

---

## 6. Sicherheits- & Privacy-Implikationen

Da OACP darauf ausgelegt ist, Kaufabsichten (Intents) und persönliche Daten (Adressen) zu verarbeiten, gelten strengere Datenschutzanforderungen als für reine Transportprotokolle.

### 6.1 Privacy of Intent (Schutz der Kaufabsicht)
Ein `NegotiateRequest` ("Ich suche Medikament X") enthält hochsensible Daten.
*   **Risiko:** Wenn diese Requests unverschlüsselt oder mit einer persistenten Identität gesendet werden, können Händler oder Netzwerkbeobachter detaillierte psychografische Profile erstellen.
*   **Mitigation:** User Agents SOLLTEN für die Discovery-Phase (Broadcasting von Intents) anonyme Netzwerke (wie Tor oder OATP-Mesh-Routing) oder vertrauenswürdige **Discovery Proxies** nutzen, die die Anfrage vom Nutzer entkoppeln.

### 6.2 Anti-Doxing & Daten-Minimierung
*   **Lieferadressen:** Die `shippingAddress` ist das sensibelste Datum. Sie darf **ausschließlich** im `OrderRequest` (Phase 2) übermittelt werden. Ein Händler, der die Adresse bereits im `NegotiateRequest` fordert, MUSS vom Agenten als "verdächtig" markiert werden.
*   **Verschlüsselung:** Da OACP über OATP transportiert wird, sind Adressdaten gegenüber Relays geschützt ("Blind Delivery"). Der Händler erhält sie jedoch im Klartext zur Abwicklung.

### 6.3 Verhinderung von Preisdiskriminierung
Händler könnten versuchen, Preise basierend auf der Kaufhistorie oder dem vermeintlichen Wohlstand einer DID anzupassen ("Dynamic Pricing").

*   **Vorschrift (Incognito Mode):** Für die Phase 1 (`NegotiateRequest`) SOLLTEN User Agents standardmäßig rotierende, ephemere Identitäten (`did:key`) verwenden, die keine verknüpfbare Historie aufweisen.
*   **Reputation-Switch:** Erst beim `OrderRequest`, wenn Vertrauen für die Abwicklung nötig ist (z.B. "Ich bin ein zuverlässiger Zahler"), KANN der Agent auf eine persistente Reputations-DID wechseln.

### 6.4 Escrow Security (Treuhand-Sicherheit)
Das Escrow-Modell basiert auf der Unabhängigkeit des Treuhänders.
*   **Separation of Duties:** Das Escrow-Konto DARF NICHT vom Händler kontrolliert werden.
*   **Warnpflicht:** Der Buyer-Agent MUSS den Nutzer warnen, wenn:
    1.  Die DID des `escrow.agent` identisch mit der DID des `recipient` (Händlers) ist.
    2.  Der Escrow-Agent eine geringe Reputation im OAP-Trust-Netzwerk hat.

### 6.5 Schutz vor "Rogue Agents" (AI Safety)
Da KI-Modelle halluzinieren oder durch "Prompt Injection" manipuliert werden können, darf der Agent niemals autonom Geld ausgeben.
*   **UserProof-Zwang:** Händler MÜSSEN Bestellungen ablehnen (`OACP_INVALID_PROOF`), die keinen gültigen, durch menschliche Interaktion (Biometrie/PIN) erzeugten kryptografischen Beweis enthalten. Eine reine KI-Signatur reicht für `OrderRequest` nicht aus.

---

## 7. JSON-Schema (Normativ)

Um die Interoperabilität und Stabilität des Netzwerks zu gewährleisten, MÜSSEN OACP-Implementierungen eingehende Nachrichten gegen die folgenden JSON-Schemas (Draft 07) validieren. Nachrichten, die das Schema verletzen, MÜSSEN mit einem Transportfehler oder `OACP_UNSUPPORTED_CONSTRAINT` abgelehnt werden.

### 7.1 `NegotiateRequest` Schema
Definiert die Struktur für Suchanfragen und Constraints.

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "https://w3id.org/oacp/v1/NegotiateRequest",
  "type": "object",
  "required": ["type", "threadId", "intent"],
  "properties": {
    "type": { "const": "NegotiateRequest" },
    "threadId": { 
      "type": "string", 
      "pattern": "^urn:uuid:[0-9a-fA-F-]{36}$" 
    },
    "intent": {
      "type": "object",
      "required": ["@type"],
      "properties": {
        "@type": { "type": "string" }, // z.B. "Product"
        "category": { "type": "string" }
      }
    },
    "constraints": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["property", "operator", "value"],
        "properties": {
          "property": { "type": "string" },
          "operator": { 
            "enum": [
              "equals", "notEquals", 
              "lessThan", "lessThanOrEquals", 
              "greaterThan", "greaterThanOrEquals", 
              "contains", "exists", 
              "regex", "inRange"
            ] 
          },
          "required": { "type": "boolean", "default": true }
        }
      }
    },
    "requiredCredentials": {
      "type": "array",
      "items": { "type": "string" } // z.B. "EcoFriendlyCertified"
    }
  }
}
```

### 7.2 `OfferResponse` Schema
Stellt sicher, dass Angebote verbindliche Gültigkeitsdaten und Preise enthalten.

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "https://w3id.org/oacp/v1/OfferResponse",
  "type": "object",
  "required": ["type", "threadId", "offer"],
  "properties": {
    "type": { "const": "OfferResponse" },
    "threadId": { "type": "string", "pattern": "^urn:uuid:.*$" },
    "offer": {
      "type": "object",
      "required": ["id", "price", "priceCurrency", "validUntil", "itemOffered"],
      "properties": {
        "id": { "type": "string", "format": "uri" },
        "price": { "type": "number", "minimum": 0 },
        "priceCurrency": { "type": "string", "minLength": 3, "maxLength": 3 },
        "validUntil": { "type": "string", "format": "date-time" },
        "itemOffered": {
          "type": "object",
          "required": ["name"],
          "properties": {
            "sku": { "type": "string" }
          }
        }
      }
    },
    "verifiableCredentials": {
      "type": "array",
      "items": { "type": "object" } // W3C VC Struktur (extern definiert)
    }
  }
}
```

### 7.3 `OrderRequest` Schema
Validiert die kryptografische Bindung des `UserProof`.

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "https://w3id.org/oacp/v1/OrderRequest",
  "type": "object",
  "required": ["type", "threadId", "acceptedOfferId", "shippingAddress", "userProof"],
  "properties": {
    "type": { "const": "OrderRequest" },
    "threadId": { "type": "string" },
    "acceptedOfferId": { "type": "string", "format": "uri" },
    "shippingAddress": {
      "type": "object",
      "required": ["@type", "streetAddress", "addressCountry"],
      "properties": {
        "@type": { "const": "PostalAddress" }
      }
    },
    "userProof": {
      "type": "object",
      "required": ["type", "created", "signedHash", "signatureValue"],
      "properties": {
        "type": { "const": "OaepSignature2025" },
        "created": { "type": "string", "format": "date-time" },
        "signedHash": { "type": "string", "description": "BLAKE3 Hash of canonical order data" },
        "signatureValue": { "type": "string", "description": "Ed25519 Signature" }
      }
    }
  }
}
```

---

## 8. Anhang (Informativ)

### 8.1 Referenz-Implementierung

Die OAP Foundation stellt eine offizielle Open-Source-Bibliothek bereit, die die normative Logik (State Machine, Validierung) kapselt:

*   **Repository:** `oap-foundation/oacp-core-rs` (Rust)
*   **Module:**
    *   `oacp::intents`: Builder für Negotiate/Order Requests.
    *   `oacp::validation`: Logik für Constraint-Matching und VC-Prüfung.
    *   `oacp::state`: Die Zustandsmaschine für Transaktionen.

### 8.2 Migration von OACP v0.1 (Legacy / GitHub)

Für Entwickler, die bereits mit der ursprünglichen Spezifikation (`oacp-spec` auf GitHub) gearbeitet haben, sind dies die wichtigsten Änderungen in v1.0:

1.  **Transport-Layer:** OACP nutzt nicht mehr direktes HTTP/WebSockets, sondern zwingend **OATP v1.0**. Alle Nachrichten müssen in OATP-Container verpackt werden.
2.  **Asynchronität:** Der `threadId` ist nun obligatorisch, um Nachrichten in einer asynchronen Umgebung zu korrelieren.
3.  **UserProof:** Die Signatur im `OrderRequest` ist nicht mehr optional oder proprietär, sondern folgt strikt dem **OAEP JCS/BLAKE3/Ed25519** Standard.
4.  **Payment:** Die Einbettung von Zahlungsinformationen erfolgt nicht mehr generisch, sondern strikt gemäß **OAPP v1.0** (`PaymentRequest` Objekt).

### 8.3 Vollständiges Beispiel-Szenario

**Szenario:** Käufer (DID: `did:key:buyer`) sucht einen nachhaltigen Laptop. Händler (DID: `did:web:tech-store.com`) bietet einen an.

#### A. NegotiateRequest (Buyer)
```json
{
  "@context": ["https://schema.org", "https://w3id.org/oacp/v1"],
  "type": "NegotiateRequest",
  "id": "urn:uuid:msg-1",
  "threadId": "urn:uuid:thread-abc-123",
  "sender": "did:key:buyer",
  "recipient": "did:web:tech-store.com",
  "intent": {
    "@type": "Product",
    "category": "Laptop",
    "description": "High performance, eco friendly"
  },
  "constraints": [
    { "property": "schema:memory", "operator": "greaterThanOrEquals", "value": "16 GB" },
    { "property": "schema:price", "operator": "lessThan", "value": 2000 }
  ],
  "requiredCredentials": ["EcoLabelEU"]
}
```

#### B. OfferResponse (Merchant)
```json
{
  "type": "OfferResponse",
  "id": "urn:uuid:msg-2",
  "threadId": "urn:uuid:thread-abc-123",
  "offer": {
    "id": "urn:uuid:offer-999",
    "validUntil": "2025-12-03T18:00:00Z",
    "price": 1899.00,
    "priceCurrency": "EUR",
    "itemOffered": {
      "@type": "Product",
      "name": "GreenBook Pro 14",
      "sku": "GBP-14-16GB"
    }
  },
  "verifiableCredentials": [
    {
      "type": ["VerifiableCredential", "EcoLabelEU"],
      "issuer": "did:web:eu-label.org",
      "proof": { "..." }
    }
  ]
}
```

#### C. OrderRequest (Buyer)
```json
{
  "type": "OrderRequest",
  "id": "urn:uuid:msg-3",
  "threadId": "urn:uuid:thread-abc-123",
  "acceptedOfferId": "urn:uuid:offer-999",
  "shippingAddress": {
    "@type": "PostalAddress",
    "streetAddress": "Innovationsstraße 1",
    "addressCountry": "AT"
  },
  "userProof": {
    "type": "OaepSignature2025",
    "signedHash": "abc...", // Hash(offer-999 + price + sku)
    "signatureValue": "xyz..."
  }
}
```

#### D. OrderConfirmation (Merchant)
```json
{
  "type": "OrderConfirmation",
  "id": "urn:uuid:msg-4",
  "threadId": "urn:uuid:thread-abc-123",
  "orderId": "ORD-2025-7788",
  "status": "WaitingForPayment",
  "paymentRequest": {
    "type": "PaymentRequest",
    "amount": "189900", // Minor Units
    "currency": "EUR",
    "beneficiary": { "did": "did:web:tech-store.com" },
    "escrow": {
      "agent": "did:web:trustee.org",
      "releaseCondition": "DELIVERY_CONFIRMED"
    }
  }
}
```
