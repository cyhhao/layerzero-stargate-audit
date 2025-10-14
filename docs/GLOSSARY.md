# LayerZero & Stargate 术语表 / Glossary

本文档提供 LayerZero 和 Stargate 协议中所有关键术语的中英文对照和详细解释。

**最后更新**: 2025-10-14

---

## 核心概念 / Core Concepts

### A

**OApp (Omnichain Application) / 全链应用**
- **定义**: 部署在 LayerZero 上的跨链应用程序
- **特点**: 可以在多条链上读写状态
- **示例**: 跨链DEX、跨链借贷协议
- **合约接口**: `ILayerZeroReceiver`, `OAppCore`

**Audits / 审计**
- **定义**: 第三方安全机构对智能合约代码的检查
- **类型**: 白盒审计、黑盒审计、形式化验证
- **本报告涉及**: Paladin, Zellic, Ottersec

### B

**Bus / 总线机制**
- **定义**: Stargate V2 的批量跨链传输机制
- **作用**: 将多个 Credit 传输打包成一个交易
- **优势**: 显著降低 Gas 费用
- **控制者**: Planner

**Bug Bounty / 漏洞赏金**
- **定义**: 奖励发现协议漏洞的白帽黑客
- **Stargate 奖励**: 最高 $10,000,000
- **平台**: Immunefi

### C

**Confirmations / 确认数**
- **定义**: DVN 等待的区块数量
- **作用**: 防止链重组攻击
- **推荐值**: Ethereum 64, BSC 15, Polygon 128
- **配置位置**: `UlnConfig.confirmations`

**Credit / 信用额度**
- **定义**: Stargate 路径级别的流动性平衡指标
- **作用**: 控制跨链转账方向和数量
- **风险**: Credit 耗尽时无法 redeem
- **代码**: `paths[eid].credit`

**Composer / 组合器**
- **定义**: LayerZero 的消息组合功能
- **作用**: 支持多步骤跨链操作
- **合约**: `MessagingComposer`

### D

**DVN (Decentralized Verifier Network) / 去中心化验证网络**
- **定义**: LayerZero 的链下验证节点
- **作用**: 验证跨链消息的真实性
- **工作流程**:
  1. 监听源链 PacketSent 事件
  2. 等待 confirmations 个区块
  3. 在目标链调用 verify()
- **主要 DVN**:
  - LayerZero Labs: `0x589dedbd617e0cbcb916a9223f4d1300c294236b`
  - Google Cloud: `0xd56e4eab23cb81f43168f9f45211eb027b9ac7cc`
  - Chainlink: `0x771d10d0c86e26ea8d3b778ad4d31b30533b9cbf`

**Delegate / 委托者**
- **定义**: OApp 授权的配置管理地址
- **权限**: 可修改 DVN 配置、MessageLib 等
- **风险**: Delegate 被攻破 = OApp 被攻破
- **推荐**: 使用多签合约

### E

**EndpointV2 / 端点V2**
- **定义**: LayerZero V2 的核心不可变合约
- **地址**: Ethereum `0x1a44076050125825900e736c501f859c50fE728c`
- **作用**:
  - 消息发送入口 (`send()`)
  - 消息验证入口 (`verify()`)
  - 消息执行入口 (`lzReceive()`)
- **状态**: 不可升级

**Executor / 执行器**
- **定义**: 在目标链执行跨链消息的实体
- **作用**: 调用 `EndpointV2.lzReceive()`
- **特点**: 任何人都可以作为 Executor
- **费用**: 由源链用户支付

**eid (Endpoint ID) / 端点ID**
- **定义**: LayerZero 中每条链的唯一标识符
- **示例**:
  - Ethereum: 30101
  - BSC: 30102
  - Arbitrum: 30110
  - Optimism: 30111

### F

**FeeLib / 费用库**
- **定义**: Stargate 的外部费用计算合约
- **作用**: 计算跨链转账费用
- **风险**: Owner 可设置恶意 FeeLib
- **接口**: `IStargateFeeLib.applyFee()`

