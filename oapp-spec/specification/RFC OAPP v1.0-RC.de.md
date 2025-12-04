# RFC: Open Agent Payment Protocol (OAPP)
**Version:** 1.0 (PROPOSED STANDARD)
**Status:** CODE FREEZE
**Date:** 2025-12-02
**Layer:** 2 (Application)
**Dependencies:** OATP v1.0, OAEP v1.0, OACP v1.0

## 1. Einleitung

Das **Open Agent Payment Protocol (OAPP)** bildet das finanzielle Rückgrat des OAP-Ökosystems. Es ist die Abwicklungsschicht (Settlement Layer), die den vertraglichen Willen zweier Parteien (definiert durch OACP) in einen unwiderruflichen Werttransfer übersetzt.

OAPP fungiert als universeller Adapter. Es entkoppelt die **Autorisierung** einer Zahlung (durch den Nutzer und seinen Agenten) von der **technischen Ausführung** (durch Ledger, Banken oder Gateways). Damit ermöglicht es eine nahtlose Koexistenz von modernen Token-Ökonomien (OAP-Credits) und traditionellen Finanzsystemen (SEPA, Kreditkarten) unter einem einheitlichen Sicherheitsstandard.

### 1.1 Zielsetzung und Reifegrad
Diese Version 1.0 definiert einen produktionsreifen Standard für den sicheren Zahlungsverkehr zwischen autonomen Agenten. Sie integriert alle notwendigen Sicherheitsmechanismen für den Realbetrieb:
*   **Rechtssicherheit:** PSD2-konforme Autorisierung und eIDAS-Integration.
*   **Treuhand-Sicherheit:** Ein nativer Escrow-Mechanismus für vertrauenslosen Handel.
*   **Robustheit:** Normatives Fehlerhandling, Idempotenz und Replay-Schutz für Finanztransaktionen.

### 1.2 Anwendungsbereiche
OAPP deckt drei primäre Szenarien ab:
1.  **Internal Economy:** Direkte P2P-Transaktionen mit OAP-Credits über den Vereins-Ledger.
2.  **Open Banking:** Auslösung von Banküberweisungen (SEPA Instant) über PSD2-Brücken.
3.  **Legacy Commerce:** Integration klassischer Zahlungsdienstleister (PSPs) wie Stripe oder PayPal.

---

## 2. Datenmodell (Harmonisiert mit OACP v1.0)

OAPP nutzt JSON-LD. Die hier definierten Objekte werden als Payload im OATP-Container transportiert und sind strukturell eng mit den Geschäftsprozessen des OACP verzahnt.

### 2.1 `PaymentRequest` (Die Zahlungsaufforderung)
Dieses Objekt definiert die Konditionen der Zahlung. Es wird typischerweise vom Payee (Händler) erstellt und ist oft in einer OACP `OrderConfirmation` eingebettet.

```json
{
  "@context": ["https://w3id.org/oapp/v1"],
  "type": "PaymentRequest",
  "id": "urn:uuid:pay-req-123",
  "threadId": "urn:uuid:oacp-thread-abc", // Link zum OACP-Handel
  
  // Kaufmännische Daten (Muss mit OACP Order übereinstimmen)
  "amount": "12999", // Minor Units (Cents) als String/Integer
  "currency": "EUR", // ISO 4217 oder "OAP"
  "reference": "ORDER-2025-ABC",
  
  // Empfänger-Daten
  "beneficiary": {
    "did": "did:web:shop.com",
    "name": "Th!nk Store GmbH"
  },

  // Technische Abwicklung (Methoden-Wahl)
  "supportedMethods": [
    {
      "type": "OAP_LEDGER",
      "targetAccount": "did:web:shop.com#wallet-oap"
    },
    {
      "type": "PSD2_SEPA",
      "targetAccount": "IBAN_ENCRYPTED_BLOB..." // JWE Encrypted
    }
  ],

  // Treuhand (Optional, aus OACP übernommen)
  "escrow": {
    "required": true,
    "agent": "did:web:trustee.org",
    "releaseCondition": "DELIVERY_CONFIRMED"
  },

  "expiresAt": "2025-12-02T15:00:00Z"
}
```

### 2.2 `PaymentAuthorization` (Die Freigabe)
Die Antwort des Payer-Agenten. Sie enthält die Wahl der Zahlungsmethode und den unwiderruflichen kryptografischen Beweis der Zustimmung (`UserProof`).

