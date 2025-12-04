# RFC: Open Agent Voting Protocol (OAVP)
**Version:** 1.0 (PROPOSED STANDARD)
**Status:** CODE FREEZE
**Date:** 2025-12-03
**Layer:** 2 (Application)
**Dependencies:** OATP v1.0, OAEP v1.0, OADP v1.0

## 1. Introduction

The **Open Agent Voting Protocol (OAVP)** defines the standard for secure, anonymous, and universally verifiable elections in the digital space. It resolves the fundamental paradox of digital elections—the simultaneous requirement for identification (eligibility) and anonymity (vote secrecy)—through the use of modern cryptography.

### 1.1 Objectives and Maturity
This Version 1.0 defines a production-ready standard for democratic processes in agent networks:
1.  **Perfect Unlinkability:** Through **Blind Signatures**, it is mathematically impossible to link a cast vote to a specific voter.
2.  **Coercion Resistance:** Mechanisms such as "Panic Credentials" protect voters from coercion and vote buying.
3.  **Universal Verifiability:** The integrity of the tally is verifiable by anyone through public ledgers (OADP).

### 1.2 Integration in the OAP Stack
*   **Identity:** Eligibility is verified via OAEP-DIDs and Verifiable Credentials.
*   **Transport:** Vote casting occurs via an **OATP Mixnet** (anonymization network) to obfuscate IP metadata.
*   **Storage:** The "Digital Ballot Box" is an immutable log in OADP format.

---

## 2. Architecture & Roles

OAVP implements a strict cryptographic separation of powers.

### 2.1 The Authorities
*   **Registration Authority (RA):** The "Electoral Roll". Knows the identity (DID), checks eligibility, and issues `VotingTokens`. It never sees the vote.
*   **Validation Authority (VA):** The "Stamp". Signs blinded ballots. It sees neither the identity (only the anonymous token) nor the vote (since it is blinded).
*   **Tallying Authority (TA):** The "Election Board". Counts the votes in the Digital Ballot Box (DBB).

### 2.2 The Voter (Voter Agent)
The Personal AI that abstracts the complex cryptographic process (blinding, mixnet routing) for the user.

---

## 3. Cryptography (Normative)

To guarantee interoperability and security, implementations MUST use the following primitives.

### 3.1 Blind Signatures: BLS12-381
OAVP mandates **Boneh-Lynn-Shacham (BLS)** signatures on the **BLS12-381** curve.
*   **Properties:** Short signatures (48 Bytes), deterministic, aggregatable.
*   **Libraries:** Implementations SHOULD use established libraries such as `blst` (IETF Draft Standard).

### 3.2 Mixnet Encryption
For onion routing within the OATP network (Phase 4), nested **OATP-JWE encryption** is used, where each hop can only decrypt the payload for the immediate next hop.

---

## 4. Protocol Flow (The Voting Process)

### Phase 1: Registration (Setup)
1.  **Announcement:** Publication of the `ElectionManifest`.
2.  **Registration:** The Voter Agent authenticates itself to the RA (via OAEP).
    *   *Normal Case:* RA issues a valid `VotingToken`.
    *   *Panic Case:* If the user authenticates with a defined "Panic Password" (coercion scenario), the RA issues a **Fake Token** (see Sec. 6.1).

### Phase 2: Preparation & Blinding (Local)
1.  **Selection:** The user makes a choice (e.g., "Candidate A").
2.  **Formatting:** The Agent creates the `Ballot` payload including a random nonce (for individual verifiability).
3.  **Blinding:** The Agent applies the BLS blinding factor $r$ -> `BlindedBallot`.

### Phase 3: Authorization (Signing)
1.  **Request:** Voter sends `BlindedBallotRequest` to the VA.
2.  **Check:** VA cryptographically verifies the `VotingToken`.
    *   *Double-Vote Check:* The VA invalidates the token to prevent multiple usage.
3.  **Sign:** VA signs the BlindedBallot with its private BLS key and sends `BlindedBallotResponse`.

