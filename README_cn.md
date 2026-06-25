# POSRouter / Lensing 协议规范 (V1.6)

> **POSRouter** 是对外品牌、包名空间与 API 表面。
> 底层网络协议在法律与技术层面称为 **Lensing Protocol**。

| 语言 | 本文档 |
|------|--------|
| 中文 | **本页** |
| English | [README_en.md](./README_en.md) |

---

## 版本迭代历史

| 版本 | 来源 / 文档 | 主要内容 |
|------|-------------|----------|
| **V1.1** | [spec-deeplink-v1.1.pdf](./spec-deeplink-v1.1.pdf) | GoMenu ↔ Ezypos 同机 Deep Link：`connect` / `pay` / `refund` / `pay_result` 回调 |
| **V1.4** | 联盟 Lensing 整合规范 | 三层集成路线图；JSON 载荷；Lensing Subject；Gateway `/init` HMAC；全景与路由 Mermaid 图 |
| **V1.5** | **当前（Level 1）** | Level 1 新增 **`void`**、回调 **`card_number`**；规范拆分为三层独立文件；中英双语 |
| **V1.6** | **当前（Level 2）** | Lensing Subject 固定六段命名：`lensing.{acquirer}.{merchant}.{sub\|_}.{tid}.{verb}` |

> **命名说明：** 旧 README 中的「V0.4」对应规范 **V1.4**；Level 1 void / 文档重组为 **V1.5**；Lensing 六段 Subject 为 **V1.6**。

---

## 1. 概述与核心愿景

本规范定义 Starrie 联盟 **Lensing 分布式支付编排协议** 及商业准入模型，消除消费系统与收单系统集成时的底层网络复杂性。合作方可按能力由简到难渐进接入。

### 分层集成制 (Integration Levels)

| Level | 文档（中文） | 文档（English） | 说明 | SDK |
|-------|-------------|-------------------|------|-----|
| **1** | [level-1-deeplink_cn.md](./level-1-deeplink_cn.md) | [level-1-deeplink_en.md](./level-1-deeplink_en.md) | 同机 `ezypos://` / `gomenu://pay_result`；**connect / pay / refund / void** | 不需要 |
| **2** | [level-2-lensing_cn.md](./level-2-lensing_cn.md) | [level-2-lensing_en.md](./level-2-lensing_en.md) | Gateway `/init` + **Lensing**（`.pay` / `.result` / `.claimed` / **`.void`** / **`.refund`**）+ JSON | 可选 |
| **3** | [level-3-secure_cn.md](./level-3-secure_cn.md) | [level-3-secure_en.md](./level-3-secure_en.md) | Level 2 + 客户端非对称密钥、安全信封、Participant 证书 | 强烈建议 |

**升级路径：** Level 1 → 2 不破坏已有 Deep Link URL。先上线 deeplink，有跨机或 void 可靠 ack 需求再升 Level 2；生产联盟环境再升 Level 3。

**JSON Schemas：** [`schemas/`](./schemas/)

**历史参考：** [spec-deeplink-v1.1.pdf](./spec-deeplink-v1.1.pdf)（规范 **V1.1**）

---

## 2. 核心设计哲学

* **隐藏底层复杂性** — 分布式协同、心跳、状态机锁在 SDK 内核；对外呈现极简控制流。
* **数据与控制流分离** — 本地 deeplink 仅承载轻量控制元数据；对账、小票等大 payload 走 Lensing 虚拟网络通道（Level 2+）。

---

## 3. 联盟准入（全局）

* **4 位参与者代码 (Participant Code)** — 新成员须向联盟申请专属 4 位大写字母代号（例：收单 Supay `SUPY`，消费 GoMenu `GPOS`）。

---

## 4. 全景架构图（各层共用）

以下图表描述 **三层整体拓扑**，不绑定单一 Level 的实现细节；各层命令、参数与时序见对应 Level 文档。

### 4.1 Lensing 全景运行用例图

```mermaid
graph TD
    subgraph Master_Zone ["主控控制端拓扑"]
        A["主控消费端 App<br>(GoMenu POS)"] -->|"1. 极简本地调用<br>(Deep Link / SDK.pay)"| B["Lensing 内核组件<br>(智能路由决策引擎)"]
    end

    subgraph Scenario_1 ["场景一：同机部署 (Level 1)"]
        B -->|"本地轨道: 同机包名<br>(Intent / Scheme)"| C["本地一体化智能终端<br>(商米 P3 Mix)"]
        subgraph Local_OS ["Android 沙箱"]
            C --> D["联盟收单 App<br>(Ezypos)"]
            C --> E["本地通道拓展<br>(Skyzer 等)"]
        end
    end

    subgraph Scenario_2 ["场景二：跨设备集群 (Level 2+)"]
        B -->|"网络轨道: Lensing 广播"| F["Lensing 分布式虚拟网络"]
        F -->|"lensing.SUPY.merchant._.TID.pay"| G["边缘终端 A<br>(远程商米 P3)"]
        subgraph Remote_Sunmi ["分体商米"]
            G --> H["Kiosk / 伴侣服务"]
            G --> I["收单内核 (Ezypos)"]
        end
        F -->|"跨网络穿透"| J["边缘终端 B<br>(Stripe S700 等)"]
        subgraph Remote_Stripe ["分体 Stripe"]
            J --> K["Stripe Terminal"]
        end
    end

    style Master_Zone fill:#f9f9f9,stroke:#333,stroke-width:2px
    style Scenario_1 fill:#e3f2fd,stroke:#1565c0,stroke-width:1px
    style Scenario_2 fill:#f5f5f5,stroke:#37474f,stroke-width:1px
```

### 4.2 智能路由决策流程

```mermaid
graph TD
    Start(["POS 调用 pay"]) --> Check{"云端 Matrix：<br>本机是否存在收单包?"}

    Check -- "是 · 同机" --> LocalTrack["Level 1 本地轨道"]
    LocalTrack --> OS{"操作系统"}
    OS -- "Android" --> ExplicitIntent["显式 Intent + setPackage"]
    OS -- "iOS" --> AppScheme["独占 URL Scheme"]
    ExplicitIntent --> LaunchLocal([拉起本地收单 UI])
    AppScheme --> LaunchLocal

    Check -- "否 · 跨机" --> NetTrack["Level 2+ 网络轨道"]
    NetTrack --> Envelope["JSON 载荷 · Lensing 广播"]
    Envelope --> Broadcast["lensing.{acquirer}.{merchant}.{sub|_}.{TID}.pay"]
    Broadcast --> LaunchRemote([远端终端亮屏刷卡])
```

> **Android SDK：** 默认 **`auto`**（本机乐观拉起 → 失败走 Lensing）。应用可通过 `RoutePreference` 字符串覆盖（如平板固定 **`remote_first`**）。详见 [Level 2 §1.1](./level-2-lensing_cn.md#11-android-sdk-路由偏好route-preference) 与 [sdk-android README](../sdk-android/README.md#route-preference)。

---

## 5. 阅读指引

| 你的目标 | 从这里开始 |
|----------|------------|
| 最快接入、同机 POS + Ezypos | [Level 1 中文](./level-1-deeplink_cn.md) |
| 跨设备、Kiosk、void ack | [Level 2 中文](./level-2-lensing_cn.md) |
| 生产联盟、非对称鉴权 | [Level 3 中文](./level-3-secure_cn.md) |
