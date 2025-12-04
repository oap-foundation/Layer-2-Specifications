# RFC: Open Agent Collaboration Protocol (OACoP)
**Version:** 1.0 (PROPOSED STANDARD)
**Status:** CODE FREEZE
**Date:** 2025-12-03
**Layer:** 2 (Application)
**Dependencies:** OATP v1.0, OAEP v1.0, OAPP v1.0

## 1. Introduction

The **Open Agent Collaboration Protocol (OACoP)** standardizes coordination and collaboration between autonomous agents. It automates processes such as scheduling, task delegation, and resource booking, placing a primary focus on privacy protection (calendar data) and the asynchronous nature of communication.

### 1.1 Objective and Maturity Level
This Version 1.0 defines a production-ready standard that fulfills the following core requirements:
1.  **Privacy of Availability:** Agents never share their full calendar ("Is Busy"), but only explicit free slots ("Is Available") for a specific context ("Need-to-Know").
2.  **Robustness:** A `revision`-based optimistic locking system prevents conflicts during concurrent modifications.
3.  **Interoperability:** Normative mapping to iCalendar (RFC 5545) ensures compatibility with existing calendar systems.

### 1.2 Integration in the OAP Stack
*   **Identity:** Agents use OAEP-DIDs for identification.
*   **Transport:** All messages are transmitted via OATP (encrypted and asynchronous).
*   **Economy:** OAPP is utilized to economically prevent spam invitations via `proofOfBurn`.

---

## 2. Data Model & Semantics

OACoP uses JSON-LD. Messages are transported as a `payload` within OATP containers.

### 2.1 The Collaboration Context
Every object contains a `revision` (integer), which is incremented with every change to prevent "stale updates."

```json
{
  "@context": ["https://schema.org", "https://w3id.org/oacop/v1"],
  "type": "OACoPMessage",
  "id": "urn:uuid:msg-1",
  "threadId": "urn:uuid:meeting-project-x", 
  "revision": 1, // Starts at 1
  "sender": "did:key:alice...",
  "created": "2025-12-03T09:00:00Z"
}
```

### 2.2 `CollaborationObject`
Describes the Event or Task.

```json
"object": {
  "type": "Event", // or "Task"
  "title": "Strategy Workshop",
  "description": "Planning Q1 2026",
  "duration": "PT60M", 
  "location": "Virtual",
  // Optional: For complex repetitions (Recurring Events)
  "icalComponent": "BEGIN:VEVENT\nRRULE:FREQ=WEEKLY;COUNT=5...",
  
  // Optional: For Tasks
  "parentTaskId": "urn:uuid:main-project-1",
  "dependencies": ["urn:uuid:blocking-task-A"]
}
```

### 2.3 `Constraint`
Defines the boundary conditions for the negotiation.

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

### 2.4 Normative iCalendar Mapping (RFC 5545)
To ensure interoperability with Google Calendar, Outlook, etc., implementations MUST apply the following mapping:

| OACoP JSON Field | iCalendar Property | Format / Conversion |
| :--- | :--- | :--- |
| `object.type` | `COMPONENT` | `Event` -> `VEVENT`, `Task` -> `VTODO` |
| `object.title` | `SUMMARY` | UTF-8 String |
| `object.description` | `DESCRIPTION` | UTF-8 String |
| `object.startDate` | `DTSTART` | ISO 8601 UTC (e.g., `20251210T090000Z`) |
| `object.endDate` | `DTEND` | ISO 8601 UTC |
| `object.duration` | `DURATION` | ISO 8601 Duration (e.g., `PT1H`) |
| `object.location` | `LOCATION` | String |
| `id` | `UID` | UUID URN |
| `revision` | `SEQUENCE` | Integer |

*Note regarding Recurring Events:* If `icalComponent` contains an `RRULE`, the OACoP state machine treats the object as a *single* series. The resolution into individual instances occurs client-side.

---

## 3. Protocol Flow: Scheduling

