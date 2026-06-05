# CAPABILITY-13: Consensus Authentication Protocol

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0) [![Version](https://img.shields.io/badge/Version-0.3.0--draft-orange.svg)]() [![Status](https://img.shields.io/badge/Status-RFC%20Draft-yellow.svg)]() [![Org](https://img.shields.io/badge/Org-CommonIntents--144-darkgray.svg)](https://github.com/CommonIntents)

**Version**: 0.3.0-draft
**Status**: Working Group Internal Draft
**Date**: 2026-05-29
**License**: Apache 2.0

---

## 1. Core Positioning

CAPABILITY-13 (Capability Authentication Protocol) is the **capability authentication and HITL decision standard**.

It defines "who can do what, under what conditions, within what timeframe." It transforms permissions from static passes into a dynamic, time-bound, evaluable trust process.

CAPABILITY-13 is the **consensus anchor** of the protocol stack — what you see is what you sign.

---

## 2. Relationship with INTENT-7

INTENT-7 defines "what AI wants to do"; CAPABILITY-13 defines "what AI can do, under what conditions, within what timeframe."

INTENT-7 and CAPABILITY-13 are connected through the **Manifest** — the Manifest declares which INTENT-7 intents an application supports and the security constraints for each intent. INTENT-7 intents are the syntactic source of the `name` field in the `actions` array of the Manifest.

INTENT-7-SECURE provides identity proof at the transport layer (mTLS). CAPABILITY-13 performs dynamic authorization based on this already-proven identity.

---

## 3. Minimal Core

CAPABILITY-13 Core defines only three things, irreducible.

### 3.1 Capability Declaration (Manifest)

Every CAPABILITY-13-compatible application MUST publish a capability manifest, declaring the sovereign boundary of the tool.

```json
{
  "agent_name": "anaphase",
  "version": "1.0.0",
  "actions": [
    {
      "id": "scrape",
      "label": "Scrape URL",
      "security_class": "normal",
      "parameters": { "url": { "type": "string" } }
    },
    {
      "id": "delete_file",
      "label": "Delete File",
      "security_class": "critical",
      "lease_ms": 30000,
      "parameters": { "path": { "type": "string" } }
    }
  ],
  "layout_hints": {
    "preferred_panels": ["state_tree", "text_panel"],
    "grid": {
      "rows": [
        { "id": "sidebar", "constraint": { "percentage": 0.3 } },
        { "id": "main", "constraint": { "percentage": 0.7 } },
        { "id": "bottom", "constraint": { "fixed_lines": 3 } }
      ]
    }
  }
}
```

**Manifest Field Description**:

| Field | Type | Required | Description |
|------|------|------|------|
| `agent_name` | string | Yes | Human-readable name of the Agent |
| `version` | string | Yes | Semantic version number of the Agent |
| `actions` | array of Action | Yes | List of invocable actions |
| `layout_hints` | LayoutHints | No | Default layout suggestions |

**Action Field Description**:

| Field | Type | Required | Description |
|------|------|------|------|
| `id` | string | Yes | Unique action identifier (maps to INTENT-7 intent name) |
| `label` | string | Yes | Human-readable button label |
| `security_class` | string | Yes | `"normal"` or `"critical"` |
| `lease_ms` | u64 | No | Time-to-live (milliseconds) of HITL approval window. Declaring this field enables the lease extension. |
| `parameters` | object | Yes | JSON Schema defining action parameter structure |

**LayoutHints Field Description**:

| Field | Type | Required | Description |
|------|------|------|------|
| `preferred_panels` | array of string | No | List of recommended node types to display |
| `grid` | GridDefinition | No | Explicit grid layout definition |

**Layout Priority (Full Override Strategy)**:

Clients **MUST** select layout sources in the following priority order:
1. `SemanticSnapshot.layout_overrides` (Highest priority, can change dynamically per snapshot)
2. `CapabilityManifest.layout_hints` (Default layout at connection establishment)
3. Client built-in implicit heuristics (Fallback when above two are absent)

Higher-priority sources **completely override** lower-priority ones; no partial field merging is performed. This eliminates implicit conflicts caused by JSON merging and guarantees predictable client behavior.

**Not declared means it does not exist. Capabilities not declared are not authorized for any Agent.**

### 3.1.1 Semantic Snapshot — Agent State Projection

`SemanticSnapshot` is a complete projection of the Agent's current state, pushed to clients via BIND-19 `snapshot/update` events. It is the **only data source** for client UI rendering.

#### SemanticSnapshot Structure
| Field | Type | Required | Description |
|------|------|------|------|
| `epoch_time` | u64 | Yes | Snapshot timestamp (Unix seconds) |
| `status` | string | Yes | Agent status: `"running"`, `"idle"`, `"error"` and others |
| `metrics` | object | No | Optional metrics data |
| `semantic_tree` | array of SemanticNode | Yes | List of UI nodes |
| `active_focus` | string | No | Suggested current focused node ID by Agent |
| `layout_overrides` | LayoutHints | No | Dynamic layout override (highest priority) |

#### SemanticNode Structure
| Field | Type | Required | Description |
|------|------|------|------|
| `id` | string | Yes | Unique node identifier |
| `node_type` | string | Yes | Node type (see enumeration below) |
| `label` | string | Yes | Human-readable label |
| `content` | object | Yes | Node content data, structure determined by `node_type` |
| `slot_binding` | string | No | Bind to a specific layout slot ID |
| `focused` | boolean | Yes | Agent suggested focus state |

#### NodeType Enumeration
| Value | Description |
|----|------|
| `"state_tree"` | Hierarchical state tree |
| `"text_panel"` | Text / Markdown panel |
| `"action_button"` | Triggerable action button |
| `"progress_bar"` | Progress indicator |
| `"code_diff"` | Code difference view |
| `"metrics"` | Numeric metrics panel |

When a client encounters an unknown `node_type`, it **MUST** fall back to a debug view and render raw `content` data, and **MUST NOT** crash.

### 3.2 Decision Request and State Machine

All operations requiring human approval enter an asynchronous queue through a standard Action Request.

**Action Request Standard Format:**
```json
{
  "action_id": "delete_file",
  "parameters": { "path": "/tmp/test.txt" },
  "view_hash": "abc123def456...",
  "lease_ms": 30000
}
```

| Field | Type | Required | Description |
|------|------|------|------|
| `action_id` | string | Yes | Matches action `id` defined in Manifest |
| `parameters` | object | No | Action parameters, structure defined by Manifest JSON Schema |
| `view_hash` | string | Yes | View hash (SHA-256 hex string) of the interface seen during user confirmation |
| `lease_ms` | u64 | No | When `lease_ms` is declared in Manifest, clients **SHOULD** carry this field to synchronize local countdown timer with Agent |

**HITL Decision State Machine:**
```
[PENDING] → (Agent processing) → [APPROVED / DENIED / EXPIRED]
```

- **PENDING**: Waiting for human confirmation (only applies to `security_class: "critical"` actions).
- **APPROVED**: User confirmed the action with valid `view_hash`.
- **DENIED**: User rejected the action.
- **EXPIRED**: Lease window timed out, action revoked.

**Lease Time Contract**:
If `lease_ms` is declared in Manifest, clients **MUST** render a visible countdown indicator on UI. Once the countdown reaches zero, the client locally discards the intent. The Agent **MUST** also mark the action as `EXPIRED`. Both sides rely on absolute lease time for state alignment, independent of network clock synchronization.

### 3.2.1 View Consensus Anchor — View Hash

#### Definition
`view_hash` is a cryptographic fingerprint of the content visible to a user when signing a decision. It anchors to the **semantic layout tree**, not physical pixel coordinates. This guarantees identical hash output for the same `SemanticSnapshot` across different screen sizes and frontend implementations (TUI / WebUI).

#### Algorithm: Semantic Tuple Serialization
**Key Design Decision**: The hash input uses a **flat array with fixed index order** instead of JSON key-value map. This fundamentally eliminates inconsistent key ordering caused by different programming languages, JSON libraries and hash table implementations.

#### Calculation Steps
1. Collect all visible `SemanticNode` items.
2. Sort nodes in lexicographical ascending order by `node.id`.
3. Construct a deterministic array for each node:
   ```
   [id, node_type, label, content_hash, slot_binding, focused]
   ```
   - `content_hash`: Hex string of `SHA-256(canonical_json(node.content))`
   - `slot_binding`: Set to `null` if not present
   - `focused`: Boolean value `true` / `false`
4. Serialize the list of arrays into a compact JSON string (no extra whitespace):
   ```
   [["id1","text_panel","Hello","abc123...",null,false],["id2","action_button","Go","def456...","bottom",true]]
   ```
5. Compute `SHA-256` over the compact string to get the final `ViewHash`.

#### Cross-Platform Consistency
Since `view_hash` excludes physical coordinates (x, y, width, height), the same `SemanticSnapshot` produces identical `view_hash` on TUI and WebUI. This is the core technical guarantee for the One-UI goal.

#### Transmission
The `ActionRequest` **MUST** carry the `view_hash` field, representing the view state when the user signed the decision. The Agent uses this value to verify that the user signed against the correct interface state.

### 3.3 Capability Handshake

For operations with declared time-bound constraints, the Agent MUST complete a handshake before execution — using its long-term identity key (proven by INTENT-7-SECURE mTLS) to obtain a short-lived operational credential (JWT), completing the operation within its validity window.

**One long-term identity proof, in exchange for one time-bound operational authorization.**

---

## 4. Optional Extensions

All advanced features exist as independent extension packages. **Activated on demand, silent by default.**

| Extension | Activation Condition | Function |
|:---|:---|:---|
| Lease Extension | `lease_ms` field declared in Manifest | Capability time-bound constraints and renewal |
| Expiry Extension | `lease_ms` carried in Action Request | Lifespan of decision approval window |
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

The `signature` field is optional. When the Manifest declares `approval.method`, this field MUST be present, and the CAPABILITY-13 server MUST verify the signature's validity before any state change.

**CAPABILITY-13 defines only the standard format of the signature, not which hardware or algorithm generates the signature.**

---

## 6. Protocol Boundaries

CAPABILITY-13 **is responsible for**:
- Defining the syntax of capability declarations
- Defining the format of decision requests and responses
- Defining the HITL state machine lifecycle
- Defining the activation mechanism for optional extensions
- Defining the standard format for approval signatures
- Defining SemanticSnapshot, SemanticNode and view hash specification

CAPABILITY-13 **is not responsible for**:
- The specific implementation of the decision queue (queue infrastructure belongs to existing middleware)
- The choice of hardware and algorithm for signatures (belongs to the implementation layer)
- Business permission judgment logic (belongs to the application layer)
- Identity verification (provided by INTENT-7-SECURE mTLS)

---

## 7. Relationship with INTENT-7-SECURE

INTENT-7-SECURE completes cryptographic identity proof within the first millisecond of connection establishment. CAPABILITY-13 performs dynamic authorization and capability handshakes based on this already-proven identity.

INTENT-7-SECURE provides a **long-term identity key** (the Agent's private key certificate). CAPABILITY-13 issues a **short-lived operational credential** (JWT) on top of it. Long-term identity is not directly used for operational authorization; operational authorization MUST pass through a time-bound credential.

**Platform Independence**: The CAPABILITY-13 protocol specification itself is published via content addressing (CID), independent of any specific platform. The decision queue can be implemented using any compatible message queue infrastructure (e.g., SQLite, Redis, Kafka); the protocol does not mandate any specific technology stack.

---

## 8. Future Direction

CAPS (Capability Authentication Protocol for Swarms) will, on the foundation of CAPABILITY-13, introduce Decentralized Identifiers (DID), Verifiable Credentials (VC), Zero-Knowledge Proofs, and a global reputation system, supporting autonomous collaboration among Agents in a network.

---

*This white paper is maintained by the INTENT-7/CAPABILITY-13 Protocol Working Group.*
