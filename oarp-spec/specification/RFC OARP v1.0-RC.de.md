# RFC: Open Agent Robotics Protocol (OARP)
**Version:** 1.0 (PROPOSED STANDARD)
**Status:** CODE FREEZE
**Date:** 2025-12-03
**Layer:** 2 (Application)
**Dependencies:** OATP v1.0, OAEP v1.0, OAPP v1.0

## 1. Einleitung

Das **Open Agent Robotics Protocol (OARP)** ist der Standard für die Interaktion zwischen autonomen Software-Agenten (Personal AIs) und physischen Aktuatoren (Robotern, Drohnen, Smart Home).

Es fungiert als **semantischer High-Level-Layer**. Während Frameworks wie ROS (Robot Operating System) die internen Regelkreise steuern, definiert OARP die **Absicht (Intent)** und die **Sicherheitsgrenzen (Constraints)** für die Ausführung in einer herstellerunabhängigen Sprache.

### 1.1 Zielsetzung und Reifegrad
Diese Version 1.0 definiert einen produktionsreifen Standard, der folgende Kernanforderungen erfüllt:
1.  **Safety First:** Physische Sicherheit wird durch ein 3-Schichten-Modell (Hardware, Firmware, Protokoll) und kryptografische User-Proofs gewährleistet.
2.  **Interoperabilität:** Eine normative Ontologie (`oarp:cap`) ermöglicht es Agenten, Fähigkeiten von unbekannten Geräten zu verstehen.
3.  **Real-Time Capable:** Integration von WebRTC-Signalling über OATP für Video-Feeds und Teleoperation.

---

## 2. Architektur & Ontologie

### 2.1 Rollenmodell
*   **Controller Agent:** Der Auftraggeber (z.B. Personal AI). Er plant Tasks und überwacht die Ausführung.
*   **Physical Agent:** Der Ausführende (Roboter/Device). Er übersetzt semantische Befehle in Motor-Aktionen und garantiert die Einhaltung von Sicherheitslimits.

### 2.2 Kontext & Versionierung
Damit "Fahr in die Küche" funktioniert, müssen sich Controller und Physical Agent auf ein gemeinsames Verständnis der Umgebung einigen.
*   **Capability Negotiation:** Der Physical Agent deklariert seine Protokoll-Version im `DeviceManifest`. Controller MÜSSEN die Kompatibilität vor dem Senden von Tasks prüfen.

### 2.3 Normative Ontologie (Capabilities)
Um Interoperabilität zu gewährleisten, MÜSSEN Physical Agents ihre Fähigkeiten mittels standardisierter URIs deklarieren.

**Basis-Vokabular (Namespace `oarp:cap:`):**

| Capability | Beschreibung | Parameter-Beispiele |
| :--- | :--- | :--- |
| `can:Locate` | Selbstlokalisierung | `accuracy` (GPS, SLAM) |
| `can:Move` | Fortbewegung | `mobility` (WHEELED, LEGGED, FLYING) |
| `can:Manipulate` | Greifen/Interagieren | `payload` (kg), `dof` (Freiheitsgrade) |
| `can:Sense` | Umgebungswahrnehmung | `modalities` (CAMERA, LIDAR, THERMAL) |
| `can:Audio` | Sprachausgabe/-aufnahme | `languages` |
| `can:Compute` | On-Edge Inference | `models` (YOLO, LLAMA) |

---

## 3. Datenmodell (JSON-LD)

Nachrichten werden als `payload` in OATP-Containern transportiert.

### 3.1 `DeviceManifest` (Die Visitenkarte)
Der Roboter beschreibt sich selbst und seine Fähigkeiten.

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