#### 2.2.1 Normatives Nonce-Format & Hashing
Um Replay-Angriffe zu verhindern und Krypto-Agilität zu wahren, gelten folgende Regeln:

1.  **Nonce-Konstruktion:**
    `Base64URL( BigEndian(Timestamp_Unix_Seconds) || Random_16_Bytes )`
    *   *Timestamp:* Ermöglicht die Prüfung des 5-Minuten-Zeitfensters.
    *   *Random:* Garantiert Einmaligkeit innerhalb des Fensters.

2.  **Hash-Algorithmus:**
    Der für `signedHash` verwendete Algorithmus MUSS der Hash-Funktion der aktiven **OAEP Cipher Suite** entsprechen (z.B. **BLAKE3** für `OAEP-v1-2026`). Die Verwendung von fixem SHA-256 ist veraltet.

```json
{
  "type": "PaymentAuthorization",
  "paymentRequestId": "urn:uuid:pay-req-123",
  "selectedMethod": "OAP_LEDGER",
  
  // Der UserProof (Signiert mit OAEP Hardware Key)
  "authorizationProof": {
    "type": "OaepSignature2025",
    "created": "2025-12-02T14:10:00Z",
    "algorithm": "Ed25519", // Gemäß OAEP Suite
    "digest": "BLAKE3",      // Gemäß OAEP Suite
    
    // Hash über: paymentRequestId + amount + currency + beneficiaryDID + nonce
    "signedHash": "HASH_VALUE_BASE64...",
    "nonce": "Base64URL(Timestamp + Random)", 
    "signatureValue": "..." 
  },

  // Methodenspezifische Daten (z.B. Token)
  "methodData": {
    "payerWallet": "did:key:user#main"
  }
}
```

### 2.3 `PaymentReceipt` (Der Beleg)
Der Abschluss der Transaktion. Wird vom Zahlungsempfänger (Payee) oder dem Gateway signiert, sobald das Geld final bewegt wurde.

```json
{
  "type": "PaymentReceipt",
  "paymentRequestId": "urn:uuid:pay-req-123",
  "status": "SUCCESS",
  "transactionId": "tx_ledger_998877", // Externe ID (Bank/Ledger)
  "settledAt": "2025-12-02T14:10:05Z",
  "fee": "0",
  // Optional: Referenz bei Rückerstattungen
  "refundId": "ref_tx_998877_refund" 
}
```

---

## 3. Szenario A: OAP Ledger API (Native Token)

Dies ist das Standard-Szenario für Transaktionen innerhalb des OAP-Vereins ("Circular Economy"). Der **OAP Ledger Service** ist die zentrale Instanz, die Konten auf Basis von DIDs führt und Transaktionen finalisiert.

### 3.1 Architektur
*   **Payer Agent:** Sendet `PaymentAuthorization` via OATP an den Payee.
*   **Payee Agent:** Leitet die Authorization an die Ledger API weiter (Settlement).
*   **Ledger:** Prüft Signatur, Deckung und führt die Buchung atomar aus.

### 3.2 Ledger API Spezifikation (REST over HTTPS)
Der Ledger bietet eine öffentliche, aber authentifizierte API. Alle Requests müssen via OAEP (Bearer Token) signiert sein.

#### 3.2.1 `POST /v1/transfer` (Ausführung)
Dieser Endpunkt führt die eigentliche Wertübertragung aus.
*   **Request Body:** Das vollständige `PaymentAuthorization` Objekt + das ursprüngliche `PaymentRequest` (zur Validierung der Bedingungen).
*   **Response (Success):** `200 OK` mit `{"txId": "oap:tx:...", "status": "SETTLED"}`.

#### 3.2.2 Normative Fehlercodes (Error Handling)
Der Ledger MUSS folgende HTTP-Statuscodes und OAPP-spezifische Fehlercodes zurückgeben:

| HTTP | OAPP Code | Bedeutung | Client-Aktion |
| :--- | :--- | :--- | :--- |
| **400** | `INVALID_SIGNATURE` | OAEP-Signatur ungültig oder Nonce-Format falsch. | Abbruch (Fatal). |
| **402** | `INSUFFICIENT_FUNDS` | Kontostand des Payers zu niedrig. | User benachrichtigen. |
| **409** | `DUPLICATE_TRANSACTION` | `paymentRequestId` wurde bereits verarbeitet. | **Erfolg behandeln** (Idempotenz). Alte `txId` zurückgeben. |
| **422** | `EXPIRED_REQUEST` | Zeitstempel zu alt (außerhalb des Replay-Fensters). | Neuen Request anfordern. |
| **500** | `LEDGER_ERROR` | Interner Fehler (Konsens, DB). | Retry mit Backoff. |

