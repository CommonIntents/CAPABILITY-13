# CAP — Capability Authentication Protocol

[![Org](https://img.shields.io/badge/Org-CommonIntents-darkgray.svg)](https://github.com/CommonIntents)

**Dynamic defense, activated on demand.**

CAP defines *who* can do *what*, under *what conditions*, within *what timeframe*. It transforms permissions from static passes into a time-bound, evaluable trust process.

CAP is the **immune system** of the CIS/CAP protocol family.

## What CAP Defines

- **Manifest** — capability declarations (what a tool exposes, with security constraints)
- **Decision Queue** — asynchronous HITL (human-in-the-loop) state machine
- **Capability Handshake** — long-term identity → short-lived operational credential
- **Optional Extensions** — Lease (time-bound), Expiry, Audit, Passkey, Delegation

## Core Philosophy

> *Not declared, means it does not exist.*
> *Extensions not activated mean zero overhead.*

## Protocol Stack

```
CIS  (intent semantics)
 ↑
CIB  (transport binding)
 ↑
CISS (mTLS security)
 ↑
CAP  ← You are here
```

## Read the Spec

- [CAP v0.1.0-draft](spec/CAP.md)
- [中文版](spec/CAP.zh-CN.md)

## Related

| Protocol | Repository |
|----------|------------|
| CIS | [CommonIntents/CIS](https://github.com/CommonIntents/CIS) |
| CIB | [CommonIntents/CIB](https://github.com/CommonIntents/CIB) |
| CISS | [CommonIntents/CISS](https://github.com/CommonIntents/CISS) |

## License

Apache 2.0 — see [LICENSE](LICENSE).