**Finality / 最终性**
- **定义**: 交易不可逆的状态
- **类型**:
  - Probabilistic: 基于确认数
  - Deterministic: Ethereum Casper FFG

### G

**Grace Period / 宽限期**
- **定义**: ReceiveLib 升级后新旧库共存的时间
- **作用**: 平滑过渡,防止消息丢失
- **风险**: 旧库漏洞在宽限期内仍可利用
- **推荐**: 尽可能短 (如 100 个区块)

**GUID (Global Unique Identifier) / 全局唯一标识符**
- **定义**: 跨链消息的唯一 ID
- **计算**: `hash(nonce, srcEid, sender, dstEid, receiver)`
- **作用**: 防止消息重放

### H

**hashLookup / 哈希查找表**
- **定义**: ReceiveUln302 中存储 DVN 验证记录的映射
- **结构**: `hashLookup[headerHash][payloadHash][dvnAddress]`
- **内容**: `Verification { submitted: bool, confirmations: uint64 }`
- **Gas 优化**: 验证后删除以回收 Gas

### I

**IGF (Instant Guaranteed Finality) / 即时保证最终性**
- **定义**: Stargate 的核心特性
- **含义**: 源链交易立即成功,无回滚风险
- **实现**: 通过严格的跨链余额同步
- **Zellic 漏洞**: V2 早期版本存在 IGF 破坏漏洞 (已修复)

**Idempotency / 幂等性**
- **定义**: 重复操作不会产生额外影响
- **DVN 应用**: DVN 链下服务防止重复提交 verify()
- **实现**: 检查 `hashLookup` 是否已提交

### L

**LayerZero Labs DVN / LayerZero 官方 DVN**
- **地址**: `0x589dedbd617e0cbcb916a9223f4d1300c294236b`
- **类型**: Push DVN
- **运营者**: LayerZero 团队
- **风险**: 中心化,官方团队控制

**LP (Liquidity Provider) / 流动性提供者**
- **定义**: 为 Stargate Pool 提供流动性的用户
- **收益**: 交易费用分成
- **风险**: Credit 耗尽时无法提取流动性

**lzReceive / LayerZero接收函数**
- **定义**: OApp 实现的消息接收接口
- **签名**: `lzReceive(Origin, guid, message, executor, extraData)`
- **执行者**: Executor
- **Gas**: 由 Executor 提供

### M

**MessageLib / 消息库**
- **定义**: LayerZero 的可升级边缘合约
- **类型**:
  - SendLib: 发送端 (SendUln302)
  - ReceiveLib: 接收端 (ReceiveUln302)
- **可升级**: 是 (但有 grace period)

**M-of-N 模型 / M选N模型**
- **定义**: DVN 验证模型
- **示例**: 3-of-5 表示 5 个 DVN 中至少 3 个验证
- **配置**:
  - Required DVNs: 必须全部验证
  - Optional DVNs: 至少 threshold 个验证

### N

**Nonce / 序号**
- **定义**: 消息的顺序编号
- **类型**:
  - outboundNonce: 发送方序号 (严格递增)
  - inboundNonce: 接收方序号 (可乱序验证,但需顺序执行)
  - lazyInboundNonce: 延迟加载的接收序号

### O

**OFT (Omnichain Fungible Token) / 全链同质化代币**
- **定义**: LayerZero 的跨链代币标准
- **特点**: 单一规范供应量,可在多链间传输
- **合约**: `OFTCore`, `OFT`, `OFTAdapter`
- **Stargate**: 基于 OFT 构建

**ONFT (Omnichain Non-Fungible Token) / 全链非同质化代币**
- **定义**: LayerZero 的跨链 NFT 标准
- **特点**: NFT 可在多链间转移

**Owner / 所有者**
- **定义**: 合约的管理员地址
- **权限**:
  - EndpointV2: 注册库、设置默认库、提取代币
  - Stargate: 设置 FeeLib/Planner/Treasurer、暂停
