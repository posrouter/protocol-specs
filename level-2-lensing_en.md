# Level 2 — Lensing Wire (JSON / NATS) (V1.6)

| Language | Document |
|----------|----------|
| 中文 | [level-2-lensing_cn.md](./level-2-lensing_cn.md) |
| English | **this page** |

> **Audience:** Partners needing **cross-device** routing, kiosk listeners, `attemptId` dedupe, and reliable **void ack** over NATS. SDK optional but recommended.
>
> **Prerequisite:** [Level 1](./level-1-deeplink_en.md) deep links remain valid for same-device fallback.

**Framework:** [README_en.md](./README_en.md) · **Level 3:** [level-3-secure_en.md](./level-3-secure_en.md)

---

## 1. Scope

Level 2 adds:

- Gateway `/init` HMAC handshake → NATS credentials
- JSON `PaymentRequest` / `PaymentResult` on NATS subjects
- Subjects: `.pay`, `.result`, `.claimed`, **`.void`**
- SDK state machine and reconnect queue

Level 1 same-device URLs are unchanged; SDK may use local track when acquirer package is present.

---

## 2. NATS subject routing (V1.6)

### 2.1 Fixed six-segment namespace (Scheme A)

```text
lensing.{acquirerCode}.{merchantId}.{subMerchantId|_}.{terminalId}.{verb}
```

| Segment | Description |
|---------|-------------|
| `acquirerCode` | Acquirer / participant code (e.g. `SUPY`), uppercase |
| `merchantId` | Merchant id; aligns with Level 1 `merchantid` / SDK `merchantId` |
| `subMerchantId` | Sub-merchant; **`_` when absent** (reserved; real ids must not use `_`) |
| `terminalId` | Terminal id within that merchant namespace |
| `verb` | `pay` \| `result` \| `void` \| **refund** \| `claimed` |

**Examples:**

```text
lensing.SUPY.abc123._.TID001.pay
lensing.SUPY.abc123.REST01.TID001.result
lensing.SUPY.abc123._.TID001.void
lensing.SUPY.abc123._.TID001.refund
```

**Monitor subscribe:** `lensing.SUPY.abc123._.TID001.>` or `lensing.SUPY.abc123.>`

Matrix registration key: `(acquirerCode, merchantId, subMerchantId|_, terminalId)`.

### 2.2 Direction and verbs

| Direction | `{verb}` | Purpose |
|-----------|----------|---------|
| POS → Terminal | `pay` | Payment request |
| POS → Terminal | `void` | Initiator void |
| POS → Terminal | **`refund`** | **Refund against original pay `orderId`** |
| Terminal → POS | `result` | Pay/refund result / void ack |
| Terminal ↔ Terminal | `claimed` | UI preemption (SDK internal) |

