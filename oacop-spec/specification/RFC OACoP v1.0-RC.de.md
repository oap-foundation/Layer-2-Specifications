# RFC: Open Agent Collaboration Protocol (OACoP)
**Version:** 1.0 (PROPOSED STANDARD)
**Status:** CODE FREEZE
**Date:** 2025-12-03
**Layer:** 2 (Application)
**Dependencies:** OATP v1.0, OAEP v1.0, OAPP v1.0

## 1. Einleitung

Das **Open Agent Collaboration Protocol (OACoP)** standardisiert die Koordination und Zusammenarbeit zwischen autonomen Agenten. Es automatisiert Prozesse wie Terminfindung, Aufgaben-Delegation und Ressourcen-Buchung, wobei der Schutz der Privatsphäre (Kalenderdaten) und die asynchrone Natur der Kommunikation im Vordergrund stehen.

### 1.1 Zielsetzung und Reifegrad
Diese Version 1.0 definiert einen produktionsreifen Standard, der folgende Kernanforderungen erfüllt:
1.  **Privacy of Availability:** Agenten teilen niemals ihren vollen Kalender ("Is Busy"), sondern nur explizite Freiräume ("Is Available") für einen spezifischen Kontext ("Need-to-Know").
2.  **Robustheit:** Ein `revision`-basiertes Optimistic Locking System verhindert Konflikte bei gleichzeitigen Änderungen.
3.  **Interoperabilität:** Normatives Mapping zu iCalendar (RFC 5545) sichert die Kompatibilität mit bestehenden Kalendersystemen.

### 1.2 Integration im OAP-Stack
*   **Identität:** Agenten nutzen OAEP-DIDs zur Identifikation.
*   **Transport:** Alle Nachrichten werden via OATP (verschlüsselt und asynchron) übertragen.
*   **Ökonomie:** OAPP wird genutzt, um Spam-Einladungen durch `proofOfBurn` wirtschaftlich zu verhindern.

---

## 2. Datenmodell & Semantik

OACoP nutzt JSON-LD. Nachrichten werden als `payload` in OATP-Containern transportiert.

### 2.1 Der Kollaborations-Kontext
Jedes Objekt enthält eine `revision` (Ganzzahl), die bei jeder Änderung inkrementiert wird, um "Stale Updates" zu verhindern.

```json
{
  "@context": ["https://schema.org", "https://w3id.org/oacop/v1"],
  "type": "OACoPMessage",
  "id": "urn:uuid:msg-1",
  "threadId": "urn:uuid:meeting-project-x", 
  "revision": 1, // Startet bei 1
  "sender": "did:key:alice...",
  "created": "2025-12-03T09:00:00Z"
}
```

### 2.2 `CollaborationObject`
Beschreibt das Event oder den Task.

```json
"object": {
  "type": "Event", // oder "Task"
  "title": "Strategy Workshop",
  "description": "Planning Q1 2026",
  "duration": "PT60M", 
  "location": "Virtual",
  // Optional: Für komplexe Wiederholungen (Recurring Events)
  "icalComponent": "BEGIN:VEVENT\nRRULE:FREQ=WEEKLY;COUNT=5...",
  
  // Optional: Für Tasks
  "parentTaskId": "urn:uuid:main-project-1",
  "dependencies": ["urn:uuid:blocking-task-A"]
}
```

### 2.3 `Constraint`
Definiert die Rahmenbedingungen für die Aushandlung.

```json
"constraints": {
  "timeWindow": {
    "start": "2025-12-10T08:00:00Z",
    "end": "2025-12-12T18:00:00Z"
  },
  "participants": ["did:key:bob...", "did:key:charlie..."],
  "requiredParticipants": ["did:key:bob..."]
}
```

### 2.4 Normatives iCalendar Mapping (RFC 5545)
Um Interoperabilität mit Google Calendar, Outlook & Co. zu sichern, MÜSSEN Implementierungen folgendes Mapping anwenden:

| OACoP JSON Field | iCalendar Property | Format / Konvertierung |
| :--- | :--- | :--- |
| `object.type` | `COMPONENT` | `Event` -> `VEVENT`, `Task` -> `VTODO` |
| `object.title` | `SUMMARY` | UTF-8 String |
| `object.description` | `DESCRIPTION` | UTF-8 String |
| `object.startDate` | `DTSTART` | ISO 8601 UTC (z.B. `20251210T090000Z`) |
| `object.endDate` | `DTEND` | ISO 8601 UTC |
| `object.duration` | `DURATION` | ISO 8601 Duration (z.B. `PT1H`) |
| `object.location` | `LOCATION` | String |
| `id` | `UID` | UUID URN |
| `revision` | `SEQUENCE` | Integer |

