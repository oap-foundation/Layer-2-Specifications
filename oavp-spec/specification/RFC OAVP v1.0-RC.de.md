# RFC: Open Agent Voting Protocol (OAVP)
**Version:** 1.0 (PROPOSED STANDARD)
**Status:** CODE FREEZE
**Date:** 2025-12-03
**Layer:** 2 (Application)
**Dependencies:** OATP v1.0, OAEP v1.0, OADP v1.0

## 1. Einleitung

Das **Open Agent Voting Protocol (OAVP)** definiert den Standard für sichere, anonyme und universell verifizierbare Wahlen im digitalen Raum. Es löst das fundamentale Paradoxon digitaler Wahlen – die gleichzeitige Anforderung an Identifikation (Berechtigung) und Anonymität (Wahlgeheimnis) – durch den Einsatz moderner Kryptografie.

### 1.1 Zielsetzung und Reifegrad
Diese Version 1.0 definiert einen produktionsreifen Standard für demokratische Prozesse in Agenten-Netzwerken:
1.  **Perfect Unlinkability:** Durch **Blind Signatures** ist es mathematisch unmöglich, eine abgegebene Stimme einem Wähler zuzuordnen.
2.  **Coercion Resistance:** Mechanismen wie "Panic Credentials" schützen Wähler vor Nötigung und Stimmenkauf.
3.  **Universal Verifiability:** Die Integrität der Auszählung ist durch öffentliche Ledger (OADP) für jeden überprüfbar.

### 1.2 Integration im OAP-Stack
*   **Identität:** Die Wahlberechtigung wird via OAEP-DIDs und Verifiable Credentials geprüft.
*   **Transport:** Die Stimmabgabe erfolgt über ein **OATP-Mixnet** (Anonymisierungsnetzwerk), um IP-Metadaten zu verschleiern.
*   **Speicher:** Die "Digitale Urne" (Ballot Box) ist ein unveränderliches Log im OADP-Format.

---

## 2. Architektur & Rollen

OAVP implementiert eine strikte kryptografische Gewaltenteilung.

### 2.1 Die Behörden (Authorities)
*   **Registration Authority (RA):** Das "Wählerverzeichnis". Kennt die Identität (DID), prüft die Berechtigung und stellt `VotingTokens` aus. Sie sieht niemals die Stimme.
*   **Validation Authority (VA):** Der "Stempel". Signiert verblindete Stimmzettel. Sie sieht weder die Identität (nur das anonyme Token) noch die Stimme (da verblindet).
*   **Tallying Authority (TA):** Der "Wahlvorstand". Zählt die Stimmen in der Digital Ballot Box (DBB).

### 2.2 Der Wähler (Voter Agent)
Die Personal AI, die den komplexen Krypto-Prozess (Blinding, Mixnet-Routing) für den Nutzer abstrahiert.

---

## 3. Kryptografie (Normativ)

Um Interoperabilität und Sicherheit zu garantieren, MÜSSEN Implementierungen folgende Primitive nutzen.

### 3.1 Blind Signatures: BLS12-381
OAVP schreibt **Boneh-Lynn-Shacham (BLS)** Signaturen auf der Kurve **BLS12-381** vor.
*   **Eigenschaften:** Kurze Signaturen (48 Bytes), deterministisch, aggregierbar.
*   **Bibliotheken:** Implementierungen SOLLTEN etablierte Libraries wie `blst` (IETF Draft Standard) verwenden.

### 3.2 Mixnet-Verschlüsselung
Für das Onion-Routing im OATP-Netzwerk (Phase 4) wird geschachtelte **OATP-JWE-Verschlüsselung** verwendet, wobei jeder Hop nur den nächsten Hop entschlüsseln kann.

---

## 4. Protokoll-Ablauf (The Voting Process)

### Phase 1: Registrierung (Setup)
1.  **Announcement:** Veröffentlichung des `ElectionManifest`.
2.  **Registration:** Voter-Agent authentifiziert sich bei der RA (via OAEP).
    *   *Normalfall:* RA stellt ein valides `VotingToken` aus.
    *   *Panic-Fall:* Meldet sich der Nutzer mit einem definierten "Panic-Passwort" an (Nötigungsszenario), stellt die RA ein **Fake-Token** aus (siehe Kap. 6.1).

### Phase 2: Vorbereitung & Blinding (Lokal)
1.  **Selection:** Der Nutzer wählt (z.B. "Kandidat A").
2.  **Formatierung:** Der Agent erstellt den `Ballot`-Payload inklusive einer Zufalls-Nonce (für individuelle Verifizierbarkeit).
3.  **Blinding:** Der Agent wendet den BLS-Blinding-Faktor $r$ an -> `BlindedBallot`.

### Phase 3: Autorisierung (Signing)
1.  **Request:** Voter sendet `BlindedBallotRequest` an die VA.
2.  **Check:** VA prüft das `VotingToken` kryptografisch.
    *   *Double-Vote Check:* Die VA entwertet das Token, um mehrfache Nutzung zu verhindern.
3.  **Sign:** VA signiert den BlindedBallot mit ihrem privaten BLS-Key und sendet `BlindedBallotResponse`.

### Phase 4: Casting (Das Mixnet)
Dies ist der kritischste Schritt für die Anonymität.
1.  **Unblind:** Der Voter entfernt den Faktor $r$ und erhält die valide Signatur $\sigma$ für seinen Stimmzettel.
2.  **Mixnet-Routing:** Der Agent versendet den `CastBallot` über eine Kette von OATP-Relays (mindestens 3 Hops).
    *   **Jitter:** Jedes Relay MUSS eine zufällige Verzögerung (Random Delay) einfügen, um Timing-Analysen zu erschweren.