**Level 1 mapping:** same-device void uses `ezypos://void?…` — [Level 1 §5.3](./level-1-deeplink_en.md#53-void--ezyposvoid-v15).

> **Deprecated:** V1.4 `lensing.terminal.{TID}.{verb}` (no acquirer/merchant isolation). Greenfield deployments use V1.6 only.

---

## 3. PaymentRequest JSON

> Level 1 same-device fields: [Level 1 §5](./level-1-deeplink_en.md#5-commands-pos--acquirer).

Schema: [`schemas/payment-request.json`](./schemas/payment-request.json)

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

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `acquirerCode` | string | ✓ | Acquirer code (subject segment 1) |
| `merchantId` | string | ✓ | Merchant id (subject segment 2) |
| `subMerchantId` | string | ✗ | Sub-merchant (segment 3; `_` when omitted) |
| `terminalId` | string | ✓ | Target terminal (segment 4) |
| `orderId` | string | ✓ | POS order id (Level 1 `orderid`) |
| `attemptId` | string | ✓ | Dedupe key; default `{orderId}#1` |
| `amount` | integer | ✓ | Minor units (cents) |
| `currency` | string | ✓ | ISO 4217 |
| `targetPackageName` | string | Android | Local-track package |
| `targetScheme` | string | iOS | Local-track scheme |
| `metadata` | object | ✗ | Extensions |

---

## 5. RefundRequest JSON (V1.6)

> Level 1 same-device: `ezypos://refund?amount=…&orderid=…` — [Level 1 §5.4](./level-1-deeplink_en.md#54-refund--ezyposrefund).

Schema: [`schemas/refund-request.json`](./schemas/refund-request.json)

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

| Field | Notes |
|-------|-------|
| `orderId` | **Original pay** order id |
| `attemptId` | Default `{orderId}#refund` (distinct from pay `#1`) |
| Result | Still on **`.result`**; `metadata.operation=refund` or Level 1 `type=REFUND` callback |

**Cross-device flow:** POS `PUBLISH …refund` → terminal launches `ezypos://refund` → terminal `PUBLISH …result` → POS callback.

---

## 6. PaymentResult JSON

> Level 1 callback query: [Level 1 §6](./level-1-deeplink_en.md#6-reverse-callback-acquirer--pos) (`card_number` on deeplink success).

Schema: [`schemas/payment-result.json`](./schemas/payment-result.json)

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

| `status` | Meaning |
|----------|---------|
| `approved` | Success |
| `declined` | Declined |
| `cancelled` | User cancel or initiator void — see `metadata.cancelReason` |
| `error` | System error |

| `metadata` key | When | Level 1 equivalent |
|----------------|------|-------------------|
| `cancelReason` | `cancelled` | `cancel_reason` query param |
| `cardLast4` | `approved` card pay | `card_number` query param |

| `operation` | Refund | callback `type=REFUND` |

---

## 7. Void wire flow

```mermaid
sequenceDiagram
    participant A as Initiator POS (A)
    participant NATS as NATS
    participant B as Terminal / Kiosk (B)

    A->>NATS: PUBLISH lensing.SUPY.abc123._.TID001.void
    Note over B: Soft void · optional ezypos://void for NFC
    B->>NATS: PUBLISH lensing.SUPY.abc123._.TID001.result<br>status=cancelled, cancelReason=initiator_void
    NATS-->>A: result delivered to subscriber
```

| `cancelReason` | Source |
|----------------|--------|
| `user_cancel` | User cancelled in acquirer UI |
| `initiator_void` | Initiator called void (deeplink or NATS) |

---

## 8. Gateway authentication (V1.4 symmetric HMAC — current)

> **Roadmap:** Symmetric HMAC until Level 3 asymmetric model ships. Skip intermediate V1 symmetric-key Postgres hash path. Target: [Level 3](./level-3-secure_en.md).

```
GET https://lensing.starrie.org/init?code=GPOS
Headers:
  X-PR-Timestamp: <unix_ms>
  X-PR-Signature: HmacSHA256(key + timestamp)
```

Success:

```json
{
  "nats_url": "nats://router.starrie.org:4222",
  "nats_token": "TOKEN_GPOS_<SECRET_SALT>"
}
```

| Item | Behaviour |
|------|-----------|
| SDK holds | Symmetric `key` (Keychain / Keystore) |
| Redis | `client:key:{CODE}` = plaintext `key` |
| Transport | Never send `key`; only timestamp + signature |
| Signature | `HMAC-SHA256(key, key + timestamp)` → hex |

---

## 7. Lensing kernel state machine

| State | Description |
|-------|-------------|
| `IDLE` | Not initialized |
| `DISCOVERING` | Fetching NATS credentials from Gateway |
| `CONNECTING` | Opening NATS connection |
| `CONNECTED` | Ready to publish / subscribe |
| `RECONNECTING` | Backoff reconnect |
| `FAILED` | Unrecoverable error |

Outbound messages queue locally on disconnect; flush after reconnect.

---

## 8. Gateway runtime architecture

Alliance Gateway on Vercel: **Edge** for `/init`, **Node Serverless** for Portal / Admin.

### 8.1 Environment comparison

| | Edge Function | Node.js Serverless |
|--|---------------|-------------------|
| Location | Global CDN edge | Regional (e.g. `iad1`) |
| Cold start | Very fast | Slower |
| Typical routes | `/init` | `/admin/*`, `/portal/*` |
| Capabilities | `fetch`, Web Crypto, Upstash REST | Prisma, PostgreSQL, PDF certs |

### 8.2 Request topology

```mermaid
flowchart TB
    subgraph Clients
        SDK["POSRouter SDK"]
        Portal["Portal UI"]
        Admin["Admin UI"]
    end

    subgraph EdgeNet ["Vercel Edge"]
        CDN["TLS · routing"]
    end

    subgraph EdgeFn ["Edge /init"]
        Init["HMAC auth · NATS token"]
    end

    subgraph NodeFn ["Node Serverless"]
        PortalAPI["/portal/*"]
        AdminAPI["/admin/*"]
    end

    subgraph Data
        Redis["Upstash Redis"]
        PG["PostgreSQL"]
        NATS["NATS Cluster"]
    end

    SDK --> CDN --> Init
    Portal --> CDN --> PortalAPI
    Admin --> CDN --> AdminAPI
    Init --> Redis
    PortalAPI --> PG
    PortalAPI --> Redis
    SDK --> NATS
```

### 8.3 Route map

| URL | Runtime | Role |
|-----|---------|------|
| `GET /init` | `edge` | HMAC auth, issue NATS token |
| `/portal/*` | `nodejs22.x` | Partner self-service, key rotation |
| `/admin/*` | `nodejs22.x` | Alliance admin, participant activation |

---

## 9. Sequence: SDK init → NATS

```mermaid
sequenceDiagram
    autonumber
    participant App as POS App
    participant SDK as POSRouter SDK
    participant Edge as Edge /init
    participant Redis as Upstash Redis
    participant NATS as NATS

    App->>SDK: initialize("GPOS", participantKey)
    SDK->>Edge: GET /init?code=GPOS<br>X-PR-Timestamp, X-PR-Signature
    Edge->>Redis: GET client:key:GPOS
    Edge-->>SDK: { nats_url, nats_token }
    SDK->>NATS: CONNECT
    Note over SDK: Subscribe lensing.{acquirer}.{merchant}.{sub|_}.{TID}.result

    App->>SDK: pay(request)
    SDK->>NATS: PUBLISH lensing.SUPY.abc123._.TID001.pay
    NATS-->>SDK: lensing.SUPY.abc123._.TID001.result
    SDK-->>App: PaymentResult
```

---

## 10. Sequence: symmetric key rotation (Portal)

```mermaid
sequenceDiagram
    autonumber
    participant User as Partner admin
    participant Portal as Portal
    participant Node as Node Serverless
    participant PG as PostgreSQL
    participant Redis as Redis
    participant Edge as Edge /init

    User->>Portal: Rotate key
    Portal->>Node: POST /portal/api/keys/rotate
    Node->>PG: Revoke old · insert new
    Node->>Redis: SET client:key:GPOS = newKey
    Portal-->>User: Show newKey once

    Note over User,Edge: Old key → 401 on /init
    User->>Edge: SDK with newKey → 200
```

### 10.1 Admin: activate participant

```mermaid
sequenceDiagram
    participant Admin as Alliance admin
    participant Node as Node Serverless
    participant PG as PostgreSQL
    participant Redis as Redis

    Admin->>Node: Create participant SUPY
    Node->>PG: INSERT organization + key record
    Node->>Redis: SET client:key:SUPY = initialKey
    Node-->>Admin: One-time key delivery
```

### 10.2 Storage split (operational target)

| Data | PostgreSQL | Redis |
|------|------------|-------|
| Orgs / users / roles | ✓ | — |
| Key rotation audit | ✓ | active key only |
| `/init` hot path (V1.4 HMAC) | — | `client:key:{CODE}` |
| `/init` hot path (Level 3) | pubkey archive | `client:pubkey:{CODE}` |

> Full asymmetric storage model: [Level 3 §4](./level-3-secure_en.md).

---

## 11. Level 2 limitations

| Capability | Level 2 | Level 3 |
|------------|---------|---------|
| Non-repudiation | ✗ | Ed25519 signed `/init` |
| Private key never leaves device | ✗ (symmetric key) | ✓ |
| Participant certificate | ✗ | ✓ |

---

## 12. Document history

| Version | Change |
|---------|--------|
| V1.4 | NATS, JSON schemas, Gateway HMAC, state machine, Portal rotation |
| V1.5 | **`.void`** subject; `metadata.cancelReason`; `cardLast4`; split Level 2 doc |
| V1.6 | **Fixed 6-segment subjects**; `merchantId` / `acquirerCode` in JSON; `_` sub-merchant placeholder |