### 3.2 `TaskRequest` (Der Befehl)
Eine semantische Aufgabe mit Priorität und Sicherheitsgrenzen.

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

  // Optional: Menschliche Bestätigung für kritische Tasks
  "userProof": { "..." } 
}
```

### 3.3 `TelemetryUpdate` (Zustand)
Periodische oder Event-basierte Statusmeldung.

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

## 4. Protokoll-Ablauf

### Phase 1: Discovery & Auth
1.  **Pairing:** Controller und Roboter tauschen DIDs aus (OAEP).
2.  **ACL Check:** Roboter prüft, ob Controller-DID berechtigt ist (siehe Kap. 6).

### Phase 2: Tasking
1.  **Request:** Controller sendet `TaskRequest`.
2.  **Validation:** Roboter prüft Machbarkeit, Sicherheit und Version.
3.  **Ack:** Roboter sendet `TaskAccepted` oder `OARPError`.

### Phase 3: Execution & Stream
1.  **Telemetry:** Roboter sendet Updates via OATP.
2.  **Real-Time (Optional):** Für Video-Feeds oder Teleoperation wird ein WebRTC-Stream ausgehandelt.

### Phase 4: Termination
1.  **Result:** Roboter sendet `TaskResult` (Erfolg/Fehlschlag).
2.  **Stop:** Controller kann jederzeit `TaskCancel` senden.

### 4.5 Real-Time Streaming (WebRTC Signalling)
OATP dient als **Signalling Channel**.

**Nachricht: `StreamSetup`**
```json
{
  "type": "StreamSetup",
  "sdp": "v=0...", // Session Description Protocol
  "iceCandidates": [ ... ],
  "streamType": "VIDEO_H264"
}
```
*   **Security:** Der WebRTC-Kanal MUSS via DTLS gesichert sein. Schlüssel können aus der OAEP-Session abgeleitet werden.

---

## 5. Fehlerbehandlung (Error Handling)

OARP definiert normative Fehlercodes.

| Code | Bedeutung | Client-Aktion |
| :--- | :--- | :--- |
| **`OARP_UNSAFE_ACTION`** | Task verletzt Sicherheitsregeln. | Constraints verschärfen oder Task abbrechen. |
| **`OARP_CAPABILITY_MISSING`** | Hardware unterstützt Aktion nicht. | Alternativen Plan generieren. |
| **`OARP_RESOURCE_LOW`** | Batterie/Speicher kritisch. | Roboter aufladen lassen. |
| **`OARP_OBSTACLE_DETECTED`** | Weg blockiert (während Execution). | Warten oder neuen Pfad planen. |
| **`OARP_EMERGENCY_STOPPED`** | Not-Halt wurde ausgelöst. | **SOFORT** manuelle Intervention erforderlich. |
| **`OARP_AUTH_FAILED`** | DID nicht in ACL oder UserProof fehlt. | Authentifizierung erneuern. |

---

## 6. Sicherheits-Implikationen (Physical Safety)

### 6.1 Access Control List (ACL)
Jeder Physical Agent MUSS eine persistente ACL führen.
*   **Struktur:** Mapping von `DID` -> `Role` (`ADMIN`, `OPERATOR`, `VIEWER`).
*   **Vorschrift:** Kritische Aktionen (Bewegung, Manipulation) erfordern mindestens `OPERATOR`.

### 6.2 UserProof (Mensch-in-der-Schleife)
Für Aktionen mit hohem Risiko (z.B. "Tür entriegeln") DARF der Roboter einen reinen KI-Befehl nicht akzeptieren.
*   **Anforderung:** Der `TaskRequest` muss ein `userProof`-Objekt enthalten (OAEP-Signatur über den Task-Hash).
*   **Validierung:** Der Roboter verifiziert die Signatur gegen die DID des menschlichen Besitzers.

### 6.3 Brute-Force Protection
*   **Rate Limiting:** Auth-Handshake und Task-Requests MÜSSEN rate-limited sein (Token Bucket).
*   **Backoff:** Nach 3 fehlgeschlagenen Auth-Versuchen MUSS der Roboter Verbindungen temporär ablehnen.

---

## 7. JSON-Schema Definitionen (Normativ)

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
*(Struktur wie in 4.5)*

---

## 8. Anhang (Informativ)

### 8.1 ROS 2 Bridge
Architekturvorschlag für die Anbindung an ROS 2 Topics (`/cmd_vel`, `/battery_state`).

### 8.2 Performance & Latenz (Benchmarks)
Für Implementierer gelten folgende Richtwerte für die Verarbeitung ("Processing Overhead") im Roboter:
*   **Emergency Stop:** < 50ms (Soft Real-Time)
*   **Task Request (Validation):** < 500ms
*   **WebRTC Signalling:** < 200ms
*   **Telemetry Update:** < 100ms