### 3.3 Escrow Settlement (Treuhand-Abwicklung)

Wenn im `PaymentRequest` ein `escrow`-Objekt definiert war, fließt das Geld nicht direkt zum Händler, sondern auf ein technisches Treuhandkonto des Ledgers.

#### 3.3.1 Der Sperr- und Freigabe-Prozess
1.  **Sperrung:** Bei erfolgreichem `POST /transfer` bucht der Ledger das Geld vom Payer auf den `EscrowAccount`. Der Transaktionsstatus ist `ESCROW_LOCKED`.
2.  **Trigger:** OACP meldet `DELIVERY_CONFIRMED` oder eine Schlichtungsentscheidung (`DisputeRuling`).
3.  **Instruktion:** Der Escrow-Agent (ein dedizierter OAEP-Agent) sendet eine signierte Instruktion an den Ledger.

#### 3.3.2 `EscrowReleaseInstruction`
```json
{
  "type": "EscrowReleaseInstruction",
  "escrowTxId": "tx_ledger_lock_123",
  "action": "RELEASE",                // "RELEASE" (an Payee) oder "REFUND" (an Payer)
  "beneficiary": "did:web:shop.com",  // Ziel-DID
  "amount": "12999",
  "reason": "Delivery confirmed by buyer",
  "signature": "..."                  // Signatur des Escrow-Agents
}
```

### 3.4 Consensus Models (Governance)
Für die Version 1.0 operiert der OAP Ledger als **Single Trusted Validator**, der technisch vom OAP Verein betrieben wird.
*   **Roadmap:** Mit Version 2.0 ist der Übergang zu einem föderierten Modell (Consortium Ledger) geplant, bei dem Genossenschaftsmitglieder via PBFT-Konsens (Practical Byzantine Fault Tolerance) validieren.

### 3.5 Fee Distribution (Gebührenmodell)
Um Preistransparenz für den Käufer zu wahren ("What you see is what you pay"), gilt normativ:
*   **Payee Pays:** Transaktionsgebühren werden vom *Empfangsbetrag* abgezogen.
*   *Beispiel:* Buyer sendet 100. Fee ist 1. Ledger bucht: Payer -100, Payee +99, FeeAccount +1.

### 3.6 Refund API (Rückerstattung)
Ermöglicht dem Händler (Payee), Geld aus einer abgeschlossenen Transaktion einfach und sicher zurückzusenden.

**Endpunkt:** `POST /v1/refund`
*   **Body:**
    ```json
    {
      "originalTxId": "tx_ledger_123",
      "amount": "12999", // Kann <= Originalbetrag sein (Partial Refund)
      "reason": "ITEM_RETURNED"
    }
    ```
*   **Auth:** Der Request MUSS vom Payee (Empfänger der Original-Tx) signiert sein.
*   **Logik:** Der Ledger prüft, ob `originalTxId` existiert, ob der Anforderer der ursprüngliche Empfänger war und ob der Betrag noch verfügbar ist.

---

## 4. Szenario B: PSD2 / Open Banking (Der Bank-Adapter)

In diesem Szenario agiert OAPP als Brücke zum klassischen Bankensystem (z.B. SEPA Instant in Europa). Da der direkte Zugriff auf Bank-APIs (XS2A) regulierten Drittanbietern (TPPs) vorbehalten ist, nutzt OAPP einen zertifizierten Intermediär: die **PSD2 Bridge**.

### 4.1 Architektur und Vertrauensmodell

*   **Der Payer Agent:** Initiiert die Zahlung, hält aber selbst kein Bankzertifikat.
*   **Die PSD2 Bridge:** Ein Dienstleister (z.B. ein lizenziertes FinTech), der OATP spricht und über die notwendigen eIDAS-Zertifikate (QWAC/QSealC) verfügt, um mit Banken zu kommunizieren.

#### 4.1.1 Bridge Authentication (Sicherheit)
Da die Bridge sensible Zahlungsdaten verarbeitet, muss ihre Vertrauenswürdigkeit absolut zweifelsfrei sein.

