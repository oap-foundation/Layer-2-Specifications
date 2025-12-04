# RFC: Open Agent Feed Protocol (OAFP)
**Version:** 1.0 (PROPOSED STANDARD)
**Status:** CODE FREEZE
**Date:** 2025-12-03
**Layer:** 2 (Application)
**Dependencies:** OATP v1.0, OAEP v1.0, OAPP v1.0

## 1. Introduction

The **Open Agent Feed Protocol (OAFP)** is the standard for the decentralized distribution and curation of content (Social Media, News, Microblogging) within the OAP ecosystem. It replaces the centralized "feeds" of traditional platforms with an architecture based on **Algorithmic Sovereignty** and **Data Ownership**.

### 1.1 Core Principles
1.  **Decoupling:** Storage location (Where does the data reside?), Index (Who points to it?), and Curation (Who selects it?) are separated.
2.  **Local Curation ("Intention Economy"):** There is no global algorithm. The compilation of the feed is performed exclusively by the Personal AI on the user's end device, based on their explicit interests and trust networks.
3.  **Provenance:** Content is cryptographically signed (OAEP) and optionally secured against manipulation via C2PA (Content Credentials).

### 1.2 Architecture Overview
OAFP distinguishes between two spaces:
*   **Private Moments:** E2E-encrypted P2P communication via OATP (Direct Messaging, Small Groups).
*   **Public Stage:** Publication of **Manifests** in public networks (DHT, PubSub), while the **Blobs** (media) reside in Content Addressable Storage (CAS) such as IPFS.

---

## 2. Data Model (Semantics)

The central element is the **Manifest**. It describes content, context, and authorship. OAFP uses JSON-LD and Schema.org.

### 2.1 `ContentManifest` (The Post)
The "catalog card" that is distributed publicly.

```json
{
  "@context": ["https://schema.org", "https://w3id.org/oafp/v1"],
  "type": "ContentManifest",
  "id": "urn:uuid:manifest-123...",
  "author": "did:web:journalist.example.com",
  "created": "2025-12-03T10:00:00Z",
  
  // Metadata for AI curation
  "meta": {
    "type": "schema:NewsArticle", // or SocialMediaPosting
    "topics": ["Tech", "Privacy"],
    "language": "en",
    "contentWarning": "POLITICAL_DISCOURSE" 
  },

  // Reference to content (CAS)
  "content": {
    "uri": "ipfs://QmHash...",
    "digest": "SHA256:...",
    "mimeType": "text/markdown",
    "encrypted": false // true for Pay-to-View
  },

  // Economic Spam Protection (see Sec. 4)
  "proofOfBurn": {
    "txId": "oap:tx:burn_transaction_123",
    "amount": "10" // Credits
  },

  // Author Signature
  "proof": {
    "type": "OaepSignature2025",
    "signatureValue": "..."
  }
}
```

### 2.2 `Interaction` (The Reaction)
Likes, comments, and shares are independent objects that reference a manifest. They do not reside with the author, but with the interactor.

```json
{
  "type": "Interaction",
  "interactionType": "schema:LikeAction", // or CommentAction, ShareAction
  "target": "urn:uuid:manifest-123...",
  "author": "did:key:fan...",
  "proof": { ... }
}
```

### 2.3 `Tombstone` (The Deletion Signal)
To socially enforce the "Right to be Forgotten" in immutable networks.

```json
{
  "type": "Tombstone",
  "targetManifestId": "urn:uuid:manifest-123...",
  "reason": "AUTHOR_REQUEST", // or LEGAL_TAKEDOWN, SPAM
  "deletedAt": "2025-12-04T10:00:00Z",
  "author": "did:web:journalist.example.com",
  "proof": { ... }
}
```

### 2.4 `KeyDelivery` (Pay-to-View Unlock)
The object that transfers the decryption key for premium content.

```json
{
  "type": "KeyDelivery",
  "targetManifestId": "urn:uuid:manifest-123...",
  "recipient": "did:key:buyer...",
  // JWE encrypted Content Key (readable only by recipient)
  "keyPayload": "eyJhbGciOiJFQ0RILUVTI..." 
}
```

---

## 3. Protocol Flow (OATP Transport)

OAFP defines specific message types transported via OATP.

### 3.1 Subscription (`OAFP_SUBSCRIBE`)
The Personal AI signals interest in content.
*   **Action:** `SUBSCRIBE` or `UNSUBSCRIBE`.
*   **Recipient:** A PubSub Relay or directly the DID of a Creator.
*   **Payload:**
    ```json
    {
      "type": "Subscription",
      "action": "SUBSCRIBE",
      "topics": ["#Privacy"],
      "authors": ["did:web:news.com"]
    }
    ```

### 3.2 Announcement (`OAFP_ANNOUNCE`)
A Creator or Relay broadcasts a new Manifest.
*   **Payload:** The `ContentManifest` JSON.
*   **Processing:** The recipient AI validates the signature and `proofOfBurn`. If valid and relevant -> Store in local DB.

### 3.3 Fetch (`OAFP_FETCH_BLOB`)
Retrieval of the actual content (e.g., image) occurs **Out-of-Band** via IPFS/Hypercore or **In-Band** via OATP if privacy is critical.
*   **Privacy Mode:** The client requests the blob via an OATP Relay to hide its IP address from the storage provider.

