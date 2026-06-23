# Level 2 — Lensing 网络传输协议 (JSON) (V1.6)

| 语言 | 文档 |
|------|------|
| 中文 | **本页** |
| English | [level-2-lensing_en.md](./level-2-lensing_en.md) |

> **适用对象：** 需要 **跨设备** 路由、Kiosk 监听、`attemptId` 去重、以及 Lensing 网络上可靠 **void ack** 的合作方。SDK 可选，建议使用。
>
> **前置：** [Level 1](./level-1-deeplink_cn.md) Deep Link 仍可作为同机回退路径。

**总览：** [README_cn.md](./README_cn.md) · **Level 3：** [level-3-secure_cn.md](./level-3-secure_cn.md)

---

## 1. 范围

Level 2 在 Level 1 之上增加：

- Gateway `/init` HMAC 握手 → Lensing 入网凭证
- Lensing 网络上的 JSON `PaymentRequest` / `PaymentResult`
- Subject（主题）：`.pay`、`.result`、`.claimed`、**`.void`**、**`.refund`**
- SDK 状态机与断线重连队列

Level 1 同机 URL 不变；SDK 检测到本机收单包时可走本地轨道。

> **术语：** 英文 **Wire** 指 Lensing **链路上传输的报文格式**（JSON），**不是**物理有线/无线之分。中文档称 **网络传输** / **跨机（Lensing）**。底层消息总线由 SDK 封装，合作方以 **Lensing Subject + JSON** 为准即可。

---

## 2. Lensing Subject 路由 (V1.6)

### 2.1 固定六段命名（方案 A）

```text
lensing.{acquirerCode}.{merchantId}.{subMerchantId|_}.{terminalId}.{verb}
```

| 段 | 说明 |
|----|------|
| `acquirerCode` | 收单/participant code（如 `SUPY`），大写 |
| `merchantId` | 商户 id，与 Level 1 `merchantid` / SDK `merchantId` 一致 |
| `subMerchantId` | 子商户；**无则必须为 `_`**（占位，禁止真实 id 使用 `_`） |
| `terminalId` | 该商户命名空间下的终端号 |
| `verb` | `pay` \| `result` \| `void` \| **refund** \| `claimed` |

**示例：**

```text
lensing.SUPY.abc123._.TID001.pay
lensing.SUPY.abc123.REST01.TID001.result
lensing.SUPY.abc123._.TID001.void
lensing.SUPY.abc123._.TID001.refund
```

**Monitor 订阅：** `lensing.SUPY.abc123._.TID001.>` 或 `lensing.SUPY.abc123.>`

Matrix 注册主键：`(acquirerCode, merchantId, subMerchantId|_, terminalId)`。

### 2.2 方向与 verb

| 方向 | `{verb}` | 说明 |
|------|----------|------|
| POS → Terminal | `pay` | 支付请求 |
| POS → Terminal | `void` | 主叫作废 attempt |
| POS → Terminal | **`refund`** | **退款（原支付 `orderId`）** |
| Terminal → POS | `result` | 支付/退款结果 / void ack |
| Terminal ↔ Terminal | `claimed` | UI 抢占（SDK 内部） |