1.  **eIDAS-Pflicht:** Die Bridge **MUSS** ein gültiges QWAC (Qualified Website Authentication Certificate) oder QSealC besitzen, das ihre Rolle als PISP (Payment Initiation Service Provider) bestätigt.
2.  **Trusted Registry:** Der Payer-Agent **MUSS** vor dem Senden von Daten prüfen, ob die DID der Bridge in der *OAP Trusted Service Registry* gelistet ist und ob das dort hinterlegte Zertifikat gültig ist.
3.  **Signierte Initiation:** Die Bridge signiert den API-Aufruf gegenüber der Bank mit ihrem eIDAS-Schlüssel, nicht mit dem User-Key.

### 4.2 Payload & Encryption (Datenschutz)

Sensible Bankdaten (IBAN, Konto-Logins, TANs) im `methodData`-Feld **DÜRFEN NIEMALS** im Klartext durch das OATP-Netzwerk gesendet werden, selbst wenn der Transportkanal verschlüsselt ist. Es gilt das Prinzip der Defense-in-Depth.

#### 4.2.1 Normative JWE-Verschlüsselung
Das Feld `accountData` innerhalb der `PaymentAuthorization` **MUSS** als **JWE Compact String** (gemäß RFC 7516) verschlüsselt sein.

*   **Algorithmus (`enc`):** **A256GCM** (AES-256-GCM).
*   **Key Management (`alg`):** **ECDH-ES** (Ephemeral Static Diffie-Hellman). Der Sender nutzt den öffentlichen Schlüssel der PSD2-Bridge (aus deren DID Document Property `keyAgreement`), um den Content Encryption Key (CEK) abzuleiten.
*   **Effekt:** Nur die Bridge (Inhaber des privaten Schlüssels) kann die IBAN entschlüsseln. Kein Relay und kein anderer Intermediär hat Zugriff.

```json
// Beispiel methodData für PSD2_SEPA
"methodData": {
  "bankId": "BIC_DER_BANK", // Unverschlüsselt für Routing
  "remittanceInformation": "Order #123",
  // Verschlüsselter Container für IBAN/Login
  "accountData": "eyJhbGciOiJFQ0RILUVTIiwiZW5jIjoiQTI1NkdDTSJ9..." 
}
```

### 4.3 Der Prozess-Ablauf (Payment Flow)

1.  **Init:** Der Payer-Agent sendet die `PaymentAuthorization` via OATP an die PSD2 Bridge.
2.  **Setup:** Die Bridge entschlüsselt die `accountData` und initiiert einen *Payment Initiation Request* bei der API der Hausbank des Nutzers.
3.  **SCA (Strong Customer Authentication):** Die Bank fordert eine Zweifaktor-Authentifizierung (z.B. Push-Nachricht auf die Banking-App des Nutzers). Dieser Schritt erfolgt "Out-of-Band".
4.  **Execution:** Nach erfolgreicher SCA führt die Bank die SEPA-Überweisung aus.
5.  **Receipt:** Die Bank bestätigt die Ausführung an die Bridge. Die Bridge erstellt daraufhin den OAPP `PaymentReceipt`, signiert ihn und sendet ihn via OATP an den Payer und den Payee (Händler).

### 4.4 Fehler-Mapping
Die Bridge MUSS Bank-Fehler auf OAPP-Status mappen:
*   *SCA abgebrochen/timeout* $\rightarrow$ `FAILED`
*   *Konto nicht gedeckt* $\rightarrow$ `FAILED` (mit Reason Code)
*   *Technische Störung Bank* $\rightarrow$ `PENDING` (Retry möglich)

---

## 5. Szenario C: PSP Integration (Stripe, PayPal)

Für die Akzeptanz von Kreditkarten, Wallets (Apple Pay/Google Pay) oder PayPal nutzt OAPP die APIs etablierter Zahlungsdienstleister (PSPs). Der Agent fungiert hier als intelligenter Orchestrator, der die Legacy-Zahlung in das OAPP-Format kapselt.

### 5.1 Architektur: "Agent-Mediated Checkout"

Da PSPs oft JavaScript-SDKs oder Web-Redirects erfordern (und KI-Agenten keine Browser sind), unterstützt OAPP zwei Integrationsmodi:

1.  **Headless (Server-Side):** Wenn der Agent bereits über ein tokenisiertes Zahlungsmittel verfügt (z.B. Stripe Customer ID `cus_...`), kann er die API direkt aufrufen.
2.  **Handoff (Interactive):** Der Payer-Agent präsentiert dem Nutzer ein natives UI-Element oder einen sicheren Webview, um die Kartendaten einzugeben.

