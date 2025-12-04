# OAP Layer 2: Application & Semantics

[![Layer](https://img.shields.io/badge/OAP-Layer%202-blueviolet)](https://oap.foundation)
[![Status](https://img.shields.io/badge/status-CODE%20FREEZE-snowflake)](./)
[![License](https://img.shields.io/badge/license-CC%20BY%204.0-green)](LICENSE)

> **The Language of the Agent Economy.**
>
> This repository contains the normative specifications for **Layer 2** of the Open Agent Protocol (OAP) framework. While Layer 0 provides trust and Layer 1 handles transport, **Layer 2 defines the "Business Logic"**. It specifies how agents negotiate contracts, transfer value, publish content, and control hardware.

## 📚 The Protocol Suite

Layer 2 is modular. Agents only need to implement the protocols relevant to their function (e.g., a "Merchant Bot" implements OACP and OAPP, while a "Social Bot" implements OAFP).

### Core Protocols (The Backbone)

| Acronym | Protocol Name | Focus | Spec Link |
| :--- | :--- | :--- | :--- |
| **OAPP** | **Payment Protocol** | 💳 Settlement, PSD2, Escrow | [Read RFC](./RFC%20OAPP%20v1.0-RC.md) |
| **OADP** | **Data Protocol** | 💾 Sovereign Data Exchange, ODRL | [Read RFC](./RFC%20OADP%20v1.0-RC.md) |

### Domain-Specific Protocols

| Acronym | Protocol Name | Focus | Spec Link |
| :--- | :--- | :--- | :--- |
| **OACP** | **Commerce Protocol** | 🛍️ Intent-Centric Trade, Negotiation | [Read RFC](./RFC%20OACP%20v1.0-RC.md) |
| **OAFP** | **Feed Protocol** | 📰 Decentralized Social Media | [Read RFC](./RFC%20OAFP%20v1.0-RC.md) |
| **OACoP** | **Collaboration Protocol** | 🤝 Scheduling, Task Delegation | [Read RFC](./RFC%20OACoP%20v1.0-RC.md) |
| **OAHP** | **Health Protocol** | 🏥 Patient-Held Records, Emergency Access | [Read RFC](./RFC%20OAHP%20v1.0-RC.md) |
| **OAVP** | **Voting Protocol** | 🗳️ Anonymous, Verifiable Elections | [Read RFC](./RFC%20OAVP%20v1.0-RC.md) |
| **OARP** | **Robotics Protocol** | 🤖 Physical Actuation, Safety Layers | [Read RFC](./RFC%20OARP%20v1.0-RC.md) |

## ⚡ Shared Technical Standards

To ensure semantic interoperability between different domains (e.g., paying for a health record), all Layer 2 protocols adhere to these common standards:

### 1. JSON-LD & Schema.org
All payloads are **JSON-LD** documents. We strictly prefer existing vocabularies from **Schema.org** (e.g., `schema:Product`, `schema:MedicalRecord`) over inventing new ones. This allows generic AI models to "understand" the data without custom training.

### 2. Dependency Injection
Layer 2 protocols are designed to work together.
*   **Commerce (OACP)** triggers **Payment (OAPP)** for settlement.
*   **Social (OAFP)** uses **Payment (OAPP)** for "Proof of Burn" (Anti-Spam).
*   **Health (OAHP)** uses **Data (OADP)** for encryption and access policies.

### 3. State Machines
Since the OAP network is asynchronous (Layer 1), all L2 protocols are defined as **State Machines** (e.g., `REQUESTED` -> `AUTHORIZED` -> `SETTLED`). Agents must maintain local state to handle message delays or network partitions robustly.

## 🏗 Relation to Other Layers

Layer 2 generates the *Meaning*, which is then secured by Layer 0 and shipped by Layer 1.

```mermaid
graph TD
    subgraph Layer 2: Application
    OACP[Commerce] -.->|Triggers| OAPP[Payment]
    OAFP[Social] -.->|Refers to| OADP[Data Blobs]
    OAHP[Health]
    end
    
    L2_Output[JSON-LD Payload] --- OACP & OAPP & OAFP & OAHP & OADP
    
    L2_Output -->|Encrypted & Sharded| L1[Layer 1: Transport (OATP)]
    L1 -->|Signed| L0[Layer 0: Trust (OAEP)]
```

## 🛠 Implementation

The OAP Foundation provides reference implementations for the core logic of these protocols.

*   **Reference Core:** [`oap-foundation/layer2-core-rs`](https://github.com/oap-foundation/layer2-core-rs)
    *   *Contains shared types, state machine engines, and validation logic for all L2 protocols.*

**⚠️ Developer Note:** When implementing these specifications, pay attention to the **Normative JSON Schemas** provided in Section 7 of each RFC. Strict validation is required to prevent logical attacks.

## 🤝 Contributing

We are currently in **Code Freeze** for v1.0.
We welcome feedback regarding:
*   Semantic ambiguities in Schema.org mapping.
*   Logical deadlocks in state machines.
*   Inconsistencies between the OAPP payment flow and other protocols.

Please see `CONTRIBUTING.md` for details.

## 📄 License

*   **Specifications:** [Creative Commons Attribution 4.0 International](LICENSE)
*   **Code Samples:** MIT License

---
**Maintained by the OAP Foundation**
*Building the semantic web of agents.*