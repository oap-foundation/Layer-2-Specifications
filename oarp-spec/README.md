# Open Agent Robotics Protocol (OARP)

[![Spec Version](https://img.shields.io/badge/spec-v1.0--RC-blue)](./RFC%20OARP%20v1.0-RC.md)
[![Layer](https://img.shields.io/badge/OAP-Layer%202-slate)](https://oap.foundation)
[![Status](https://img.shields.io/badge/status-CODE%20FREEZE-snowflake)](./RFC%20OARP%20v1.0-RC.md)
[![License](https://img.shields.io/badge/license-MIT%2Fwm-green)](LICENSE)

> **⚠️ STATUS ALERT: CODE FREEZE**
>
> This specification is currently a **Release Candidate (v1.0-RC)**.
> We are in **Code Freeze**. No new features will be added. Feedback is strictly limited to Safety Constraints logic, ROS 2 bridging compatibility, and JSON Schema validation.

## 🤖 Introduction

The **Open Agent Robotics Protocol (OARP)** is the **Robotics Layer (Layer 2)** of the Open Agent Protocol framework. It defines the standard language for autonomous software agents (Personal AIs) to control physical actuators, drones, and smart home devices.

OARP acts as the **Semantic Intent Layer**. Unlike low-level control frameworks (like ROS DDS), OARP defines *what* needs to be done ("Go to the Kitchen") and the *safety boundaries* ("Max speed 0.5m/s"), while leaving the *how* to the device's local firmware.

### Core Capabilities
*   **Safety First:** Implements a strict "User-in-the-Loop" policy via cryptographic **UserProofs** for critical actions (e.g., unlocking doors, operating heavy machinery).
*   **Semantic Interoperability:** Uses a normative ontology (`oarp:cap`) so agents can understand the capabilities of unknown devices instantly.
*   **Real-Time Ready:** Integrates **WebRTC** signalling over OATP for low-latency video feeds and teleoperation.
*   **ROS 2 Bridgeable:** Designed to sit cleanly on top of ROS 2 / DDS stacks.

## 🏗 Architecture

OARP sits between the high-level AI reasoning and the low-level hardware control loops.

```text
[    Controller Agent (AI)   ]  <-- "Task: Locate User in Kitchen"
             |
[     OARP (Intent/Safe)     ]  <-- Semantic Commands & Safety Constraints
             |
[   OATP (Transport)    L1   ]  <-- Encrypted / Async Delivery
             |
[    Physical Agent (Robot)  ]  <-- "Translating Intent to Motor Velocity"
      |               |
   [ROS 2]         [WebRTC]
```

## 📂 The Specification

The full normative specification is available here:

👉 **[READ THE SPECIFICATION (v1.0-RC)](RFC%20OARP%20v1.0-RC.md)**

### Supported Flows
*   **[Discovery](RFC%20OARP%20v1.0-RC.md#phase-1-discovery--auth):** Capability negotiation via `DeviceManifest`.
*   **[Tasking](RFC%20OARP%20v1.0-RC.md#phase-2-tasking):** Sending semantic `TaskRequests` with priority and safety limits.
*   **[Telemetry](RFC%20OARP%20v1.0-RC.md#33-telemetryupdate-zustand):** Asynchronous state updates (Battery, Pose, Sensors).
*   **[Streaming](RFC%20OARP%20v1.0-RC.md#45-real-time-streaming-webrtc-signalling):** Establishing P2P video/audio channels via WebRTC.

## ⚡ Technical Standards

Implementers must strictly adhere to these primitives:

| Component | Standard / Requirement |
| :--- | :--- |
| **Ontology** | `oarp:cap` (Move, Sense, Manipulate) |
| **Data Format** | JSON-LD |
| **Safety** | **UserProof** (OAEP Sig) for critical tasks |
| **Streaming** | **WebRTC** (Signalling via OATP) |
| **Transport** | OATP Container (Encrypted) |

### Example: Task Request
```json
{
  "type": "TaskRequest",
  "priority": "NORMAL",
  "goal": {
    "action": "oarp:cap:Move",
    "target": { "room": "Kitchen", "pose": [1.0, 2.0, 1.57] }
  },
  "safetyConstraints": {
    "maxSpeed": "0.5",
    "keepDistance": "0.2"
  },
  "userProof": { "..." } // Mandatory for critical ops
}
```

## 🛠 Reference Implementations

*   **Core Logic (Rust):** [`oap-foundation/oarp-core-rs`](https://github.com/oap-foundation/oarp-core-rs)
    *   *Includes Ontology Types, Safety Validators, and WebRTC Signalling helpers.*
*   **Hardware Bridge:** `oarp-ros2-bridge` (C++ Node for ROS 2)
*   **Controller SDK:** `oarp-controller-py` (Python SDK for AI Agents)

## 🧪 Conformance & Validation

All messages must be validated against the JSON Schemas defined in **[Section 7](RFC%20OARP%20v1.0-RC.md#7-json-schema-definitionen-normativ)** of the RFC.

**Critical Safety Warning:**
> Implementations MUST enforce the **Access Control List (ACL)** and **UserProof** requirements. A robot accepting unsigned critical commands is non-compliant and dangerous. The `EmergencyStop` signal must be handled with <50ms latency.

## 🤝 Contributing

We are currently in **Code Freeze**.
*   **Accepted:** Fixes for ROS 2 type mapping, Safety Constraint logic loopholes, or WebRTC SDP negotiation bugs.
*   **Not Accepted:** New hardware capabilities (e.g., Underwater Drone specifics) are deferred to v1.1.

Please review `CONTRIBUTING.md` before submitting PRs.

## 📄 License & Legal

### Specification License (Copyleft)
The specification text, architecture definitions, and protocol logic contained in this repository are licensed under the **Creative Commons Attribution-ShareAlike 4.0 International License (CC BY-SA 4.0)**.

[![License: CC BY-SA 4.0](https://img.shields.io/badge/License-CC%20BY--SA%204.0-lightgrey.svg)](https://creativecommons.org/licenses/by-sa/4.0/)

**Intent of this License:**
The goal of using CC BY-SA 4.0 is to permanently protect the open nature of this standard.
*   **ShareAlike:** If you modify, extend, or build upon this specification (e.g., creating a "Layer 2.5"), you **must** distribute your contributions under the same **CC BY-SA 4.0** license.
*   **No Proprietary Forks:** It is legally prohibited to create a proprietary, closed-source version or extension of this specification text. All derivatives must remain free and open to the community.

### Note on Implementation
To facilitate broad adoption, the use of the concepts, data structures (JSON-LD), and logic defined in this specification to create **software implementations** (libraries, applications, agents) is permitted without triggering the ShareAlike clause for the software itself.

However, any changes to the **specification document itself** remain subject to the ShareAlike requirement.

---
**Maintained by the OAP Foundation**
*Bridging the gap between bits and atoms.*