### Phase 4: Casting (The Mixnet)
This is the most critical step for anonymity.
1.  **Unblind:** The Voter removes the factor $r$ and obtains the valid signature $\sigma$ for their ballot.
2.  **Mixnet Routing:** The Agent sends the `CastBallot` via a chain of OATP Relays (minimum 3 hops).
    *   **Jitter:** Each Relay MUST insert a random delay to hinder timing analyses.
3.  **Drop:** The final hop drops the ballot into the public DBB.
4.  **Receipt:** The DBB acknowledges receipt (via OATP back through the Mixnet or as a PubSub broadcast).

### Phase 5: Verification & Counting (Tallying)
1.  **Validation:** The DBB accepts only messages with a valid BLS signature from the VA.
2.  **Filtering:** After the election closes, the RA (or TA) publishes the list of "Fake Token Serial Numbers" (or uses Zero-Knowledge Tags) to filter out panic votes without revealing the identities of coerced voters.
3.  **Count:** Counting of valid votes.

---

## 5. Error Handling

Normative error codes for OAVP:

| Code | Meaning | Client Action |
| :--- | :--- | :--- |
| **`OAVP_INVALID_TOKEN`** | The RA's VotingToken is invalid/expired. | Re-register with RA. |
| **`OAVP_ALREADY_VOTED`** | Token has already been used. | Abort (Security Warning). |
| **`OAVP_BLINDING_ERROR`** | Blinding format does not match BLS12-381. | Check crypto library. |
| **`OAVP_ELECTION_CLOSED`** | Time window expired. | Voting no longer possible. |
| **`OAVP_SIGNATURE_REJECTED`** | The DBB rejects the VA signature. | Critical Alarm (VA key corrupt?). |

---

## 6. Security Implications

### 6.1 Coercion Resistance
OAVP protects voters who are forced to reveal their vote.

1.  **Panic Credentials:**
    *   If the voter uses their "Panic Secret" at the RA, they receive a token that is signed by the VA and accepted by the DBB.
    *   To the attacker, the process ("Success") and the receipt ("Vote Accepted") look identical to a real vote.
    *   However, during the tallying phase, this vote is eliminated as invalid.

2.  **Re-Voting (Optional Policy):**
    *   Elections CAN be configured so that a voter can have their token signed multiple times. Only the last vote (identified by timestamp or sequence) counts. This allows coerced voters to overwrite their "fake" vote later.

### 6.2 Mixnet Security
*   **Regulation:** A direct submission (`POST` to DBB) without a Mixnet is FORBIDDEN (`OAVP_INSECURE_TRANSPORT`), as the IP address would de-anonymize the voter.
*   **Topology:** Clients MUST choose a route over at least 3 independent OATP Relays ("Stratified Mixing").

---

## 7. JSON Schema Definitions (Normative)

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

### 7.3 `CastBallot` (The Casting)

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["type", "electionId", "selection", "signature", "nonce"],
  "properties": {
    "type": { "const": "CastBallot" },
    "electionId": { "type": "string", "format": "uri" },
    "selection": { "type": "string" }, // The selected candidate
    "signature": { "type": "string", "description": "Unblinded BLS Signature" },
    "nonce": { "type": "string", "description": "Random nonce for individual verification" }
  }
}
```

---

## 8. Appendix (Informative)

### 8.1 Reference Implementation
Repo: `oavp-core-rs` (Rust). Uses the `blst` crate for high-performance BLS12-381 operations.

### 8.2 Performance & Latency Budgets
To ensure scalability, the following guidelines apply to implementations:
*   **Blinding/Unblinding (Client):** < 100ms
*   **Signing (VA):** < 50ms per vote (horizontally scalable)
*   **Mixnet Latency (E2E):** < 60 seconds (due to jitter)
*   **Tallying:** O(N) – approx. 10,000 votes/second per Core during signature verification.

### 8.3 Formal Verification
The OAP Foundation aims to formally prove the unlinkability properties of the protocol using **Tamarin Prover** or **ProVerif**. The model will be published in the `oavp-verification` repository.