**Der Flow (Beispiel Stripe PaymentIntents):**
1.  **Request:** Der Händler-Agent erstellt serverseitig ein `PaymentIntent` und sendet das `client_secret` im `PaymentRequest` (Feld `gatewayData`).
2.  **Authorization:** Der Payer-Agent nutzt das Secret, um die Zahlung beim PSP zu bestätigen (via SDK oder API).
3.  **Receipt Generation:** Der Händler-Agent wartet auf die Bestätigung vom PSP (Webhook) und stellt erst dann den OAPP `PaymentReceipt` aus.

### 5.2 PSP Webhook Security (Normativ)

Da die finale Bestätigung ("Geld ist da") asynchron über einen Webhook des PSPs an den Händler-Agenten erfolgt, ist dieser Endpunkt ein kritisches Angriffsziel. Ein Angreifer könnte gefälschte "Success"-Webhooks senden, um Ware ohne Bezahlung zu erhalten.

Implementierungen **MÜSSEN** folgende Sicherheitsprüfungen durchführen:

#### 5.2.1 Signatur-Validierung
Der Händler-Agent **MUSS** die kryptografische Signatur des Webhooks validieren.
*   **Mechanismus:** Berechnung eines HMAC (meist SHA-256) über den Raw Body des Webhooks mit dem vom PSP bereitgestellten *Webhook Secret* (z.B. `whsec_...`).
*   **Prüfung:** Der berechnete Hash muss mit dem Header (z.B. `Stripe-Signature` oder `Paypal-Transmission-Sig`) übereinstimmen.
*   **Fehler:** Bei Mismatch **MUSS** der Request mit HTTP 400 abgelehnt und ignoriert werden.

#### 5.2.2 Idempotenz & Replay-Schutz
Webhooks können mehrfach gesendet werden (Retry des PSPs) oder von Angreifern aufgezeichnet und wiederholt werden.
1.  **Event-ID Cache:** Der Agent **MUSS** die eindeutige ID des Webhook-Events (z.B. `evt_123...`) speichern.
2.  **Check:** Ist die ID bereits bekannt?
    *   *Ja:* Antworte sofort mit `200 OK`, aber führe keine Business-Logik aus (Idempotenz).
    *   *Nein:* Verarbeite das Event.
3.  **Zeitfenster:** Der Agent **SOLLTE** den Timestamp im Webhook-Header prüfen. Events, die älter als **5 Minuten** sind, müssen als potenzieller Replay-Angriff verworfen werden (sofern der PSP dies nicht selbst durch die Signatur abdeckt).

#### 5.2.3 State Consistency (Race Conditions)
Ein `payment.succeeded` Event darf nur verarbeitet werden, wenn die zugehörige OACP-Bestellung noch in einem validen Status ist (z.B. `LOCKED`).
*   Ist die Order bereits `CANCELLED` oder `TIMED_OUT`, muss der Agent automatisch einen **Refund** beim PSP initiieren, da die Ware nicht mehr reserviert ist.

### 5.3 Mapping auf OAPP Receipt
Nach erfolgreicher Validierung des Webhooks generiert der Händler-Agent den `PaymentReceipt`:

```json
{
  "type": "PaymentReceipt",
  "status": "SUCCESS",
  "transactionId": "pi_3Mem...", // Die ID des PSP PaymentIntent
  "gateway": "STRIPE",
  "fee": "299" // Die vom PSP erhobene Gebühr (Minor Units)
}
```

---

## 6. State Machine (Zustandsautomat)

Die Zustände einer OAPP-Transaktion bilden die Asynchronität von Banken und Blockchains ab. Um Race Conditions zu vermeiden, sind die Übergänge strikt definiert.

### 6.1 Die OAPP-Zustände