*Hinweis zu Recurring Events:* Wenn `icalComponent` eine `RRULE` enthält, behandelt die OACoP-State-Machine das Objekt als *eine* Serie. Die Auflösung in einzelne Instanzen erfolgt client-seitig.

---

## 3. Protokoll-Ablauf: Terminfindung (Scheduling Flow)

### 3.1 `ProposeCollaboration` (Initiator -> Participants)
Der Initiator startet den Prozess.
*   **Anti-Spam:** Wenn der Initiator keine hohe Reputation beim Empfänger hat, MUSS er einen **OAPP Proof-of-Burn** beifügen (siehe Kap. 6.2).

```json
{
  "type": "ProposeCollaboration",
  "threadId": "urn:uuid:meet-1",
  "revision": 1,
  "object": { "type": "Event", "duration": "PT30M" },
  "constraints": { ... },
  "proofOfBurn": { 
    "txId": "oap:tx:burn_123", 
    "amount": "5" 
  }
}
```

### 3.2 `ProposalResponse` (Participants -> Initiator)
Teilnehmer antworten mit Verfügbarkeit.
*   **Privacy:** Es werden nur Zeitfenster gesendet, die im angefragten `timeWindow` liegen ("Intersection").

```json
{
  "type": "ProposalResponse",
  "threadId": "urn:uuid:meet-1",
  "respondingToRevision": 1,
  "status": "INTERESTED", // oder DECLINED, TENTATIVE
  "availableSlots": [
    { "start": "2025-12-10T09:00:00Z", "end": "2025-12-10T12:00:00Z" }
  ]
}
```

### 3.3 `ConfirmCollaboration` (Initiator -> Participants)
Der Initiator wählt den besten Slot und fixiert den Termin. Dies erhöht die `revision`.

```json
{
  "type": "ConfirmCollaboration",
  "threadId": "urn:uuid:meet-1",
  "revision": 2,
  "finalSlot": {
    "start": "2025-12-10T10:00:00Z",
    "end": "2025-12-10T10:30:00Z"
  }
}
```

### 3.4 Konfliktlösung (Optimistic Locking)
1.  **Initiator Authority:** Der Initiator entscheidet final.
2.  **Revision Check:** Jeder Agent MUSS prüfen: `incoming.revision > local.revision`.
    *   Niedriger/Gleich: **Ignorieren** (Stale Update).
    *   Höher: **Akzeptieren** und State aktualisieren.
3.  **Race Condition:** Sendet ein Teilnehmer eine `ProposalResponse` für Revision 1, während der Initiator schon Revision 2 (`Confirm`) gesendet hat, wird die Response mit Fehler `OACOP_STALE_REVISION` verworfen.

### 3.5 Cancellation Flow
Abbruch durch Initiator oder Teilnehmer.

```json
{
  "type": "CancelCollaboration",
  "threadId": "urn:uuid:meet-1",
  "revision": 3,
  "reason": "SICKNESS"
}
```

---

## 4. Protokoll-Ablauf: Aufgaben (Task Flow)

### 4.1 `AssignTask` (Manager -> Worker)
```json
{
  "type": "AssignTask",
  "threadId": "urn:uuid:task-1",
  "object": { 
    "type": "Task", 
    "title": "Report",
    "parentTaskId": "urn:uuid:project-alpha"
  },
  "deadline": "2025-12-20T12:00:00Z"
}
```

### 4.2 `TaskUpdate` (Worker -> Manager)
Der Worker ändert den Status.

```json
{
  "type": "TaskUpdate",
  "threadId": "urn:uuid:task-1",
  "status": "ACCEPTED", // DECLINED, IN_PROGRESS, COMPLETED, BLOCKED
  "progress": 25, // Prozent
  "comment": "Draft started."
}
```

---

## 5. Fehlerbehandlung (Error Handling)

Wenn eine Nachricht logisch nicht verarbeitet werden kann, wird ein `OACoPError` (via OATP) gesendet.

| Code | Bedeutung | Client-Aktion |
| :--- | :--- | :--- |
| **`OACOP_CONFLICT`** | Slot nicht mehr verfügbar. | Neuen Slot aus `availableSlots` wählen oder Re-Negotiation. |
| **`OACOP_STALE_REVISION`** | Nachricht bezieht sich auf veralteten State. | State synchronisieren und Retry. |
| **`OACOP_INVALID_ICAL`** | iCal-Daten inkonsistent/unparseable. | Abbruch (Bug Report). |
| **`OACOP_PAYMENT_REQUIRED`** | Fehlender/Ungültiger Burn für Einladung. | OAPP Burn durchführen und Retry. |
| **`OACOP_UNSUPPORTED_CONSTRAINT`** | Constraint (z.B. Location) nicht erfüllbar. | Constraints lockern. |

---

## 6. Sicherheits- & Privacy-Implikationen