**Level 1 对照：** 同机 void 用 `ezypos://void?…`，见 [Level 1 §5.3](./level-1-deeplink_cn.md#53-void--ezyposvoid-v15)。

> **已废弃：** V1.4 的 `lensing.terminal.{TID}.{verb}`（未区分收单/商户，TID 跨商户碰撞）。未投产环境直接采用 V1.6。

---

## 3. PaymentRequest JSON

> Level 1 同机字段：[Level 1 §5](./level-1-deeplink_cn.md#5-命令pos--收单端)。

Schema：[`schemas/payment-request.json`](./schemas/payment-request.json)

```json
{
  "terminalId": "TID001",
  "orderId": "GM20260602001",
  "attemptId": "GM20260602001#1",
  "acquirerCode": "SUPY",
  "merchantId": "abc123",
  "subMerchantId": "REST01",
  "amount": 1250,
  "currency": "USD",
  "targetPackageName": "com.ezypos.app",
  "targetScheme": "ezypos://",
  "metadata": {}
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `merchantId` | string | ✓ | 商户 id（Subject 第 2 段） |
| `acquirerCode` | string | ✓ | 收单 code（Subject 第 1 段） |
| `subMerchantId` | string | ✗ | 子商户（Subject 第 3 段；省略则 `_`） |
| `terminalId` | string | ✓ | 目标终端（Subject 第 4 段） |
| `orderId` | string | ✓ | POS 订单号（Level 1 `orderid`） |
| `attemptId` | string | ✓ | 去重键；默认 `{orderId}#1` |
| `amount` | integer | ✓ | 最小货币单位（分） |
| `currency` | string | ✓ | ISO 4217 |
| `targetPackageName` | string | Android | 本地轨道包名 |
| `targetScheme` | string | iOS | 本地轨道 scheme |
| `metadata` | object | ✗ | 扩展 |

---

## 4. RefundRequest JSON (V1.6)

> Level 1 同机：`ezypos://refund?amount=…&orderid=…`，见 [Level 1 §5.4](./level-1-deeplink_cn.md#54-refund--ezyposrefund)。

Schema：[`schemas/refund-request.json`](./schemas/refund-request.json)

```json
{
  "terminalId": "TID001",
  "orderId": "GM20260602001",
  "attemptId": "GM20260602001#refund",
  "acquirerCode": "SUPY",
  "merchantId": "abc123",
  "amount": 1250,
  "currency": "NZD"
}
```

| 字段 | 说明 |
|------|------|
| `orderId` | **原支付**订单号 |
| `attemptId` | 默认 `{orderId}#refund`，与 pay 的 `#1` 区分 |
| 结果 | 仍走 **`.result`**；`metadata.operation=refund` 或 Level 1 回调 `type=REFUND` |

**时序（跨机）：** POS `PUBLISH …refund` → 终端拉起 `ezypos://refund` → 终端 `PUBLISH …result` → POS 收到回调。

---

## 5. PaymentResult JSON

> Level 1 回调 query：[Level 1 §6](./level-1-deeplink_cn.md#6-反向回调收单端--pos)（成功时 **`card_number`**）。

Schema：[`schemas/payment-result.json`](./schemas/payment-result.json)

```json
{
  "terminalId": "TID001",
  "orderId": "GM20260602001",
  "attemptId": "GM20260602001#1",
  "status": "approved",
  "transactionId": "txn_abc123",
  "amount": 1250,
  "currency": "USD",
  "message": "Payment approved",
  "metadata": {
    "cancelReason": "initiator_void",
    "cardLast4": "4242"
  }
}
```

| `status` | 说明 |
|----------|------|
| `approved` | 成功 |
| `declined` | 拒绝 |
| `cancelled` | 用户取消或主叫 void — 见 `metadata.cancelReason` |
| `error` | 系统错误 |

| `metadata` 键 | 时机 | Level 1 对应 |
|----------------|------|--------------|
| `cancelReason` | `cancelled` | `cancel_reason` |
| `cardLast4` | 卡支付 `approved` | `card_number` |
| **`operation`** | 退款 | 回调 `type=REFUND` |

---

## 6. Void 跨机时序（Lensing）

```mermaid
sequenceDiagram
    participant A as 主叫 POS (A)
    participant L as Lensing 网络
    participant B as 终端 / Kiosk (B)

    A->>L: PUBLISH lensing.SUPY.abc123._.TID001.void
    Note over B: Soft void · 可选 ezypos://void 停 NFC
    B->>L: PUBLISH lensing.SUPY.abc123._.TID001.result
    L-->>A: 投递 result
```

| `cancelReason` | 来源 |
|----------------|------|
| `user_cancel` | 用户在收单 UI 取消 |
| `initiator_void` | 主叫 void（deeplink 或 Lensing） |

---

## 7. Gateway 认证（V1.4 对称 HMAC — 现行）

> **路线：** 维持对称 HMAC 直至 Level 3 非对称模型落地；跳过 V1 中间态。终态见 [Level 3](./level-3-secure_cn.md)。

```
GET https://lensing.starrie.org/init?code=GPOS
Headers:
  X-PR-Timestamp: <unix_ms>
  X-PR-Signature: HmacSHA256(key + timestamp)
```

成功响应（JSON 字段名保留实现细节；语义为 **Lensing 入网地址与令牌**）：

```json
{
  "nats_url": "nats://router.starrie.org:4222",
  "nats_token": "TOKEN_GPOS_<SECRET_SALT>"
}
```

| 项目 | 行为 |
|------|------|
| SDK 持有 | 对称密钥 `key`（Keychain / Keystore） |
| Redis | `client:key:{CODE}` = 明文 `key` |
| 传输 | **不传** `key`；仅 timestamp + signature |
| 签名 | `HMAC-SHA256(key, key + timestamp)` → hex |

---

## 8. Lensing 内核状态机

| 状态 | 说明 |
|------|------|
| `IDLE` | 未初始化 |
| `DISCOVERING` | 向 Gateway 请求 Lensing 入网凭证 |
| `CONNECTING` | 连接 Lensing 网络 |
| `CONNECTED` | 可收发 |
| `RECONNECTING` | 指数退避重连 |
| `FAILED` | 不可恢复错误 |

断线时出站消息入本地队列，重连后 flush。

---

## 9. Gateway 运行时架构

> **实现说明：** 联盟 Lensing 虚拟网络当前由 NATS 集群承载；**合作方集成以 Lensing 协议为准**，经 SDK 接入，无需直接调用 NATS API。本节供联盟运营与自研 SDK 参考。

联盟 Gateway 部署于 Vercel：**Edge** 处理 `/init`，**Node Serverless** 处理 Portal / Admin。

### 9.1 执行环境

| | Edge Function | Node.js Serverless |
|--|---------------|-------------------|
| 位置 | 全球 CDN 边缘 | 区域数据中心 |
| 冷启动 | 极快 | 较慢 |
| 典型路由 | `/init` | `/admin/*`、`/portal/*` |

### 9.2 请求拓扑

```mermaid
flowchart TB
    subgraph Clients ["客户端"]
        SDK["POSRouter SDK"]
        Portal["Portal"]
        Admin["Admin"]
    end

    CDN["Vercel Edge"] --> Init["/init"]
    CDN --> PortalAPI["/portal/*"]
    CDN --> AdminAPI["/admin/*"]

    Init --> Redis["Upstash Redis"]
    PortalAPI --> PG["PostgreSQL"]
    SDK --> Net["Lensing 网络"]
```

### 9.3 路由映射

| URL | Runtime | 职责 |
|-----|---------|------|
| `GET /init` | `edge` | HMAC 鉴权、签发 Lensing 入网令牌 |
| `/portal/*` | `nodejs22.x` | 参与方自助、密钥轮换 |
| `/admin/*` | `nodejs22.x` | 联盟管理、成员激活 |

---

## 10. 时序：SDK 初始化 → Lensing 入网

```mermaid
sequenceDiagram
    autonumber
    participant App as POS App
    participant SDK as POSRouter SDK
    participant Edge as Edge /init
    participant Redis as Redis
    participant L as Lensing 网络

    App->>SDK: initialize("GPOS", key)
    SDK->>Edge: GET /init + HMAC headers
    Edge->>Redis: GET client:key:GPOS
    Edge-->>SDK: 入网地址 + 令牌
    SDK->>L: CONNECT · 订阅 result

    App->>SDK: pay()
    SDK->>L: PUBLISH .pay
    L-->>SDK: .result
```

---

## 11. 时序：对称密钥轮换

```mermaid
sequenceDiagram
    participant User as 参与方管理员
    participant Portal as Portal
    participant Node as Node Serverless
    participant PG as PostgreSQL
    participant Redis as Redis

    User->>Portal: 轮换密钥
    Portal->>Node: POST /portal/api/keys/rotate
    Node->>PG: 吊销旧钥 · 写入新钥
    Node->>Redis: SET client:key:GPOS
    Portal-->>User: 一次性展示新 key
```

### 11.1 超级管理：激活成员

```mermaid
sequenceDiagram
    participant Admin as 联盟管理员
    participant Node as Node Serverless
    participant PG as PostgreSQL
    participant Redis as Redis

    Admin->>Node: 创建参与者 SUPY
    Node->>PG: INSERT 组织 + 密钥记录
    Node->>Redis: SET client:key:SUPY
```

### 11.2 存储职责

| 数据 | PostgreSQL | Redis |
|------|------------|-------|
| 组织 / 用户 | ✓ | — |
| 密钥轮换审计 | ✓ | 当前 active key |
| `/init` 热路径（V1.4 HMAC） | — | `client:key:{CODE}` |
| `/init` 热路径（Level 3） | 公钥归档 | `client:pubkey:{CODE}` |

非对称存储详见 [Level 3 §4](./level-3-secure_cn.md)。

---

## 12. Level 2 能力边界

| 能力 | Level 2 | Level 3 |
|------|---------|---------|
| 不可抵赖 | ✗ | Ed25519 签名 `/init` |
| 私钥不出设备 | ✗（对称 key） | ✓ |
| Participant 证书 | ✗ | ✓ |

---

## 13. 文档历史

| 版本 | 变更 |
|------|------|
| V1.4 | Lensing 网络、JSON Schema、Gateway HMAC、状态机、Portal 轮换 |
| V1.5 | **`.void`**；`metadata.cancelReason`；`cardLast4`；独立 Level 2 文档 |
| V1.6 | **固定六段 Subject**；`merchantId` / `acquirerCode` 入 JSON；`subMerchantId` 占位 `_` |
