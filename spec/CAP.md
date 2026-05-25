# CAP: Consensus Authentication Protocol

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0) [![Version](https://img.shields.io/badge/Version-0.1.0--draft-orange.svg)]() [![Status](https://img.shields.io/badge/Status-RFC%20Draft-yellow.svg)]() [![Org](https://img.shields.io/badge/Org-CommonIntents-darkgray.svg)](https://github.com/CommonIntents)

**Version**: 0.1.0-draft
**Status**: Working Group Internal Draft
**Date**: 2026-05-22
**License**: Apache 2.0

---

## 1. Core Positioning

CAP (Capability Authentication Protocol) is the **capability authentication and HITL decision standard**.

It defines "who can do what, under what conditions, within what timeframe." It transforms permissions from static passes into a dynamic, time-bound, evaluable trust process.

CAP is the **consensus anchor** of the protocol stack — what you see is what you sign.

---

## 2. Relationship with CIS

CIS defines "what AI wants to do"; CAP defines "what AI can do, under what conditions, within what timeframe."

CIS and CAP are connected through the **Manifest** — the Manifest declares which CIS intents an application supports and the security constraints for each intent. CIS intents are the syntactic source of the `name` field in the `actions` array of the Manifest.

CISS provides identity proof at the transport layer (mTLS). CAP performs dynamic authorization based on this already-proven identity.

---

## 3. Minimal Core

CAP Core defines only three things, irreducible.

### 3.1 Capability Declaration (Manifest)

Every CAP-compatible application MUST publish a capability manifest, declaring the sovereign boundary of the tool.

```json
{
  "application": "cellrix-payment",
  "actions": [
    {
      "name": "transfer",
      "description": "Initiate a funds transfer",
      "securityClass": "critical",
      "lease": {
        "maxDuration": "15m",
        "renewable": false
      }
    },
    {
      "name": "balance",
      "description": "Query account balance",
      "securityClass": "safe"
    }
  ]
}
```

The `name` field declared in the Manifest corresponds to CIS intent syntax. `securityClass` determines whether this action requires HITL approval. When the `lease` field exists, the Lease Extension is activated; when absent, the operation involves no time-bound constraints.

**Not declared means it does not exist. Capabilities not declared are not authorized for any Agent.**

### 3.2 Decision Request and State Machine

All operations requiring human approval enter an asynchronous queue through a standard Decision Request.

```
pending → notified → viewed → approved / rejected / modified → completed
                                 ↘ expired (if decision expiry is set)
```

**Decision Request Standard Format:**

```json
{
  "decision_id": "unique-identifier",
  "action": "transfer",
  "parameters": {},
  "agent_id": "identity-of-requester",
  "created_at": "timestamp",
  "expires_at": "decision-expiry-time (optional)",
  "context": {
    "request_id": "idempotency-guarantee",
    "state_hash": "hash-of-agent-execution-state",
    "timestamp": "request-timestamp"
  }
}
```

The server MUST verify state consistency before execution. If inconsistent, it MUST return a `state_mismatch` error.

### 3.3 Capability Handshake

For operations with declared time-bound constraints, the Agent MUST complete a handshake before execution — using its long-term identity key (proven by CISS mTLS) to obtain a short-lived operational credential (JWT), completing the operation within its validity window.

**One long-term identity proof, in exchange for one time-bound operational authorization.**

---

## 4. Optional Extensions

All advanced features exist as independent extension packages. **Activated on demand, silent by default.**

| Extension | Activation Condition | Function |
|:---|:---|:---|
| Lease Extension | `lease` field declared in Manifest | Capability time-bound constraints and renewal |
| Expiry Extension | `expires_at` carried in Decision Request | Lifespan of the decision itself |
| Audit Extension | Enabled in Manifest or server config | Complete event audit trail |
| Passkey Extension | `approval.method: "passkey"` declared in Manifest | Approval result requires hardware signature |
| Delegation Extension | Defined in future CAPS phase | Capability decay and transfer |

**Extensions not declared are not activated. Not activated means zero overhead.**

---

## 5. Asynchronous HITL

### 5.1 Model

After submitting a decision request for approval, the Agent **continues executing other tasks** — its logic flow is not torn. The human asynchronously reviews the queue at their convenience. When the decision result returns, the Agent seamlessly resumes.

The human is not a system bottleneck. The human is the strategist holding ultimate veto power.

### 5.2 Approval Signature Standardization

Standard format for approval results:

```json
{
  "decision_id": "original-decision-id",
  "status": "approved",
  "approver": "approver-identity",
  "timestamp": "approval-time",
  "signature": {
    "payload": "decision_id + status + timestamp",
    "algorithm": "signature-algorithm-identifier",
    "value": "signature-value"
  }
}
```

The `signature` field is optional. When the Manifest declares `approval.method`, this field MUST be present, and the CAP server MUST verify the signature's validity before any state change.

**CAP defines only the standard format of the signature, not which hardware or algorithm generates the signature.**

---

## 6. Protocol Boundaries

CAP **is responsible for**:
- Defining the syntax of capability declarations
- Defining the format of decision requests and responses
- Defining the HITL state machine lifecycle
- Defining the activation mechanism for optional extensions
- Defining the standard format for approval signatures

CAP **is not responsible for**:
- The specific implementation of the decision queue (queue infrastructure belongs to existing middleware)
- The choice of hardware and algorithm for signatures (belongs to the implementation layer)
- Business permission judgment logic (belongs to the application layer)
- Identity verification (provided by CISS mTLS)

---

## 7. Relationship with CISS

CISS completes cryptographic identity proof within the first millisecond of connection establishment. CAP performs dynamic authorization and capability handshakes based on this already-proven identity.

CISS provides a **long-term identity key** (the Agent's private key certificate). CAP issues a **short-lived operational credential** (JWT) on top of it. Long-term identity is not directly used for operational authorization; operational authorization MUST pass through a time-bound credential.

**Platform Independence**: The CAP protocol specification itself is published via content addressing (CID), independent of any specific platform. The decision queue can be implemented using any compatible message queue infrastructure (e.g., SQLite, Redis, Kafka); the protocol does not mandate any specific technology stack.

---

## 8. Future Direction

CAPS (Capability Authentication Protocol for Swarms) will, on the foundation of CAP, introduce Decentralized Identifiers (DID), Verifiable Credentials (VC), Zero-Knowledge Proofs, and a global reputation system, supporting autonomous collaboration among Agents in a network.

---

*This white paper is maintained by the CIS/CAP Protocol Working Group.*