- **风险**: 权限过大,单点控制

### P

**PacketSent / 数据包发送事件**
- **定义**: SendLib 发出的跨链消息事件
- **监听者**: DVN 链下服务
- **内容**: encodedPacket, options, sendLibrary

**Path / 路径**
- **定义**: Stargate 中的跨链通道
- **包含**: `credit` (余额), `target credit` (目标余额)
- **唯一性**: 每个 (srcEid, dstEid, asset) 一个 path

**Planner / 规划者**
- **定义**: Stargate 的 Bus 控制者
- **权限**: 决定何时发送 credit 到哪条链
- **风险**: 可延迟或优先特定链

**Pull DVN / 拉取式 DVN**
- **定义**: 通过链上读取验证消息的 DVN
- **示例**: lzRead DVN (Nethermind)
- **优势**: 无需链下服务持续运行
- **劣势**: Gas 费用较高

**Push DVN / 推送式 DVN**
- **定义**: 通过链下服务主动提交验证的 DVN
- **示例**: LayerZero Labs, Google Cloud, Chainlink
- **优势**: Gas 效率高
- **劣势**: 需要维护链下服务

### R

**Required DVN / 必需 DVN**
- **定义**: UlnConfig 中必须全部验证的 DVN
- **逻辑**: ALL(required DVNs) must verify
- **默认配置**: LayerZero Labs + Google Cloud (2个)
- **推荐配置**: 至少 3 个来自不同实体

**ReceiveLib / 接收库**
- **定义**: 接收端的 MessageLib
- **主要实现**: ReceiveUln302
- **作用**: 验证 DVN 签名,调用 EndpointV2.verify()

### S

**SendLib / 发送库**
- **定义**: 发送端的 MessageLib
- **主要实现**: SendUln302
- **作用**: 计算费用,发出 PacketSent 事件

**Slash / 罚没**
- **定义**: 惩罚作恶验证者的机制
- **LayerZero 现状**: ❌ 无 Slash 机制
- **风险**: DVN 作恶无经济成本

**Stargate / 星门**
- **定义**: 基于 LayerZero 的跨链桥协议
- **版本**: V1 (2022), V2 (2024)
- **核心特性**: IGF, Unified Liquidity, Native Assets

**Staking / 质押**
- **定义**: 锁定代币以获得奖励
- **Stargate**: LP 质押获得 STG 奖励
- **DVN**: 无质押要求 (风险点)

### T

**Treasurer / 财务官**
- **定义**: Stargate 的资金管理者
- **权限**: 提取累积的 treasury fee
- **风险**: 可随时提取所有费用

**Treasury Fee / 财政费用**
- **定义**: Stargate 协议收取的费用
- **去向**: 由 Treasurer 管理
- **透明度**: 不清晰

**Timelock / 时间锁**
- **定义**: 关键操作延迟执行的机制
- **作用**: 给社区时间响应恶意操作
- **LayerZero 现状**: ❌ 无 Timelock
- **推荐**: 7 天 Timelock

**TVL (Total Value Locked) / 总锁仓价值**
- **定义**: 协议中锁定的资产总价值
- **Stargate**: 数十亿美元

### U

**UlnConfig / Ultra Light Node 配置**
- **定义**: OApp 的 DVN 验证配置
- **结构**:
  ```solidity
  struct UlnConfig {
      uint64 confirmations;
      uint8 requiredDVNCount;
      uint8 optionalDVNCount;
      uint8 optionalDVNThreshold;
      address[] requiredDVNs;
      address[] optionalDVNs;
  }
  ```
- **配置位置**: ReceiveUln302

### V

**Verification / 验证**
- **定义**: DVN 确认消息真实性的过程
- **存储**: `hashLookup[headerHash][payloadHash][dvnAddress]`
- **条件**: ALL required + threshold optional

### W

**Whitelist / 白名单**
- **定义**: 允许执行特定操作的地址列表
- **应用**: OApp 可白名单特定 Executor 或发送者

