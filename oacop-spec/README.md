# Open Agent Collaboration Protocol (OACoP)

[![Spec Version](https://img.shields.io/badge/spec-v1.0--RC-blue)](./RFC%20OACoP%20v1.0-RC.md)
[![Layer](https://img.shields.io/badge/OAP-Layer%202-teal)](https://oap.foundation)
[![Status](https://img.shields.io/badge/status-CODE%20FREEZE-snowflake)](./RFC%20OACoP%20v1.0-RC.md)
[![License](https://img.shields.io/badge/license-MIT%2Fwm-green)](LICENSE)

> **⚠️ STATUS ALERT: CODE FREEZE**
>
> This specification is currently a **Release Candidate (v1.0-RC)**.
> We are in **Code Freeze**. No new features will be added. Feedback is limited to iCalendar (RFC 5545) mapping correctness, optimistic locking logic, and anti-spam integration.

## 🤝 Introduction

The **Open Agent Collaboration Protocol (OACoP)** is the **Collaboration Layer (Layer 2)** of the Open Agent Protocol framework. It provides the primitives for autonomous agents to schedule meetings, delegate tasks, and manage shared resources.

Unlike traditional calendar APIs that require sharing full "Free/Busy" data, OACoP operates on a **Privacy of Availability** principle. Agents negotiate time slots contextually without revealing their owner's complete schedule.

### Core Capabilities
*   **Privacy-First Scheduling:** Negotiate specific slots via intersection logic ("I am free *here*") rather than exposing the full calendar.
*   **Optimistic Locking:** Built-in `revision` tracking to handle asynchronous scheduling conflicts robustly.
*   **Legacy Interop:** Normative mapping to **iCalendar (RFC 5545)** ensures compatibility with Outlook, Google Calendar, and Apple Calendar.
*   **Spam Protection:** Utilizing **OAPP** (Proof-of-Burn) to prevent "Calendar Spam" from unknown entities.

## 🏗 Architecture

OACoP coordinates actions, using OAPP for spam protection and OATP for delivery.

```text
[   OACoP (Collaboration)  ]  <-- "Propose Meeting" / "Assign Task"
             |
[   OAPP (Economics)  L2   ]  <-- "Burn Credits for Invite" (Anti-Spam)
             |
[   OATP (Transport)  L1   ]  <-- Asynchronous Message Delivery
             |
[      OAEP (Trust)   L0   ]  <-- Agent Identity
```

## 📂 The Specification

The full normative specification is available here:

👉 **[READ THE SPECIFICATION (v1.0-RC)](RFC%20OACoP%20v1.0-RC.md)**

### Supported Flows
*   **[Scheduling Flow](RFC%20OACoP%20v1.0-RC.md#3-protokoll-ablauf-terminfindung-scheduling-flow):** The 3-way handshake (`Propose` -> `Response` -> `Confirm`) for finding time slots.
*   **[Task Flow](RFC%20OACoP%20v1.0-RC.md#4-protokoll-ablauf-aufgaben-task-flow):** Delegating work (`AssignTask`) and tracking progress (`TaskUpdate`).
*   **Resource Booking:** Using "Room Agents" that auto-accept tasks if constraints are met.

## ⚡ Technical Standards

Implementers must strictly adhere to these primitives:

| Component | Standard / Requirement |
| :--- | :--- |
| **Data Format** | JSON-LD |
| **Interop** | **iCalendar (RFC 5545)** mapping |
| **Concurrency** | **Optimistic Locking** (Revision-based) |
| **Anti-Spam** | **OAPP Proof-of-Burn** (Transaction ID check) |

### Example: Propose Collaboration
```json
{
  "type": "ProposeCollaboration",
  "threadId": "urn:uuid:meet-1",
  "revision": 1,
  "object": { 
    "type": "Event", 
    "title": "Strategy Workshop",
    "duration": "PT30M" 
  },
  "proofOfBurn": { 
    "txId": "oap:tx:burn_123", 
    "amount": "5" 
  }
}
```

## 🛠 Reference Implementations

*   **Core Logic (Rust):** [`oap-foundation/oacop-core-rs`](https://github.com/oap-foundation/oacop-core-rs)
    *   *Includes the State Machine, Conflict Resolution, and iCal Parser.*
*   **CalDAV Bridge:** `oacop-caldav-adapter` (Beta)
    *   *Syncs OACoP states to local calendar servers.*

## 🧪 Conformance & Validation

All messages must be validated against the JSON Schemas defined in **[Section 7](RFC%20OACoP%20v1.0-RC.md#7-json-schema-definitionen-normativ)** of the RFC.

**Important:** The **Proof of Burn** mechanism (Section 3.1) is mandatory for invitations sent to agents outside of the sender's Web of Trust. Receiving agents must verify the burn transaction on the OAPP Ledger before processing the invite.

## 🤝 Contributing

We are currently in **Code Freeze**.
*   **Accepted:** Fixes for iCalendar mapping edge cases (e.g., Timezones/Recurrence), State Machine deadlocks, or OAPP integration bugs.
*   **Not Accepted:** New task types or complex project management features (Gantt) are deferred to v1.1.

Please review `CONTRIBUTING.md` before submitting PRs.

## 📄 License

Specification text: **CC BY 4.0 International**.
Schemas and Code samples: **MIT**.

---
**Maintained by the OAP Foundation**
*Coordinating the autonomous workforce.*