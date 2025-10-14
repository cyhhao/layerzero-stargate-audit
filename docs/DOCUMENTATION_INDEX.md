# LayerZero & Stargate 官方文档归档索引

本文档提供 LayerZero V2 和 Stargate V2 官方文档的完整索引和关键内容摘要。

**文档快照时间**: 2025-10-14
**文档版本**: LayerZero V2, Stargate V2

---

## LayerZero V2 官方文档

### 主文档站点
- 🔗 **官方文档**: https://docs.layerzero.network/v2
- 🔗 **GitHub**: https://github.com/LayerZero-Labs/LayerZero-v2
- 🔗 **官网**: https://layerzero.network/

### 核心文档章节

#### 1. 协议概览 (Protocol Overview)
- 🔗 https://docs.layerzero.network/v2/home/v2-overview
- **核心概念**: Omnichain互操作协议
- **支持网络**: 120+ 区块链
- **关键特性**:
  - 不可变协议核心
  - 可配置边缘合约
  - 去中心化消息验证
  - 统一流动性标准

#### 2. 协议架构 (Protocol Architecture)

**核心组件**:
```
LayerZero V2 架构:
├── EndpointV2 (不可变核心)
│   ├── MessageLibManager
│   ├── MessagingChannel
│   ├── MessagingComposer
│   └── MessagingContext
├── MessageLib (可升级)
│   ├── SendUln302
│   └── ReceiveUln302
├── DVN Network (去中心化验证网络)
│   ├── LayerZero Labs DVN
│   ├── Google Cloud DVN
│   ├── Chainlink DVN
│   ├── Nethermind DVN
│   └── 其他第三方DVN
└── Executor (消息执行者)
```

**文档链接**:
- 🔗 EndpointV2: https://docs.layerzero.network/v2/home/protocol/contract-standards
- 🔗 MessageLib: https://docs.layerzero.network/v2/home/protocol/message-library
- 🔗 DVNs: https://docs.layerzero.network/v2/home/protocol/dvn

#### 3. 安全模型 (Security Model)
- 🔗 https://docs.layerzero.network/v2/home/protocol/security

**核心安全特性**:
1. **可配置的DVN选择** - OApp自主选择验证网络
2. **M-of-N验证模型** - 灵活的多签验证
3. **不可变协议核心** - EndpointV2无法升级
4. **可升级边缘** - MessageLib可升级但有grace period

**安全配置示例**:
```solidity
UlnConfig {
    confirmations: 64,
    requiredDVNs: [LayerZero Labs, Chainlink, Nethermind],
    optionalDVNs: [Google, Polyhedra, BCW, Axelar, Switchboard],
    optionalDVNThreshold: 3
}
```

#### 4. 开发者指南 (Developer Guides)

**EVM 开发**:
- 🔗 https://docs.layerzero.network/v2/developers/evm/overview
- 🔗 OFT Quickstart: https://docs.layerzero.network/v2/developers/evm/oft/quickstart
- 🔗 OApp Quickstart: https://docs.layerzero.network/v2/developers/evm/oapp/overview

**关键合约标准**:
- OApp (Omnichain Application)
- OFT (Omnichain Fungible Token)
- ONFT (Omnichain Non-Fungible Token)

#### 5. 支持网络 (Supported Chains)
- 🔗 https://docs.layerzero.network/v2/developers/evm/technical-reference/deployed-contracts

**主要网络** (截至2025-10-14):
- Ethereum (eid: 30101)
- Arbitrum (eid: 30110)
- Optimism (eid: 30111)
- Base (eid: 30184)
- Polygon (eid: 30109)
- BSC (eid: 30102)
- Avalanche (eid: 30106)
- 以及 100+ 其他网络

#### 6. DVN 生态系统
- 🔗 https://docs.layerzero.network/v2/developers/evm/technical-reference/dvn-addresses

**主要DVN提供商**:
| DVN | 类型 | 以太坊地址 | 官网 |
|-----|------|-----------|------|
| LayerZero Labs | Push | 0x589dedbd617e0cbcb916a9223f4d1300c294236b | LayerZero官方 |
| Google Cloud | Push | 0xd56e4eab23cb81f43168f9f45211eb027b9ac7cc | (身份待确认) |
| Chainlink | Push | 0x771d10d0c86e26ea8d3b778ad4d31b30533b9cbf | Chainlink |
| Nethermind | Pull | 0xf4064220871e3b94ca6ab3b0cee8e29178bf47de | Nethermind |
| Polyhedra | Push | 0x8DDf05F9A5c488b4973897E278B58895bF87Cb24 | Polyhedra |

---

## Stargate V2 官方文档

