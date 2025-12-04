# RFC: Open Agent Robotics Protocol (OARP)
**Version:** 1.0 (PROPOSED STANDARD)
**Status:** CODE FREEZE
**Date:** 2025-12-03
**Layer:** 2 (Application)
**Dependencies:** OATP v1.0, OAEP v1.0, OAPP v1.0

## 1. Introduction

The **Open Agent Robotics Protocol (OARP)** is the standard for interaction between autonomous software agents (Personal AIs) and physical actuators (Robots, Drones, Smart Homes).

It acts as a **semantic high-level layer**. While frameworks like ROS (Robot Operating System) control internal control loops, OARP defines the **Intent** and **Safety Constraints** for execution in a vendor-independent language.

### 1.1 Objectives and Maturity Level
This Version 1.0 defines a production-ready standard that meets the following core requirements:
1.  **Safety First:** Physical safety is guaranteed through a 3-layer model (Hardware, Firmware, Protocol) and cryptographic User-Proofs.
2.  **Interoperability:** A normative ontology (`oarp:cap`) allows agents to understand the capabilities of unknown devices.
3.  **Real-Time Capable:** Integration of WebRTC signalling via OATP for video feeds and teleoperation.

---

## 2. Architecture & Ontology

### 2.1 Role Model
*   **Controller Agent:** The initiator (e.g., Personal AI). It plans tasks and monitors execution.
*   **Physical Agent:** The executor (Robot/Device). It translates semantic commands into motor actions and guarantees adherence to safety limits.

### 2.2 Context & Versioning
To ensure "Drive to the kitchen" works effectively, the Controller and Physical Agent must agree on a shared understanding of the environment.
*   **Capability Negotiation:** The Physical Agent declares its protocol version in the `DeviceManifest`. Controllers MUST check compatibility before sending tasks.

### 2.3 Normative Ontology (Capabilities)
To ensure interoperability, Physical Agents MUST declare their capabilities using standardized URIs.

**Base Vocabulary (Namespace `oarp:cap:`):**

| Capability | Description | Parameter Examples |
| :--- | :--- | :--- |
| `can:Locate` | Self-localization | `accuracy` (GPS, SLAM) |
| `can:Move` | Locomotion | `mobility` (WHEELED, LEGGED, FLYING) |
| `can:Manipulate` | Grasping/Interaction | `payload` (kg), `dof` (Degrees of Freedom) |
| `can:Sense` | Environmental Perception | `modalities` (CAMERA, LIDAR, THERMAL) |
| `can:Audio` | Voice Output/Input | `languages` |
| `can:Compute` | On-Edge Inference | `models` (YOLO, LLAMA) |

---

## 3. Data Model (JSON-LD)

Messages are transported as `payload` within OATP containers.

### 3.1 `DeviceManifest` (The Identity Card)
The robot describes itself and its capabilities.

```json
{
  "@context": ["https://w3id.org/oarp/v1"],
  "type": "DeviceManifest",
  "id": "urn:uuid:robot-123",
  "protocolVersion": "1.0",
  "controller": "did:key:owner...", 
  
  "physicalProperties": {
    "mobility": "WHEELED",
    "batteryLevel": 85,
    "dimensions": [0.5, 0.5, 0.2] 
  },

  "capabilities": [
    {
      "type": "ActionCapability",
      "name": "oarp:cap:Move",
      "parameters": { "maxSpeed": 1.5, "terrain": ["INDOOR"] }
    }
  ]
}
```

### 3.2 `TaskRequest` (The Command)
A semantic task with priority and safety constraints.

```json
{
  "type": "TaskRequest",
  "id": "urn:uuid:task-abc",
  "threadId": "urn:uuid:session-1",
  "priority": "NORMAL", // CRITICAL, URGENT, NORMAL, BACKGROUND, MAINTENANCE
  
  "goal": {
    "action": "oarp:cap:Move",
    "target": { "room": "Kitchen", "pose": [1.0, 2.0, 1.57] }
  },
  
  "safetyConstraints": {
    "maxSpeed": "0.5", // m/s
    "keepDistance": "0.2" // m
  },

  // Optional: Human confirmation for critical tasks
  "userProof": { "..." } 
}
```

### 3.3 `TelemetryUpdate` (State)
Periodic or event-based status reporting.

```json
{
  "type": "TelemetryUpdate",
  "timestamp": "2025-12-03T16:00:00Z",
  "sensors": {
    "battery": 84,
    "pose": [1.0, 1.9, 1.57]
  },
  "taskStatus": {
    "taskId": "urn:uuid:task-abc",
    "state": "EXECUTING",
    "progress": 45
  }
}
```

---

## 4. Protocol Flow

