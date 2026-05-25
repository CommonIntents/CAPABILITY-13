# CAP — Consensus Authentication Protocol [![Org](https://img.shields.io/badge/Org-CommonIntents-darkgray.svg)](https://github.com/CommonIntents)

**What you see is what you sign.** CAP provides a verifiable confirmation mechanism based on Cellrix's deterministic visual rendering.

CAP defines the structure of user confirmation events — carrying the visual content hash of the Cellrix interface that the user actually saw and agreed to. This bridges the semantic gap between AI intent and human authorization.

## What CAP Defines
- **Confirmation Event** — structured event carrying Cellrix view hash + user action
- **Decision Queue** — asynchronous HITL (human-in-the-loop) state machine
- **Consensus Anchor** — the Cellrix-rendered interface is the ground truth for both human and AI

## Core Philosophy
> *Not declared, means it does not exist.*
> *Extensions not activated mean zero overhead.*

## Protocol Stack
```
Cellrix (visual consensus layer)
  ↑
CAP ← You are here (consensus confirmation)
  ↑
CIS (intent syntax — SIDL)
  ↑
CIB (transport binding)
  ↑
CISS (optional mTLS transport)
```

## Read the Spec
- [CAP v0.2.0-draft](spec/CAP.md)
- [中文版](spec/CAP.zh-CN.md)

## Related
| Protocol | Repository |
|:---|:---|
| CIS | [CommonIntents/CIS](https://github.com/CommonIntents/CIS) |
| CIB | [CommonIntents/CIB](https://github.com/CommonIntents/CIB) |
| CISS | [CommonIntents/CISS](https://github.com/CommonIntents/CISS) |

## License
Apache 2.0 — see [LICENSE](LICENSE).