| Status | Beschreibung | Nächster Schritt / Trigger |
| :--- | :--- | :--- |
| **CREATED** | Request lokal erstellt, aber noch nicht gesendet. | Senden via OATP $\rightarrow$ `PENDING` |
| **PENDING** | An Payer gesendet. Warte auf Autorisierung. | User-Signatur $\rightarrow$ `AUTHORIZED` |
| **AUTHORIZED** | `PaymentAuthorization` liegt vor. | Einreichen bei Ledger/Bank $\rightarrow$ `PROCESSING` |
| **SCA_REQUIRED** | (Nur PSD2) Bank wartet auf 2-Faktor-Auth. | User bestätigt in Bank-App $\rightarrow$ `PROCESSING` |
| **PROCESSING** | Technische Ausführung läuft (Mining, Batch). | Erfolg $\rightarrow$ `SETTLED`<br>Fehler $\rightarrow$ `FAILED`<br>Escrow $\rightarrow$ `ESCROW_LOCKED` |
| **ESCROW_LOCKED**| Geld liegt beim Treuhänder. | Release-Befehl $\rightarrow$ `SETTLED`<br>Refund-Befehl $\rightarrow$ `REFUNDED` |
| **SETTLED** | Geld ist final beim Empfänger angekommen. | (Endzustand) |
| **FAILED** | Fehler (keine Deckung, Tech-Fehler, Timeout). | Retry oder Abbruch $\rightarrow$ (Endzustand) |
| **REFUNDED** | Rückzahlung erfolgreich durchgeführt. | (Endzustand) |

### 6.2 Mapping auf OACP (Order Status)

Damit der Handelsablauf (OACP) weiß, wann er fortfahren kann, definiert OAPP normative Mappings auf die OACP-Zustände.

| OAPP Status | Empfohlener OACP Order Status | Bedeutung für den Handel |
| :--- | :--- | :--- |
| `PENDING` | `LOCKED` | Bestellung ist reserviert, Zahlung steht aus. |
| `AUTHORIZED` | `LOCKED` (Payment Processing) | Zahlung ist unterwegs, Ware bleibt reserviert. |
| `ESCROW_LOCKED` | `PAID` (bzw. `ESCROWED`) | Geld ist gesichert. **Versand kann erfolgen.** |
| `SETTLED` | `PAID` | Geld ist beim Händler. **Versand kann erfolgen.** |
| `FAILED` | `PAYMENT_FAILED` | Händler bricht Reservierung ab (nach Retry). |
| `REFUNDED` | `CANCELLED` | Bestellung wurde rückabgewickelt. |

**Wichtig:** Für den Händler ist der Status `ESCROW_LOCKED` gleichbedeutend mit `PAID` im Sinne der Versandsicherheit. Das Geld ist zwar noch nicht auf seinem Konto, aber es hat den Payer verlassen und ist durch den Smart Contract oder Treuhänder gesichert. Er kann gefahrlos versenden.

---

Hier ist **Abschnitt 7 (Sicherheits-Betrachtungen)** für den **RFC OAPP v1.0**.

Dieser Abschnitt bündelt die Sicherheitsmechanismen für alle drei Szenarien und definiert normativ den **Replay-Schutz** sowie die **Separation of Duties**, um Betrug auf Protokollebene zu verhindern.

***

## 7. Sicherheits-Betrachtungen

### 7.1 Separation of Concerns (Architektur-Sicherheit)
Das OAP-Framework erzwingt eine strikte Trennung der Kompetenzen:
*   Das **OACP**-Protokoll verwaltet den *Vertrag* (Warenkorb, Lieferbedingungen).
*   Das **OAPP**-Protokoll verwaltet das *Geld*.
*   **Vorschrift:** Ein Händler-Agent (OACP) darf niemals direkten Zugriff auf die Wallet-Keys des Nutzers haben. Eine Zahlung erfordert immer eine dedizierte `PaymentAuthorization`, die vom Nutzer (bzw. dessen Payer-Agenten) separat signiert wird.

### 7.2 PSD2-Bridge & Daten-Verschlüsselung
Für Szenario B (Open Banking) gelten verschärfte Regeln, da IBANs und Zugangsdaten verarbeitet werden.
*   **Defense in Depth:** Auch wenn der OATP-Transportkanal verschlüsselt ist, MÜSSEN sensible Bankdaten (`methodData`) zusätzlich via JWE verschlüsselt werden (siehe 4.2.1), sodass nur die zertifizierte PSD2-Bridge sie lesen kann.
*   **Tokenization:** Wo möglich, SOLLTEN Agenten nur temporäre Zugriffstoken (AIS/PIS Tokens) speichern, niemals die Online-Banking-Credentials (PIN/TAN) des Nutzers.

### 7.3 Replay-Schutz & Idempotenz

#### 7.3.1 Ledger Replay Cache (Normativ)
Der OAP Ledger MUSS einen **Replay-Cache** implementieren, um Double-Spending durch wiederholtes Senden derselben signierten Nachricht zu verhindern.