### 6.1 Kalender-Scraping & Fuzzy Availability
*   **Risiko:** Angreifer sendet viele Anfragen, um den Kalender zu mappen.
*   **Schutz:** Agenten SOLLTEN bei Anfragen von unbekannten DIDs die Antwort "verrauschen" (Fuzzy Availability):
    1.  Maximal 3 Slots zurückgeben, auch wenn 20 frei sind.
    2.  Startzeiten leicht randomisieren (+/- 5 Min), wenn es der Kontext erlaubt.
    3.  Rate Limiting pro DID erzwingen.

### 6.2 OAPP Anti-Spam
Für Einladungen von DIDs, die nicht im "Web of Trust" sind, MUSS der Empfänger-Agent prüfen, ob ein valider `proofOfBurn` enthalten ist.
*   **Mechanismus:** Überprüfung der `txId` im OAPP-Ledger.
*   **Governance:** Die Mindesthöhe (z.B. 1 OAP Credit) wird lokal konfiguriert oder aus der OAP Registry bezogen.

---

## 7. JSON-Schema Definitionen (Normativ)

Um Interoperabilität zu sichern, MÜSSEN Implementierungen diese Schemata nutzen.

### 7.1 `ProposeCollaboration`

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["type", "threadId", "revision", "object"],
  "properties": {
    "type": { "const": "ProposeCollaboration" },
    "threadId": { "type": "string", "format": "uri" },
    "revision": { "type": "integer", "minimum": 1 },
    "object": {
      "type": "object",
      "required": ["type"],
      "properties": {
        "type": { "enum": ["Event", "Task"] },
        "duration": { "type": "string", "pattern": "^P" } // ISO 8601 Duration
      }
    },
    "proofOfBurn": {
      "type": "object",
      "required": ["txId"],
      "properties": { "txId": { "type": "string" } }
    }
  }
}
```

### 7.2 `ProposalResponse`

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["type", "threadId", "respondingToRevision", "status"],
  "properties": {
    "type": { "const": "ProposalResponse" },
    "respondingToRevision": { "type": "integer" },
    "status": { "enum": ["INTERESTED", "DECLINED", "TENTATIVE"] },
    "availableSlots": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["start", "end"],
        "properties": {
          "start": { "type": "string", "format": "date-time" },
          "end": { "type": "string", "format": "date-time" }
        }
      }
    }
  }
}
```

### 7.3 `ConfirmCollaboration`

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["type", "threadId", "revision", "finalSlot"],
  "properties": {
    "type": { "const": "ConfirmCollaboration" },
    "revision": { "type": "integer" },
    "finalSlot": {
      "type": "object",
      "required": ["start", "end"],
      "properties": {
        "start": { "type": "string", "format": "date-time" },
        "end": { "type": "string", "format": "date-time" }
      }
    }
  }
}
```

### 7.4 `AssignTask`

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["type", "threadId", "object", "deadline"],
  "properties": {
    "type": { "const": "AssignTask" },
    "threadId": { "type": "string", "format": "uri" },
    "object": {
      "type": "object",
      "required": ["type", "title"],
      "properties": {
        "type": { "const": "Task" },
        "title": { "type": "string" },
        "parentTaskId": { "type": "string", "format": "uri" },
        "dependencies": { "type": "array", "items": { "type": "string" } }
      }
    },
    "deadline": { "type": "string", "format": "date-time" }
  }
}
```

### 7.5 `TaskUpdate`

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["type", "threadId", "status"],
  "properties": {
    "type": { "const": "TaskUpdate" },
    "threadId": { "type": "string", "format": "uri" },
    "status": { "enum": ["ACCEPTED", "DECLINED", "IN_PROGRESS", "COMPLETED", "BLOCKED"] },
    "progress": { "type": "integer", "minimum": 0, "maximum": 100 },
    "comment": { "type": "string" }
  }
}
```

### 7.6 `CancelCollaboration`

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["type", "threadId", "revision", "reason"],
  "properties": {
    "type": { "const": "CancelCollaboration" },
    "threadId": { "type": "string", "format": "uri" },
    "revision": { "type": "integer" },
    "reason": { "type": "string" }
  }
}
```

---

## 8. Anhang (Informativ)

### 8.1 Referenz-Implementierung
Repo: `oap-foundation/oacop-core-rs` (Rust)

### 8.2 Resource Sharing (Raum-Agenten)
Räume oder Geräte (z.B. ein MRI-Scanner) können als Agenten mit eigener DID modelliert werden.
*   **Verhalten:** Sie akzeptieren `AssignTask` (für Buchungen) automatisch, wenn der Slot im Kalender frei ist.
*   **Constraints:** Sie können spezielle Constraints publizieren (z.B. `minDuration`, `requiresCleaningGap`).