### 主文档站点
- 🔗 **官方文档**: https://docs.stargate.finance/
- 🔗 **GitBook**: https://stargateprotocol.gitbook.io/stargate/
- 🔗 **GitHub**: https://github.com/stargate-protocol/stargate-v2
- 🔗 **官网**: https://stargate.finance/

### 核心文档章节

#### 1. 协议概览 (Protocol Overview)
- 🔗 https://docs.stargate.finance/

**核心定位**: 可组合的跨链流动性传输协议

**关键特性**:
1. **Instant Guaranteed Finality (IGF)** - 即时确定性
2. **Unified Liquidity** - 统一流动性池
3. **Native Assets** - 原生资产支持
4. **Hydra Assets** - Hydra资产(即将推出)

#### 2. Stargate V2 改进
- 🔗 https://docs.stargate.finance/primitives/routes/stargateV2

**V2主要改进**:
1. **大幅降低Gas费用** - 通过Bus机制批量传输
2. **提高资本效率** - 优化的Credit机制
3. **保持IGF** - 即时最终性保证
4. **统一流动性池** - 更好的流动性管理
5. **原生资产支持** - ETH等原生代币直接桥接

**V2架构**:
```
Stargate V2:
├── StargatePool (流动性池)
│   ├── StargatePoolNative (ETH等)
│   ├── StargatePoolUSDC
│   ├── StargatePoolUSDT
│   └── StargatePoolMETIS/mETH
├── StargateBase (核心逻辑)
│   ├── TokenMessaging (OFT集成)
│   ├── Credit机制
│   └── Bus机制
├── FeeLib (费用计算)
├── StargateStaking (质押奖励)
└── Treasurer (资金管理)
```

#### 3. Credit分配系统
- 🔗 https://stargateprotocol.gitbook.io/stargate/v2-developer-docs/integrate-with-stargate/credit-allocation-system

**Credit机制说明**:
- 每个路径维护credit余额
- 跨链转出消耗credit
- 跨链转入补充credit
- Credit耗尽时redeem失败

**代码示例**:
```solidity
struct Path {
    uint64 credit;  // 该路径可用的credit
}

function redeem(uint256 _amountLD) external {
    if (paths[localEid].credit < amountSD)
        revert Stargate_InsufficientCredits();
    paths[localEid].credit -= amountSD;
    // ...
}
```

#### 4. 费用机制
- 🔗 https://stargateprotocol.gitbook.io/stargate/user-docs/tokenomics/protocol-fees

**费用结构**:
1. **跨链转账费** - 基于路径和流动性
2. **LP费用** - 支付给流动性提供者
3. **Treasury费用** - 协议收入
4. **LayerZero费用** - 消息传递费用

#### 5. 开发者集成
- 🔗 https://stargateprotocol.gitbook.io/stargate/v2-developer-docs/integrate-with-stargate

**集成方式**:
1. **Stargate Composer** - 组合跨链调用
2. **Direct Integration** - 直接集成Stargate合约
3. **Stargate API** - API集成

#### 6. 支持资产和网络

**主要资产** (截至2025-10-14):
- ETH (Ethereum, Arbitrum, Optimism, Base等)
- USDC (多链)
- USDT (多链)
- METIS (Metis)
- mETH (Mantle Staked ETH)

**支持网络**:
- Ethereum
- Arbitrum
- Optimism
- Base
- Polygon
- BSC
- Avalanche
- Metis
- Mantle
- 以及更多...

---

## 关键技术概念对照表

| 概念 | LayerZero V2 | Stargate V2 |
|------|-------------|-------------|
| **消息传递** | Endpoint + MessageLib + DVN | 基于LayerZero构建 |
| **验证模型** | M-of-N DVN验证 | 继承LayerZero安全模型 |
| **最终性** | 等待DVN验证 | Instant Guaranteed Finality |
| **流动性** | 无需流动性(消息协议) | 统一流动性池 |
| **资金锁定** | 无资金锁定 | 流动性锁定在Pool |
| **费用** | DVN费用 + Executor费用 | LayerZero费用 + Stargate费用 + LP费用 |
| **安全性** | 依赖DVN网络 | LayerZero + Stargate合约安全 |

---

## 重要博客文章和深度解读

### LayerZero 官方博客
- 🔗 **LayerZero V2 Deep Dive**: https://medium.com/layerzero-official/layerzero-v2-deep-dive-869f93e09850
- 🔗 **Introducing LayerZero V2**: https://medium.com/layerzero-official/introducing-layerzero-v2-076a9b3cb029
- 🔗 **LayerZero V2 is Live**: https://medium.com/layerzero-official/layerzero-v2-is-live-740290f2dbe6

