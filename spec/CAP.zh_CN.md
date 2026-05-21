# CAP 协议白皮书

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0) [![Version](https://img.shields.io/badge/Version-0.1.0--draft-orange.svg)]() [![Status](https://img.shields.io/badge/Status-RFC%20Draft-yellow.svg)]() [![Org](https://img.shields.io/badge/Org-CommonIntents-darkgray.svg)](https://github.com/CommonIntents)


## 能力认证协议

**版本**: v0.1.0 草案
**日期**: 2026-05-21
**状态**: Jasonmilk
**许可证**：Apache 2.0

---

## 一、核心定位

CAP（Capability Authentication Protocol）是**能力认证与HITL决策标准**。

它定义“谁能做什么，以什么条件，在什么时限内”。将权限从静态通行证转变为动态、有时效、可评估的信任过程。

CAP是协议栈的**免疫系统**——动态防御，按需激活。

---

## 二、与CIS的关系

CIS定义“AI想做什么”，CAP定义“AI能做什么，以什么条件，在什么时限内”。

CIS与CAP通过**Manifest**连接——Manifest声明了应用支持哪些CIS意图，以及每个意图的安全约束。CIS意图是Manifest中`actions`数组的`name`字段的语义来源。

CISS在传输层提供身份证明（mTLS），CAP基于此已证明身份进行动态授权。

---

## 三、极简核心

CAP Core只定义三样，不可再少。

### 3.1 能力声明

每个兼容CAP的应用必须发布一份能力清单，声明工具的主权边界。

```json
{
  "application": "cellrix-payment",
  "actions": [
    {
      "name": "transfer",
      "description": "发起一笔转账",
      "securityClass": "critical",
      "lease": {
        "maxDuration": "15m",
        "renewable": false
      }
    },
    {
      "name": "balance",
      "description": "查询账户余额",
      "securityClass": "safe"
    }
  ]
}
```

Manifest中声明的`name`字段对应CIS意图语义。`securityClass`决定该动作是否需要HITL审批。`lease`字段存在时，时效性扩展激活；不存在时，该操作不涉及时效性约束。

**不声明，即不存在。不声明的能力，Agent无权调用。**

### 3.2 决策请求与状态机

所有需人类审批的操作，通过标准决策请求进入异步队列。

```
pending → notified → viewed → approved / rejected / modified → completed
                                 ↘ expired (若设置了决策有效期)
```

**决策请求标准格式：**

```json
{
  "decision_id": "唯一标识",
  "action": "transfer",
  "parameters": {},
  "agent_id": "请求方身份",
  "created_at": "时间戳",
  "expires_at": "决策有效期（可选）",
  "context": {
    "request_id": "幂等性保证",
    "state_hash": "Agent执行状态哈希",
    "timestamp": "请求时间戳"
  }
}
```

服务端在执行前必须验证状态一致性，不一致则返回`state_mismatch`错误。

### 3.3 能力握手

对于声明了时效性约束的操作，Agent必须在执行前完成握手——用CISS的mTLS证明其长期身份，换取短生命周期操作凭据（JWT），在有效窗口内完成操作。

**一次长期身份证明，换取一次有时限的操作授权。**

---

## 四、可选扩展

所有高级特性以独立扩展包形式存在。**按需激活，默认静默。**

| 扩展 | 激活条件 | 功能 |
|:---|:---|:---|
| Lease 扩展 | Manifest中声明`lease`字段 | 能力时效性约束与续期 |
| Expiry 扩展 | 决策请求中携带`expires_at` | 决策本身的生存时间 |
| Audit 扩展 | Manifest或服务端配置启用 | 完整的事件审计追踪 |
| Passkey 扩展 | Manifest中声明`approval.method: "passkey"` | 审批结果需硬件签名 |
| Delegation 扩展 | 未来CAPS阶段定义 | 能力的衰减与转移 |

**扩展不声明则不激活。不激活则零开销。**

---

## 五、异步HITL

### 5.1 模型

Agent提交待审批决策后**继续执行其他任务**，逻辑流不撕裂。人类在方便时异步审查队列。决策结果返回后Agent无缝衔接。

人类不是系统的瓶颈，是握有最终否决权的战略家。

### 5.2 审批签名标准化

审批结果的标准格式：

```json
{
  "decision_id": "原始决策ID",
  "status": "approved",
  "approver": "审批者身份",
  "timestamp": "审批时间",
  "signature": {
    "payload": "decision_id + status + timestamp",
    "algorithm": "签名算法标识",
    "value": "签名值"
  }
}
```

`signature`字段为可选。当Manifest声明`approval.method`时，此字段必须存在，CAP服务端在状态变更前必须验证签名有效性。

**CAP只规定签名的标准格式，不规定用什么硬件、什么算法生成签名。**

---

## 六、协议边界

CAP **负责**：
- 定义能力声明的语法
- 定义决策请求与响应格式
- 定义HITL状态机生命周期
- 定义可选扩展的激活机制
- 定义审批签名的标准格式

CAP **不负责**：
- 决策队列的具体实现（队列管道属于基建）
- 签名的硬件和算法选择（属于实现层）
- 业务权限判断逻辑（属于应用层）
- 身份验证（由CISS的mTLS提供）

---

## 七、与CISS的关系

CISS在连接建立的第一毫秒完成密码学身份证明。CAP基于此已证明身份，进行动态授权和能力握手。

CISS提供的是**长期身份密钥**（Agent的私钥证书），CAP在此之上签发**短生命周期操作凭据**（JWT）。长期身份不直接用于操作授权，操作授权必须通过有时效的凭据。

---

## 八、未来方向

CAPS（Capability Authentication Protocol for Swarms）将在CAP基础上引入去中心化身份（DID）、可验证凭证（VC）、零知识证明、全局信誉系统，支持Agent在网络中自主协作。

---

*本白皮书由CIS/CAP协议工作组维护。*