3.  **Drop:** Der letzte Hop wirft den Stimmzettel in die öffentliche DBB.
4.  **Receipt:** Die DBB quittiert den Empfang (via OATP zurück durch das Mixnet oder als PubSub-Broadcast).

### Phase 5: Verifizierung & Zählung (Tallying)
1.  **Validation:** Die DBB akzeptiert nur Nachrichten mit valider BLS-Signatur der VA.
2.  **Filtering:** Nach Wahlschluss veröffentlicht die RA (oder TA) die Liste der "Fake-Token-Seriennummern" (oder nutzt Zero-Knowledge-Tags), um Panic-Votes auszufiltern, ohne die Identität der Genötigten preiszugeben.
3.  **Count:** Auszählung der validen Stimmen.

---

## 5. Fehlerbehandlung (Error Handling)

Normative Fehlercodes für OAVP:

| Code | Bedeutung | Client-Aktion |
| :--- | :--- | :--- |
| **`OAVP_INVALID_TOKEN`** | Das VotingToken der RA ist ungültig/abgelaufen. | Neu bei RA registrieren. |
| **`OAVP_ALREADY_VOTED`** | Token wurde bereits verwendet. | Abbruch (Sicherheitswarnung). |
| **`OAVP_BLINDING_ERROR`** | Das Blinding-Format entspricht nicht BLS12-381. | Krypto-Bibliothek prüfen. |
| **`OAVP_ELECTION_CLOSED`** | Zeitfenster abgelaufen. | Keine Stimmabgabe mehr möglich. |
| **`OAVP_SIGNATURE_REJECTED`** | Die DBB lehnt die VA-Signatur ab. | Kritischer Alarm (VA-Schlüssel korrupt?). |

---

## 6. Sicherheits-Implikationen

### 6.1 Coercion Resistance (Schutz vor Nötigung)
OAVP schützt Wähler, die zur Herausgabe ihrer Stimme gezwungen werden.

1.  **Panic Credentials:**
    *   Nutzt der Wähler bei der RA sein "Panic-Secret", erhält er ein Token, das von der VA signiert wird und von der DBB akzeptiert wird.
    *   Für den Angreifer sieht der Prozess ("Success") und der Receipt ("Vote Accepted") identisch aus.
    *   Bei der Auszählung wird diese Stimme jedoch als ungültig eliminiert.

2.  **Re-Voting (Optionale Policy):**
    *   Wahlen KÖNNEN so konfiguriert werden, dass ein Wähler sein Token mehrfach signieren lassen kann. Nur die letzte Stimme (identifiziert durch Timestamp oder Sequenz) zählt. Dies ermöglicht es Genötigten, später ihre "falsche" Stimme zu überschreiben.

### 6.2 Mixnet-Sicherheit
*   **Vorschrift:** Ein direkter Einwurf (`POST` an DBB) ohne Mixnet ist VERBOTEN (`OAVP_INSECURE_TRANSPORT`), da die IP-Adresse den Wähler deanonymisiert.
*   **Topologie:** Clients MÜSSEN eine Route über mindestens 3 unabhängige OATP-Relays wählen ("Stratified Mixing").

---

## 7. JSON-Schema Definitionen (Normativ)

### 7.1 `ElectionManifest`

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["type", "id", "candidates", "authorities", "schedule"],
  "properties": {
    "type": { "const": "ElectionManifest" },
    "id": { "type": "string", "format": "uri" },
    "candidates": { "type": "array", "items": { "type": "string" } },
    "authorities": {
      "type": "object",
      "required": ["validationKey"],
      "properties": {
        "validationKey": { "type": "string", "description": "BLS12-381 Public Key (Hex)" }
      }
    },
    "schedule": {
      "type": "object",
      "required": ["end"],
      "properties": {
        "end": { "type": "string", "format": "date-time" }
      }
    }
  }
}
```

### 7.2 `BlindedBallotRequest`

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["type", "electionId", "votingToken", "blindedPayload"],
  "properties": {
    "type": { "const": "BlindedBallotRequest" },
    "electionId": { "type": "string", "format": "uri" },
    "votingToken": { "type": "string", "description": "JWT/VC from RA" },
    "blindedPayload": { "type": "string", "description": "BLS G1 Element (Hex)" }
  }
}
```

### 7.3 `CastBallot` (Der Einwurf)

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["type", "electionId", "selection", "signature", "nonce"],
  "properties": {
    "type": { "const": "CastBallot" },
    "electionId": { "type": "string", "format": "uri" },
    "selection": { "type": "string" }, // Der gewählte Kandidat
    "signature": { "type": "string", "description": "Unblinded BLS Signature" },
    "nonce": { "type": "string", "description": "Random nonce for individual verification" }
  }
}
```

---

## 8. Anhang (Informativ)

### 8.1 Referenz-Implementierung
Repo: `oavp-core-rs` (Rust). Nutzt die `blst` Crate für performante BLS12-381 Operationen.

### 8.2 Performance & Latenz-Budgets
Um Skalierbarkeit zu sichern, gelten folgende Richtwerte für Implementierungen:
*   **Blinding/Unblinding (Client):** < 100ms
*   **Signing (VA):** < 50ms pro Stimme (horizontal skalierbar)
*   **Mixnet Latenz (E2E):** < 60 Sekunden (durch Jitter bedingt)
*   **Tallying:** O(N) – ca. 10.000 Stimmen/Sekunde pro Core bei Signatur-Verifikation.

### 8.3 Formale Verifikation
Die OAP Foundation strebt an, die Unlinkability-Eigenschaften des Protokolls mittels **Tamarin Prover** oder **ProVerif** formal zu beweisen. Das Modell wird im Repository `oavp-verification` veröffentlicht.