### Stargate 官方博客
- 🔗 **Stargate V2 公告**: https://medium.com/@Stargate.Finance/stargate-v2-28a18379269e

### 第三方分析
- 🔗 **Messari - State of LayerZero Q1 2024**: https://messari.io/report/state-of-layerzero-q1-2024
- 🔗 **Messari - State of LayerZero Q2 2024**: https://messari.io/report/state-of-layerzero-q2-2024
- 🔗 **Messari - State of LayerZero Q3 2024**: https://messari.io/report/state-of-layerzero-q3-2024
- 🔗 **Messari - State of LayerZero Q4 2024**: https://messari.io/report/state-of-layerzero-q4-2024

---

## 文档更新追踪

### LayerZero V2
| 日期 | 版本 | 主要变更 |
|------|------|---------|
| 2023-09 | V2 Beta | V2测试网发布 |
| 2024-01 | V2 Mainnet | V2主网发布 |
| 2024-06 | V2.1 | 新增多个DVN支持 |
| 2025-01 | V2.2 | (截至审计时的最新版本) |

### Stargate V2
| 日期 | 版本 | 主要变更 |
|------|------|---------|
| 2022-03 | V1 | V1发布 |
| 2024-05 | V2 | V2发布: Gas优化, Credit机制 |
| 2024-08 | V2.1 | 新增原生资产支持 |
| 2025-01 | V2.2 | (截至审计时的最新版本) |

---

## 文档本地化和翻译

### 中文文档
本仓库提供关键文档的中文翻译:

1. ✅ **主审计报告** - [LayerZero-Stargate-Security-Audit-Report.md](../reports/LayerZero-Stargate-Security-Audit-Report.md) (中文)
2. ✅ **审计摘要** - [AUDIT_SUMMARY.md](../AUDIT_SUMMARY.md) (中文)
3. 📝 **官方文档关键章节翻译** - 见 [translations/](translations/) 目录

### 文档翻译状态

| 文档 | 原文链接 | 中文翻译 | 状态 | 译者 |
|------|---------|---------|------|------|
| LayerZero V2 Overview | [链接](https://docs.layerzero.network/v2/home/v2-overview) | `translations/layerzero-v2-overview-zh.md` | 待完成 | - |
| Security Model | [链接](https://docs.layerzero.network/v2/home/protocol/security) | `translations/layerzero-security-zh.md` | 待完成 | - |
| Stargate V2 Overview | [链接](https://docs.stargate.finance/) | `translations/stargate-v2-overview-zh.md` | 待完成 | - |
| Credit System | [链接](https://stargateprotocol.gitbook.io/stargate/v2-developer-docs/integrate-with-stargate/credit-allocation-system) | `translations/stargate-credit-system-zh.md` | 待完成 | - |

---

## 社区资源

### Discord
- 🔗 LayerZero: https://discord.gg/layerzero
- 🔗 Stargate: https://discord.gg/stargate

### Twitter
- 🔗 @LayerZero_Core
- 🔗 @StargateFinance

### GitHub
- 🔗 LayerZero-Labs: https://github.com/LayerZero-Labs
- 🔗 stargate-protocol: https://github.com/stargate-protocol

### 开发者支持
- 🔗 Telegram: LayerZero Developers
- 🔗 Forum: https://stargate.discourse.group/

---

## 文档归档说明

### 为什么需要本地归档?

1. **版本控制** - 协议持续更新,归档历史版本便于对比
2. **离线访问** - 网络问题时仍可查阅
3. **审计追溯** - 审计基于特定时间点的文档
4. **中文化** - 提供中文翻译便于理解

### 如何使用本归档

1. **最新信息** - 始终优先访问官方文档获取最新信息
2. **历史对比** - 使用本归档对比协议演进
3. **审计参考** - 审计报告基于2025-10-14的快照
4. **翻译参考** - 中文翻译基于归档版本

### 更新频率

- **建议**: 每季度更新一次文档快照
- **重大更新**: 协议重大升级时立即更新
- **审计追踪**: 每次审计后更新相关文档

---

## 贡献指南

欢迎贡献文档翻译和更新:

1. Fork 本仓库
2. 创建新分支: `git checkout -b docs/translation-xxx`
3. 提交翻译: `translations/xxx-zh.md`
4. 提交PR并说明翻译内容和版本

**翻译规范**:
- 保持专业术语一致性 (参考术语表)
- 标注原文链接和翻译日期
- 代码示例保持英文
- 保留原文结构

---

## 免责声明

本文档索引基于2025-10-14的官方文档快照。协议可能已更新,请访问官方文档获取最新信息。

**最后更新**: 2025-10-14
**维护者**: Claude (AI Security Researcher)
**仓库**: https://github.com/cyhhao/layerzero-stargate-audit