### 3.4 Unlock (`OAFP_KEY_DELIVERY`)
Transport of the `KeyDelivery` message. This MUST be sent via OATP (encrypted and authenticated) to the buyer.

---

## 4. Economy: Earn & Burn (OAPP Integration)

OAFP utilizes **OAPP** to combat spam and enable monetization.

### 4.1 Proof of Burn (Anti-Spam)
To protect the "Public Stage" from bot networks, publishing manifests requires a fee.
*   **Mechanism:** The author sends an OAPP payment to a defined "Burn Address".
*   **Governance:** The minimum burn amount (`amount`) is dynamically determined by the **OAP Network Registry** (Governance Parameter).
*   **Validation:** Receiving clients verify via the OAPP ledger whether the `txId` exists.

### 4.2 Pay-to-View (Monetization)
Content can be encrypted.
1.  **Manifest:** `content.encrypted = true`. Contains `unlockCondition` (Price, Currency).
2.  **Payment:** Consumer sends OAPP `PaymentAuthorization` to the author.
3.  **Unlock:** Upon receipt of the `PaymentReceipt`, the author agent sends the `KeyDelivery` message via OATP to the buyer.

---

## 5. Moderation & Trust

### 5.1 User Control Lists (Block/Mute)
Every user maintains a local list of blocked DIDs.
*   **Effect:** Manifests and interactions from blocked DIDs are filtered out by the AI before display.
*   **Sharing:** Users can subscribe to blocklists ("Shared Blocklists"), e.g., from anti-spam initiatives.

### 5.2 Community Moderation
Communities can appoint moderators.
*   **Verification:** Moderators must be legitimized by a Verifiable Credential (`CommunityModeratorVC`) or an entry in the `OAP Trusted Registry`.
*   **Effect:** Tombstones from validated moderators are accepted by clients, even if they do not originate from the author.

### 5.3 Reputation
OAFP is a consumer of reputation data. The local AI weights content higher if the author has high reputation in the user's "Web of Trust" (e.g., "Friend of a Friend" or "Verified Journalist").

---

## 6. Security Implications

### 6.1 Reader Privacy
Retrieving content from public IPFS networks exposes the reader's IP address.
*   **Mitigation:** Clients SHOULD use anonymization networks (Tor) or **OATP Relays as Proxies** for content retrieval.
*   **Jitter:** Blob retrieval should be temporally decoupled from manifest reception (Random Delay) to complicate timing analysis.

### 6.2 C2PA Binding (Deepfake Protection)
To ensure an image is authentic:
*   The media (Blob) contains C2PA metadata.
*   The Manifest contains the OAEP signature.
*   **Check:** The client verifies if the C2PA identity matches the OAEP identity of the Manifest author.

---

## 7. JSON-Schema Definitions (Normative)

### 7.1 `ContentManifest` Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "https://w3id.org/oafp/v1/ContentManifest",
  "type": "object",
  "required": ["type", "id", "author", "created", "content", "proof"],
  "properties": {
    "type": { "const": "ContentManifest" },
    "id": { "type": "string", "format": "uri" },
    "author": { "type": "string", "format": "uri" },
    "content": {
      "type": "object",
      "required": ["uri", "digest"],
      "properties": {
        "uri": { "type": "string" },
        "digest": { "type": "string" },
        "encrypted": { "type": "boolean", "default": false }
      }
    },
    "proofOfBurn": {
      "type": "object",
      "required": ["txId", "amount"],
      "properties": {
        "txId": { "type": "string" },
        "amount": { "type": "string" }
      }
    }
  }
}
```

### 7.2 `Interaction` Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "https://w3id.org/oafp/v1/Interaction",
  "type": "object",
  "required": ["type", "interactionType", "target", "author", "proof"],
  "properties": {
    "type": { "const": "Interaction" },
    "interactionType": {
      "enum": ["schema:LikeAction", "schema:CommentAction", "schema:ShareAction"]
    },
    "target": { "type": "string", "format": "uri" },
    "author": { "type": "string", "format": "uri" },
    "proof": { "type": "object" }
  }
}
```

### 7.3 `Tombstone` Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "https://w3id.org/oafp/v1/Tombstone",
  "type": "object",
  "required": ["type", "targetManifestId", "reason", "proof"],
  "properties": {
    "type": { "const": "Tombstone" },
    "targetManifestId": { "type": "string", "format": "uri" },
    "reason": { 
      "enum": ["AUTHOR_REQUEST", "LEGAL_TAKEDOWN", "SPAM", "NSFW"] 
    }
  }
}
```

### 7.4 `KeyDelivery` Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "https://w3id.org/oafp/v1/KeyDelivery",
  "type": "object",
  "required": ["type", "targetManifestId", "recipient", "keyPayload"],
  "properties": {
    "type": { "const": "KeyDelivery" },
    "targetManifestId": { "type": "string", "format": "uri" },
    "recipient": { "type": "string", "format": "uri" },
    "keyPayload": { "type": "string", "description": "JWE Compact String" }
  }
}
```

---

## 8. Appendix (Informative)

### 8.1 Reference Implementation
Repo: `oafp-core-rs` (Rust)

### 8.2 Storage Adapters
OAFP is storage-agnostic. The reference implementation supports **IPFS** (for public content) and **Hypercore** (for mutable feeds).