*   **Cache Key:** `SHA256(paymentRequestId || payerDID)`.
*   **Zeitfenster:** Das Fenster beträgt **5 Minuten**.
*   **Validierungs-Logik:**
    1.  **Timestamp Check:** Ist `created` im `authorizationProof` älter als 5 Minuten?
        *   *Ja:* **Reject** (`422 EXPIRED_REQUEST`).
    2.  **Cache Lookup:** Ist der Key im Cache?
        *   *Ja:* **Idempotenz-Fall.** Gib das Ergebnis der ursprünglichen Transaktion zurück (`200 OK` mit alter `txId`), führe aber KEINE erneute Buchung durch.
    3.  **New Transaction:** Führe Buchung durch und speichere Key im Cache (mit TTL 5 Min).

### 7.4 Separation of Duties (Betrugsprävention)
Um Kollusion im Treuhand-Fall zu verhindern:
*   **Regel:** Der **Escrow-Agent** DARF NIEMALS identisch mit dem **Payee** (Händler) sein. Die DIDs müssen unterschiedlich sein und idealerweise zu verschiedenen juristischen Personen gehören.
*   **Client-Check:** Der Payer-Agent SOLLTE den Nutzer warnen, wenn `beneficiary` und `escrow.agent` dieselbe Domain oder Root-Identität verwenden.

### 7.5 Hash-Agilität vs. Banking-Standards
Während OAEP moderne Algorithmen wie **BLAKE3** favorisiert, verlangen externe Banken-APIs oft **SHA-256**.
*   **Intern (OAP Ledger):** Signaturen MÜSSEN der aktiven OAEP Cipher Suite folgen (aktuell BLAKE3).
*   **Extern (PSD2/PSP):** Für die Kommunikation mit Legacy-Systemen (z.B. Webhook-Validierung) MÜSSEN die dort vorgeschriebenen Standards (meist HMAC-SHA256) verwendet werden. Implementierungen müssen diesen Kontextwechsel sauber handhaben.

---

## 8. JSON-Schema Definitionen (Normativ)

Um die Interoperabilität sicherzustellen, MÜSSEN Implementierungen eingehende OAPP-Nachrichten gegen die folgenden JSON-Schemas (Draft 07) validieren.

### 8.1 `PaymentRequest` Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "https://w3id.org/oapp/v1/PaymentRequest",
  "type": "object",
  "required": ["type", "id", "amount", "currency", "beneficiary"],
  "properties": {
    "type": { "const": "PaymentRequest" },
    "id": { "type": "string", "format": "uri" },
    "amount": { "type": "string", "pattern": "^[0-9]+$" },
    "currency": { "type": "string", "minLength": 3, "maxLength": 4 },
    "beneficiary": {
      "type": "object",
      "required": ["did"],
      "properties": {
        "did": { "type": "string", "format": "uri" }
      }
    },
    "escrow": {
      "type": "object",
      "required": ["agent"],
      "properties": {
        "agent": { "type": "string", "format": "uri" }
      }
    }
  }
}
```

### 8.2 `PaymentAuthorization` Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "https://w3id.org/oapp/v1/PaymentAuthorization",
  "type": "object",
  "required": ["type", "paymentRequestId", "selectedMethod", "authorizationProof"],
  "properties": {
    "type": { "const": "PaymentAuthorization" },
    "paymentRequestId": { "type": "string", "format": "uri" },
    "selectedMethod": { "enum": ["OAP_LEDGER", "PSD2_SEPA", "PSP_STRIPE"] },
    "authorizationProof": {
      "type": "object",
      "required": ["type", "created", "signedHash", "nonce", "signatureValue"],
      "properties": {
        "type": { "const": "OaepSignature2025" },
        "created": { "type": "string", "format": "date-time" },
        "signedHash": { "type": "string" },
        "nonce": { "type": "string" },
        "signatureValue": { "type": "string" }
      }
    }
  }
}
```