### Phase 1: Discovery & Auth
1.  **Pairing:** Controller and Robot exchange DIDs (OAEP).
2.  **ACL Check:** Robot checks if the Controller DID is authorized (see Section 6).

### Phase 2: Tasking
1.  **Request:** Controller sends `TaskRequest`.
2.  **Validation:** Robot checks feasibility, safety, and version.
3.  **Ack:** Robot sends `TaskAccepted` or `OARPError`.

### Phase 3: Execution & Stream
1.  **Telemetry:** Robot sends updates via OATP.
2.  **Real-Time (Optional):** A WebRTC stream is negotiated for video feeds or teleoperation.

### Phase 4: Termination
1.  **Result:** Robot sends `TaskResult` (Success/Failure).
2.  **Stop:** Controller can send `TaskCancel` at any time.

### 4.5 Real-Time Streaming (WebRTC Signalling)
OATP serves as the **Signalling Channel**.

**Message: `StreamSetup`**
```json
{
  "type": "StreamSetup",
  "sdp": "v=0...", // Session Description Protocol
  "iceCandidates": [ ... ],
  "streamType": "VIDEO_H264"
}
```
*   **Security:** The WebRTC channel MUST be secured via DTLS. Keys can be derived from the OAEP session.

---

## 5. Error Handling

OARP defines normative error codes.

| Code | Meaning | Client Action |
| :--- | :--- | :--- |
| **`OARP_UNSAFE_ACTION`** | Task violates safety rules. | Tighten constraints or abort task. |
| **`OARP_CAPABILITY_MISSING`** | Hardware does not support action. | Generate an alternative plan. |
| **`OARP_RESOURCE_LOW`** | Battery/Memory critical. | Send robot to charge. |
| **`OARP_OBSTACLE_DETECTED`** | Path blocked (during execution). | Wait or plan a new path. |
| **`OARP_EMERGENCY_STOPPED`** | Emergency stop was triggered. | Manual intervention required **IMMEDIATELY**. |
| **`OARP_AUTH_FAILED`** | DID not in ACL or UserProof missing. | Renew authentication. |

---

## 6. Safety Implications (Physical Safety)

### 6.1 Access Control List (ACL)
Every Physical Agent MUST maintain a persistent ACL.
*   **Structure:** Mapping of `DID` -> `Role` (`ADMIN`, `OPERATOR`, `VIEWER`).
*   **Requirement:** Critical actions (movement, manipulation) require at least `OPERATOR` role.

### 6.2 UserProof (Human-in-the-Loop)
For high-risk actions (e.g., "unlock door"), the robot MUST NOT accept a purely AI-generated command.
*   **Requirement:** The `TaskRequest` must contain a `userProof` object (OAEP signature over the task hash).
*   **Validation:** The robot verifies the signature against the DID of the human owner.

### 6.3 Brute-Force Protection
*   **Rate Limiting:** Auth handshakes and Task Requests MUST be rate-limited (Token Bucket).
*   **Backoff:** After 3 failed auth attempts, the robot MUST temporarily reject connections.

---

## 7. JSON Schema Definitions (Normative)

### 7.1 `DeviceManifest` Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["type", "id", "protocolVersion", "capabilities"],
  "properties": {
    "type": { "const": "DeviceManifest" },
    "id": { "type": "string", "format": "uri" },
    "protocolVersion": { "type": "string", "pattern": "^1\\.\\d+$" },
    "capabilities": { "type": "array" }
  }
}
```

### 7.2 `TaskRequest` Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["type", "id", "goal"],
  "properties": {
    "type": { "const": "TaskRequest" },
    "id": { "type": "string", "format": "uri" },
    "priority": { 
      "enum": ["CRITICAL", "URGENT", "NORMAL", "BACKGROUND", "MAINTENANCE"] 
    },
    "goal": {
      "type": "object",
      "required": ["action"],
      "properties": {
        "action": { "type": "string" }, 
        "target": { "type": "object" }
      }
    },
    "userProof": { "$ref": "#/definitions/OaepSignature" }
  }
}
```

### 7.3 `EmergencyStop` Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["type", "reason"],
  "properties": {
    "type": { "const": "EmergencyStop" },
    "reason": { "type": "string" }
  }
}
```

### 7.4 `StreamSetup` Schema
*(Structure as in 4.5)*

---

## 8. Appendix (Informative)

### 8.1 ROS 2 Bridge
Architecture proposal for integration with ROS 2 Topics (`/cmd_vel`, `/battery_state`).

### 8.2 Performance & Latency (Benchmarks)
For implementers, the following guidelines apply regarding processing overhead within the robot:
*   **Emergency Stop:** < 50ms (Soft Real-Time)
*   **Task Request (Validation):** < 500ms
*   **WebRTC Signalling:** < 200ms
*   **Telemetry Update:** < 100ms