### 3.1 `ProposeCollaboration` (Initiator -> Participants)
The initiator starts the process.
*   **Anti-Spam:** If the initiator does not have a high reputation with the recipient, they MUST attach an **OAPP Proof-of-Burn** (see Section 6.2).

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
Participants respond with availability.
*   **Privacy:** Only time slots falling within the requested `timeWindow` are sent ("Intersection").

```json
{
  "type": "ProposalResponse",
  "threadId": "urn:uuid:meet-1",
  "respondingToRevision": 1,
  "status": "INTERESTED", // or DECLINED, TENTATIVE
  "availableSlots": [
    { "start": "2025-12-10T09:00:00Z", "end": "2025-12-10T12:00:00Z" }
  ]
}
```

### 3.3 `ConfirmCollaboration` (Initiator -> Participants)
The initiator selects the best slot and locks the appointment. This increments the `revision`.

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

### 3.4 Conflict Resolution (Optimistic Locking)
1.  **Initiator Authority:** The initiator makes the final decision.
2.  **Revision Check:** Every agent MUST check: `incoming.revision > local.revision`.
    *   Lower/Equal: **Ignore** (Stale Update).
    *   Higher: **Accept** and update state.
3.  **Race Condition:** If a participant sends a `ProposalResponse` for Revision 1 while the initiator has already sent Revision 2 (`Confirm`), the response is discarded with error `OACOP_STALE_REVISION`.

### 3.5 Cancellation Flow
Cancellation by initiator or participant.

```json
{
  "type": "CancelCollaboration",
  "threadId": "urn:uuid:meet-1",
  "revision": 3,
  "reason": "SICKNESS"
}
```

---

## 4. Protocol Flow: Tasks

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
The worker changes the status.

```json
{
  "type": "TaskUpdate",
  "threadId": "urn:uuid:task-1",
  "status": "ACCEPTED", // DECLINED, IN_PROGRESS, COMPLETED, BLOCKED
  "progress": 25, // Percent
  "comment": "Draft started."
}
```

---

## 5. Error Handling

If a message cannot be processed logically, an `OACoPError` is sent (via OATP).

| Code | Meaning | Client Action |
| :--- | :--- | :--- |
| **`OACOP_CONFLICT`** | Slot is no longer available. | Select new slot from `availableSlots` or re-negotiate. |
| **`OACOP_STALE_REVISION`** | Message refers to outdated state. | Synchronize state and retry. |
| **`OACOP_INVALID_ICAL`** | iCal data inconsistent/unparseable. | Abort (Bug Report). |
| **`OACOP_PAYMENT_REQUIRED`** | Missing/Invalid burn for invitation. | Perform OAPP Burn and retry. |
| **`OACOP_UNSUPPORTED_CONSTRAINT`** | Constraint (e.g., Location) cannot be met. | Relax constraints. |

---

## 6. Security & Privacy Implications

### 6.1 Calendar Scraping & Fuzzy Availability
*   **Risk:** Attacker sends many requests to map out the calendar.
*   **Protection:** Agents SHOULD "noise" (Fuzzy Availability) the response for requests from unknown DIDs:
    1.  Return a maximum of 3 slots, even if 20 are free.
    2.  Slightly randomize start times (+/- 5 min), if the context allows.
    3.  Enforce rate limiting per DID.

### 6.2 OAPP Anti-Spam
For invitations from DIDs not in the "Web of Trust", the receiving agent MUST verify if a valid `proofOfBurn` is included.
*   **Mechanism:** Verification of the `txId` in the OAPP Ledger.
*   **Governance:** The minimum amount (e.g., 1 OAP Credit) is configured locally or retrieved from the OAP Registry.

---

## 7. JSON-Schema Definitions (Normative)

To ensure interoperability, implementations MUST use these schemas.

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

## 8. Appendix (Informative)

### 8.1 Reference Implementation
Repo: `oap-foundation/oacop-core-rs` (Rust)

### 8.2 Resource Sharing (Room Agents)
Rooms or devices (e.g., an MRI scanner) can be modeled as agents with their own DID.
*   **Behavior:** They automatically accept `AssignTask` (for bookings) if the slot is free in the calendar.
*   **Constraints:** They can publish specific constraints (e.g., `minDuration`, `requiresCleaningGap`).