### 8.3 `PaymentReceipt` Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "https://w3id.org/oapp/v1/PaymentReceipt",
  "type": "object",
  "required": ["type", "paymentRequestId", "status", "transactionId"],
  "properties": {
    "type": { "const": "PaymentReceipt" },
    "status": { "enum": ["SUCCESS", "PENDING", "FAILED"] },
    "transactionId": { "type": "string" },
    "fee": { "type": "string", "pattern": "^[0-9]+$" }
  }
}
```

---

## 9. Anhang (Informativ)

Dieser Abschnitt enthält Beispiele und Hilfestellungen für Implementierer.

### 9.1 Referenz-Implementierungen

Die OAP Foundation stellt Open-Source-Bibliotheken bereit, die diesen Standard implementieren:

*   **Core Logic (Rust):** `oap-ledger-rs`
    *   Enthält die normative Logik für Signatur-Validierung, State-Machine und Ledger-Funktionen.
*   **PHP SDK:** `oapp-php`
    *   Eine Referenz-Implementierung für Händler-Systeme (z.B. WooCommerce, Magento Plugins).
*   **Dart/Flutter SDK:** `oapp-dart`
    *   Für die Integration in Wallet-Apps und Mobile Clients.

### 9.2 Vollständiges JSON-Beispiel (Szenario A: OAP-Ledger)

Nachfolgend ein vollständiger Trace einer Transaktion ("Kauf eines E-Books für 12,99 OAP").

#### A. PaymentRequest (vom Händler)
```json
{
  "@context": ["https://w3id.org/oapp/v1"],
  "type": "PaymentRequest",
  "id": "urn:uuid:req-550e8400-e29b-41d4-a716-446655440000",
  "threadId": "urn:uuid:thread-98765432-1234-5678-9abc-def012345678",
  "amount": "1299",
  "currency": "OAP",
  "description": "E-Book: The Future of AI",
  "beneficiary": {
    "did": "did:web:books.example.com",
    "name": "Example Books Ltd."
  },
  "supportedMethods": [
    { "type": "OAP_LEDGER", "targetAccount": "did:web:books.example.com" }
  ],
  "expiresAt": "2025-12-02T16:00:00Z"
}
```

#### B. PaymentAuthorization (vom Käufer)
```json
{
  "type": "PaymentAuthorization",
  "paymentRequestId": "urn:uuid:req-550e8400-e29b-41d4-a716-446655440000",
  "selectedMethod": "OAP_LEDGER",
  "authorizationProof": {
    "type": "OaepSignature2025",
    "created": "2025-12-02T15:30:00Z",
    "algorithm": "Ed25519",
    "digest": "BLAKE3",
    // Base64URL( Timestamp(4 bytes) || Random(16 bytes) )
    "nonce": "AAABjPz8_30Qh3_z8_30Qh3_z8_30Qh3",
    "signedHash": "K7gNU3kbf...", 
    "signatureValue": "OpP..."
  }
}
```

#### C. PaymentReceipt (vom Ledger via Händler)
```json
{
  "type": "PaymentReceipt",
  "paymentRequestId": "urn:uuid:req-550e8400-e29b-41d4-a716-446655440000",
  "status": "SUCCESS",
  "transactionId": "oap:tx:8374928374928374928374",
  "settledAt": "2025-12-02T15:30:02Z",
  "fee": "0"
}
```

### 9.3 Test-Vektor für die Signatur-Erstellung

Um sicherzustellen, dass verschiedene Implementierungen denselben Hash signieren, wird folgender kanonischer Input definiert.

**Input-Parameter:**
*   `paymentRequestId`: `urn:uuid:test-123`
*   `amount`: `100`
*   `currency`: `EUR`
*   `beneficiaryDID`: `did:web:test.com`
*   `nonce`: `testnonce`

**Kanonischer String (JCS):**
`{"amount":"100","beneficiary":"did:web:test.com","currency":"EUR","nonce":"testnonce","requestId":"urn:uuid:test-123"}`

**Erwarteter Hash (BLAKE3):**
*   Hex: `b4c5...` *(Hinweis: Dieser Wert muss in der Referenz-Implementierung validiert werden)*.

### 9.4 Mapping-Tabelle für Externe Status

Hilfestellung für die Implementierung von Adaptern (Szenario B und C).

| OAPP Status | PSD2 (Sepa) Status | Stripe PaymentIntent Status |
| :--- | :--- | :--- |
| `PENDING` | `RCVD` (Received) | `requires_payment_method` |
| `AUTHORIZED` | `ACCP` (Accepted) | `processing` |
| `SCA_REQUIRED` | `PDNG` (Pending SCA) | `requires_action` |
| `SETTLED` | `ACSC` (Accepted Settlement Completed) | `succeeded` |
| `FAILED` | `RJCT` (Rejected) | `canceled` / `requires_payment_method` |