---

## 缩写对照表 / Abbreviations

| 缩写 | 英文全称 | 中文 | 说明 |
|------|---------|------|------|
| OApp | Omnichain Application | 全链应用 | LayerZero 应用 |
| OFT | Omnichain Fungible Token | 全链同质化代币 | LayerZero 代币标准 |
| ONFT | Omnichain Non-Fungible Token | 全链非同质化代币 | LayerZero NFT标准 |
| DVN | Decentralized Verifier Network | 去中心化验证网络 | LayerZero 验证节点 |
| ULN | Ultra Light Node | 超轻节点 | LayerZero 验证库名称 |
| IGF | Instant Guaranteed Finality | 即时保证最终性 | Stargate 特性 |
| LP | Liquidity Provider | 流动性提供者 | Stargate 用户 |
| TVL | Total Value Locked | 总锁仓价值 | DeFi 指标 |
| PoC | Proof of Concept | 概念验证 | 漏洞证明 |
| DoS | Denial of Service | 拒绝服务攻击 | 攻击类型 |
| DAO | Decentralized Autonomous Organization | 去中心化自治组织 | 治理模型 |
| KYC | Know Your Customer | 了解你的客户 | 身份验证 |

---

## 安全术语 / Security Terms

**Critical / 严重**
- **定义**: 可导致资金损失或协议瘫痪的漏洞
- **本报告**: 7 个 Critical 风险

**High / 高危**
- **定义**: 严重影响协议安全但需特定条件的漏洞
- **Paladin 审计**: 0 个 High

**Medium / 中危**
- **定义**: 有限影响或需复杂攻击条件的漏洞
- **本报告**: 7 个 Medium 风险

**Low / 低危**
- **定义**: 边缘情况或对安全影响有限的问题
- **本报告**: 4 个 Low 风险

**Informational / 信息性**
- **定义**: 代码质量、Gas 优化、最佳实践建议
- **Paladin 审计**: 20 个 Informational

**Exploit / 漏洞利用**
- **定义**: 利用漏洞进行攻击的代码或方法

**Attack Vector / 攻击向量**
- **定义**: 攻击者可利用的攻击路径

**Surface / 攻击面**
- **定义**: 协议可被攻击的所有入口点

---

## 代码术语 / Code Terms

**Immutable / 不可变**
- **示例**: EndpointV2
- **含义**: 部署后无法升级

**Upgradeable / 可升级**
- **示例**: MessageLib
- **含义**: 可通过代理或管理员升级

**Proxy / 代理**
- **类型**: Transparent, UUPS, Beacon
- **作用**: 实现合约升级

**Reentrancy / 重入**
- **定义**: 恶意合约重复调用目标合约
- **防御**: Check-Effects-Interactions 模式

**Integer Overflow / 整数溢出**
- **定义**: 数值超出类型最大值
- **防御**: Solidity 0.8+ 内置检查

---

## 使用建议 / Usage Guidelines

### 翻译规范

1. **专业术语保持一致** - 使用本术语表的翻译
2. **保留英文原文** - 首次出现时标注中英文
3. **代码不翻译** - 变量名、函数名保持英文
4. **注释可翻译** - 代码注释可用中文

### 术语示例

✅ **正确**:
```
DVN (Decentralized Verifier Network, 去中心化验证网络) 是 LayerZero
的核心安全组件。每个 OApp 可以配置自己的 DVN 集合。
```

❌ **错误**:
```
去中心化验证网络是层零的核心安全组件。每个全链应用可以配置自己的
去中心化验证网络集合。
```

---

## 贡献

欢迎补充新术语:
1. 提交 PR 添加术语
2. 包含: 中英文名称、定义、示例、相关概念
3. 保持格式一致

---

**最后更新**: 2025-10-14
**维护者**: Claude (AI Security Researcher)
**仓库**: https://github.com/cyhhao/layerzero-stargate-audit
