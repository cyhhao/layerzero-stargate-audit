# Stargate V2 核心概念 (中文)

> **文档来源**: 基于官方文档和代码分析整理
> **翻译时间**: 2025-10-14
> **原文参考**: https://docs.stargate.finance/
> **审计报告**: [Stargate完整审计](../../analysis/04-Stargate-Complete-Audit.md)

---

## 目录

1. [协议概述](#协议概述)
2. [核心架构](#核心架构)
3. [关键概念](#关键概念)
4. [工作流程](#工作流程)
5. [安全模型](#安全模型)
6. [使用指南](#使用指南)

---

## 协议概述

### 什么是 Stargate?

Stargate 是基于 LayerZero 构建的**可组合跨链流动性传输协议** (Composable Native Asset Bridge)，实现了**即时保证最终性** (Instant Guaranteed Finality, IGF)。

**核心定位**:
- 🌉 全链资产桥 - 原生资产跨链转移
- 💧 统一流动性池 - 单一流动性支持多链
- ⚡ 即时最终性 - 源链立即成功，无回滚风险
- 🔗 可组合性 - 跨链+调用一体化

### V2 主要改进

**Stargate V2 (2024年5月)** 相比 V1 的重大升级:

| 特性 | V1 | V2 | 改进 |
|------|----|----|------|
| **Gas费用** | 高 | 大幅降低 | Bus批量传输 |
| **资本效率** | 中等 | 显著提升 | Credit优化机制 |
| **原生资产** | 部分支持 | 完整支持 | StargatePoolNative |
| **流动性管理** | 单一策略 | 灵活调控 | Planner + Bus |
| **费用透明度** | 固定 | 动态可调 | FeeLib外部化 |
| **IGF** | ✅ | ✅ | 保持核心特性 |

**V2核心创新**:
1. **Bus机制** - 批量credit传输，降低Gas 50-80%
2. **Credit优化** - 更精细的路径余额管理
3. **原生资产池** - ETH等原生代币直接桥接
4. **模块化费用** - FeeLib可升级

---

## 核心架构

### 整体架构图

```
┌────────────────────────────────────────────────────────┐
│              Stargate V2 完整架构                        │
└────────────────────────────────────────────────────────┘

                    ┌─────────────────┐
                    │   用户 / dApp   │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │ StargateComposer│
                    │  (跨链+调用)     │
                    └────────┬────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
┌───────▼───────┐   ┌────────▼────────┐   ┌──────▼──────┐
│StargatePoolETH│   │StargatePoolUSDC │   │StargatePool │
│ (原生资产)     │   │ (ERC20代币)     │   │    USDT     │
└───────┬───────┘   └────────┬────────┘   └──────┬──────┘
        │                    │                    │
        └────────────────────┼────────────────────┘
                             │
                    ┌────────▼────────┐
                    │  StargateBase   │
                    │  (核心逻辑)      │
                    │                 │
                    │ ├─ Credit机制   │
                    │ ├─ Bus机制      │
                    │ ├─ Fee计算      │
                    │ └─ 路径管理     │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │ TokenMessaging  │
                    │   (OFT集成)      │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │  LayerZero V2   │
                    │   (消息传递)     │
                    └─────────────────┘

外部组件:
┌──────────┐  ┌───────────┐  ┌──────────┐
│  FeeLib  │  │  Planner  │  │Treasurer │
│ (费用计算)│  │(Bus控制)  │  │(资金管理)│
└──────────┘  └───────────┘  └──────────┘
```

### 合约继承关系

```solidity
// StargatePool 继承链
StargatePool
  └─ StargateBase
      ├─ StargatePoolMigratable
      ├─ TokenMessaging
      │   └─ OFTCore
      │       └─ OAppCore
      │           └─ EndpointV2 (LayerZero)
      └─ Ownable
```

**合约职责**:

| 合约 | 职责 |
|-----|------|
| **StargatePool** | 特定资产的流动性池 (USDC, USDT等) |
| **StargatePoolNative** | 原生资产池 (ETH, AVAX等) |
| **StargateBase** | 核心业务逻辑 (Credit, Bus, Fee) |
| **TokenMessaging** | OFT跨链代币标准集成 |
| **FeeLib** | 外部费用计算库 |
| **StargateStaking** | LP质押奖励 |
| **Treasurer** | 资金管理和费用提取 |

---

## 关键概念

### 1. Instant Guaranteed Finality (IGF)

**定义**: 源链交易立即成功，目标链保证执行，无回滚风险

**实现原理**:
```
传统跨链桥:
  源链锁定资产 → 等待目标链确认 → 目标链释放
  问题: 等待时间长，可能失败

Stargate IGF:
  源链立即扣除 (本地finality) → 目标链保证有足够流动性
  关键: 严格的跨链余额同步机制
```

**代码体现**:
```solidity
// 源链: Ethereum
function sendToken(
    uint32 _dstEid,
    uint256 _amount,
    address _to
) external payable returns (MessagingReceipt memory) {
    // 1. 立即从池中扣除 (IGF!)
    balanceOf[msg.sender] -= _amountLD;
    tvl -= _amountSD;

    // 2. 减少目标路径的credit
    paths[_dstEid].credit -= _amountSD;

    // 3. 发送LayerZero消息
    MessagingReceipt memory receipt = _send(...);

    // 用户立即看到成功! 无需等待目标链
    return receipt;
}

// 目标链: BSC
function lzReceive(...) internal override {
    // 1. 增加本地路径的credit
    paths[localEid].credit += _amountSD;

    // 2. 从池中释放给用户
    tvl += _amountSD;
    _safeTransfer(_to, _amountLD);

    // 保证执行成功!
}
```

**IGF的保证**:
- ✅ 源链立即成功
- ✅ 无双花风险
- ✅ 无回滚 (除非源链整体回滚)
- ⚠️ 依赖严格的Credit机制

**Zellic审计发现** (已修复):
> V2早期版本存在余额不同步漏洞，可能导致IGF破坏和资金锁定。
> 已在正式发布前修复。

### 2. Credit 机制

**定义**: 路径级别的流动性平衡管理

**数据结构**:
```solidity
struct Path {
    uint64 credit;        // 该路径当前可用的credit
    uint64 lkb;           // Last Known Block
    uint128 targetCredit; // Planner设置的目标credit
}

// 每个Pool维护多个路径
mapping(uint32 eid => Path) public paths;
```

**Credit流动**:
```
初始状态 (Ethereum Pool):
  paths[Ethereum].credit = 1000
  paths[BSC].credit = 500
  paths[Arbitrum].credit = 500
  总TVL = 2000

用户从 Ethereum → BSC 转账 100:
  ├─ paths[BSC].credit -= 100       (BSC路径减少)
  ├─ Ethereum Pool TVL -= 100
  └─ LayerZero消息 → BSC

BSC收到消息:
  ├─ paths[BSC].credit += 100       (BSC本地路径增加)
  ├─ BSC Pool TVL += 100
  └─ 释放给用户

新状态:
  Ethereum:
    paths[BSC].credit = 400  (500-100)
    TVL = 1900

  BSC:
    paths[BSC].credit = 1100  (1000+100)
    TVL = 2100
```

**Credit耗尽问题** ⚠️:
```solidity
function redeem(uint256 _amountLD, address _receiver)
    external
    returns (uint256)
{
    uint64 amountSD = _ld2sd(_amountLD);

    // 🔴 Critical: Credit耗尽时redeem失败!
    if (paths[localEid].credit < amountSD) {
        revert Stargate_InsufficientCredits();
    }

    // LP无法提取流动性
    paths[localEid].credit -= amountSD;
    // ...
}
```

**触发场景**:
```
市场恐慌时:
  Ethereum → BSC 大量单向流出
  ├─ paths[BSC].credit快速减少
  ├─ 最终 paths[BSC].credit = 0
  └─ 所有BSC上的redeem()失败
      → LP资金被锁定!
      → 只能等待反向跨链补充credit
```

**缓解方案** (审计建议):
```solidity
// 建议: 保留emergency withdrawal能力
uint64 constant EMERGENCY_RESERVE = 10% of TVL;

function redeem(...) external returns (uint256) {
    uint64 amountSD = _ld2sd(_amountLD);

    // 允许提取到emergency reserve
    uint64 reservedCredit = uint64(tvl * EMERGENCY_RESERVE / 100);
    require(
        paths[localEid].credit - amountSD >= reservedCredit,
        "Below emergency reserve"
    );

    // ...
}
```

### 3. Bus 机制

**定义**: 批量Credit传输机制，显著降低Gas费用

**传统方式 vs Bus**:
```
传统 (每次跨链都单独发送credit):
  用户1: Eth→BSC (发送100 credit) → Gas: 0.01 ETH
  用户2: Eth→BSC (发送50 credit)  → Gas: 0.01 ETH
  用户3: Eth→BSC (发送200 credit) → Gas: 0.01 ETH
  总Gas: 0.03 ETH

Bus机制 (批量发送):
  Planner定期触发:
    sendCredits(BSC, [
      (user1, 100),
      (user2, 50),
      (user3, 200)
    ]) → Gas: 0.015 ETH

  节省: 50% Gas!
```

**代码实现**:
```solidity
struct TargetCredit {
    uint32 srcEid;      // 源链
    uint64 amount;      // credit数量
    uint64 minAmount;   // 最小数量
}

// Planner调用 (onlyPlanner)
function sendCredits(
    uint32 _dstEid,
    TargetCredit[] calldata _credits
) external payable onlyPlanner {
    // 1. 批量打包credit
    bytes memory payload = abi.encode(_credits);

    // 2. 一次LayerZero消息发送所有
    _send(_dstEid, payload, ...);

    // 目标链接收并分配credit
}
```

**Planner角色**:
```
Planner职责:
  ├─ 监控各路径credit水平
  ├─ 决定何时发车 (sendCredits)
  ├─ 选择目标链
  └─ 优化Gas效率

风险:
  ⚠️ Planner可延迟发送
  ⚠️ Planner可优先某些链
  → 需要监督和透明度
```

### 4. 费用机制

**费用结构**:
```
总费用 = LayerZero费用 + Stargate费用

Stargate费用:
  ├─ LP Fee (给流动性提供者)
  ├─ Treasury Fee (协议收入)
  ├─ Reward Fee (质押奖励)
  └─ 动态调整 (基于流动性和路径)
```

**FeeLib外部化**:
```solidity
// StargateBase.sol
function _chargeFee(
    FeeParams memory _params,
    uint64 _minAmountOutSD
) internal returns (uint64 amountOutSD) {
    // 🔴 Critical: 完全信任外部FeeLib!
    amountOutSD = IStargateFeeLib(feeLib).applyFee(_params);

    // ⚠️ 无验证! Owner可设置恶意FeeLib
    // ⚠️ 恶意FeeLib可返回amountOut=0

    require(amountOutSD >= _minAmountOutSD, "Slippage");
    // ...
}
```

**风险**:
- ⚠️ Owner可更换FeeLib为恶意合约
- ⚠️ 恶意FeeLib返回amountOut=0 → 用户损失100%
- ⚠️ 无合理性验证

**审计建议**:
```solidity
// 建议: 添加合理性检查
function _chargeFee(...) internal returns (uint64) {
    uint64 amountOutSD = IStargateFeeLib(feeLib).applyFee(_params);

    // ✅ 添加: 最大费率检查 (如10%)
    uint64 maxFee = _params.amountInSD * 10 / 100;
    require(
        _params.amountInSD - amountOutSD <= maxFee,
        "Fee too high"
    );

    // ✅ 添加: 最小输出检查 (如90%)
    require(
        amountOutSD >= _params.amountInSD * 90 / 100,
        "Output too low"
    );

    // ...
}
```

### 5. 统一流动性池

**概念**: 单一流动性池支持多条链的跨链转账

**对比**:
```
传统多池模型:
  Ethereum Pool (1000 USDC)  ←→  BSC Pool (500 USDC)
  问题: 流动性分散，效率低

Stargate统一池:
  全局Pool (1500 USDC)
    ├─ Ethereum路径: 1000 credit
    └─ BSC路径: 500 credit
  优势: 流动性统一，效率高
```

**LP代币**:
```solidity
// LP存入资产
function deposit(uint256 _amountLD) external returns (uint256 shares) {
    // 1. 计算LP份额
    shares = _amountLD * totalSupply / tvl;

    // 2. 铸造LP代币
    _mint(msg.sender, shares);

    // 3. 增加TVL
    tvl += _ld2sd(_amountLD);
    paths[localEid].credit += _ld2sd(_amountLD);

    return shares;
}

// LP赎回资产
function redeem(uint256 shares) external returns (uint256 amountLD) {
    // 🔴 Critical: 需要检查credit!
    require(paths[localEid].credit >= amount, "Insufficient credits");

    // 计算份额价值
    amountLD = shares * tvl / totalSupply;

    // 销毁LP代币
    _burn(msg.sender, shares);

    // 减少TVL
    tvl -= _ld2sd(amountLD);
    paths[localEid].credit -= _ld2sd(amountLD);

    // 转账
    _safeTransfer(msg.sender, amountLD);
}
```

**LP收益来源**:
```
LP收益:
  ├─ 交易费用 (每笔跨链转账)
  ├─ 质押奖励 (StargateStaking)
  └─ TVL增长

风险:
  ⚠️ Credit耗尽时无法redeem
  ⚠️ 价格波动 (如原生资产)
  ⚠️ 智能合约风险
```

---

## 工作流程

### 完整跨链转账流程

**用户视角**:
```
用户想从 Ethereum 转 1000 USDC 到 BSC

1. 用户调用 Stargate:
   StargatePool(USDC).sendToken{value: fee}(
     dstEid: 30102,      // BSC
     amount: 1000 USDC,
     to: 0x...,          // BSC地址
     minAmountOut: 995   // 最小接收量 (滑点保护)
   )

2. 源链立即成功 (IGF!)
   ✅ USDC从用户扣除
   ✅ 交易确认
   ✅ 用户可以离开

3. 等待 LayerZero 传递 (~1-5分钟)

4. 目标链执行
   ✅ BSC收到USDC
   ✅ 转账到用户地址
```

**合约视角**:

**阶段1: 源链 (Ethereum)**
```solidity
// StargatePool(USDC).sendToken()
function sendToken(...) external payable returns (MessagingReceipt) {
    // 1. 验证参数
    require(_dstEid != 0, "Invalid dest");
    require(_amount > 0, "Invalid amount");

    // 2. 计算费用
    FeeParams memory feeParams = FeeParams({
        sender: msg.sender,
        dstEid: _dstEid,
        amountInSD: _ld2sd(_amountLD),
        isTaxi: false
    });

    uint64 amountOutSD = _chargeFee(feeParams, _minAmountOutSD);

    // 3. 更新状态 (IGF! 立即扣除)
    tvl -= _ld2sd(_amountLD);
    paths[_dstEid].credit -= amountOutSD;

    // 4. 构造消息
    bytes memory message = abi.encode(
        SEND,               // 消息类型
        _to,                // 目标地址
        amountOutSD,        // 输出数量
        _composeMsg         // 可选组合调用
    );

    // 5. 发送 LayerZero 消息
    MessagingReceipt memory receipt = _send(
        _dstEid,
        message,
        _options,
        MessagingFee(msg.value, 0),
        _refundAddress
    );

    // 6. 发出事件
    emit OFTSent(guid, _dstEid, msg.sender, amountOutSD);

    return receipt;
}
```

**阶段2: LayerZero传递**
```
LayerZero消息生命周期:
1. SendUln302.send() → emit PacketSent
2. DVN监听并验证
3. ReceiveUln302.verify()
4. commitVerification()
5. 等待Executor执行
```

**阶段3: 目标链 (BSC)**
```solidity
// TokenMessaging.lzReceive() (OFT)
function lzReceive(...) internal override {
    // 1. 解码消息
    (uint8 msgType, bytes memory _payload) = abi.decode(
        _message,
        (uint8, bytes)
    );

    // 2. 路由消息类型
    if (msgType == SEND) {
        _handleSend(_origin, _payload, _executor);
    } else if (msgType == SEND_AND_CALL) {
        _handleSendAndCall(_origin, _payload, _executor);
    }
}

// StargateBase._handleSend()
function _handleSend(Origin memory _origin, bytes memory _payload) internal {
    // 1. 解码payload
    (address to, uint64 amountSD, bytes memory composeMsg) =
        abi.decode(_payload, (address, uint64, bytes));

    // 2. 更新状态
    tvl += amountSD;
    paths[localEid].credit += amountSD;  // 恢复credit

    // 3. 转账给用户
    uint256 amountLD = _sd2ld(amountSD);
    _safeTransfer(to, amountLD);

    // 4. 发出事件
    emit OFTReceived(_origin, to, amountSD);

    // 5. 可选: 执行组合调用
    if (composeMsg.length > 0) {
        _executeCompose(to, composeMsg);
    }
}
```

### Taxi vs Bus 模式

**Taxi模式** (即时发送):
```
用户支付额外费用，立即发送credit

StargatePool.sendToken(..., isTaxi: true)
  → 立即发送LayerZero消息 (带credit)
  → 目标链立即恢复credit
  → Gas较高，但实时性好
```

**Bus模式** (批量发送):
```
正常跨链，credit稍后通过Bus批量发送

StargatePool.sendToken(..., isTaxi: false)
  → 不立即发送credit
  → Planner定期调用sendCredits()批量发送
  → Gas大幅降低，但有延迟

Planner.sendCredits(dstEid, [
    TargetCredit(srcEid1, 1000),
    TargetCredit(srcEid2, 500),
    ...
]) → 一次消息发送所有credit
```

---

## 安全模型

### 继承LayerZero安全

Stargate完全依赖LayerZero的消息传递:
```
Stargate安全 ⊆ LayerZero安全

如果LayerZero被攻破:
  → 伪造跨链消息
  → 在目标链凭空铸造代币
  → 耗尽流动性池
```

**DVN配置至关重要**:
```solidity
// Stargate应该配置强DVN
UlnConfig({
    confirmations: 64,
    requiredDVNs: [LZ Labs, Chainlink, Nethermind],
    optionalDVNs: [Google, Polyhedra, BCW, Axelar, Switchboard],
    optionalDVNThreshold: 3
})
```

### Stargate特有风险

#### 1. Owner中心化

**Owner权限** (StargateBase):
```solidity
contract StargateBase is Ownable {
    // 🔴 Critical权限
    function setFeeLib(address _feeLib) external onlyOwner {
        feeLib = _feeLib;
    }

    function setPlanner(address _planner) external onlyOwner {
        planner = _planner;
    }

    function setTreasurer(address _treasurer) external onlyOwner {
        treasurer = _treasurer;
    }

    function pause() external onlyOwner {
        _pause();
    }

    function unpause() external onlyOwner {
        _unpause();
    }
}
```

**攻击场景**:
```
场景1: Owner设置恶意FeeLib
  → feeLib.applyFee() 返回 amountOut = 0
  → 用户跨链1000 USDC，收到0
  → 资金损失100%

场景2: Owner暂停合约
  → 所有跨链转账停止
  → 流动性被锁定
  → LP无法提取

场景3: Owner频繁更换Planner
  → Bus机制混乱
  → Credit不平衡
  → 影响用户体验
```

**缓解建议**:
1. ✅ Owner迁移到多签 (至少3/5)
2. ✅ 实施Timelock (7天)
3. ✅ 披露Owner地址和治理模型
4. ✅ 限制FeeLib更换频率

#### 2. Credit耗尽锁定

**问题**: 详见前文 [Credit机制](#2-credit-机制)

**触发条件**:
- 大量单向跨链流动
- 市场恐慌或套利机会
- Planner延迟发送credit

**影响**:
- LP无法提取流动性
- 资金被长期锁定
- 需等待反向跨链补充

**解决方案**:
```solidity
// 建议1: Emergency withdrawal
uint64 constant EMERGENCY_RESERVE = 10%;

function emergencyRedeem(uint256 shares) external {
    require(
        paths[localEid].credit < tvl * EMERGENCY_RESERVE / 100,
        "Not emergency"
    );

    // 允许LP提取部分流动性
    uint256 maxWithdraw = tvl * EMERGENCY_RESERVE / 100;
    uint256 amountLD = min(shares * tvl / totalSupply, maxWithdraw);

    // ...
}

// 建议2: 动态费用
function _chargeFee(...) internal returns (uint64) {
    // credit低时提高费用，抑制流出
    if (paths[_dstEid].credit < targetCredit * 20 / 100) {
        fee = fee * 150 / 100;  // 增加50%
    }
    // ...
}
```

#### 3. Treasurer特权

```solidity
function withdrawFee(address _to, uint256 _amountLD) external onlyTreasurer {
    // ⚠️ Treasurer可随时提取所有累积费用
    treasuryFee -= _ld2sd(_amountLD);
    _safeTransfer(_to, _amountLD);
}
```

**风险**:
- Treasurer私钥被盗 → 所有费用被提取
- 费用分配不透明 → LP收益受损
- 无提取频率限制 → 可能被滥用

**建议**:
1. Treasurer使用多签
2. 实施提取冷却期
3. 公开费用分配方案

---

## 使用指南

### 对LP (流动性提供者)

#### 存入流动性
```solidity
// 1. 授权
IERC20(USDC).approve(stargatePool, amount);

// 2. 存入
IStargatePool(stargatePool).deposit(amount, msg.sender);

// 返回LP代币
```

#### 监控Credit状态
```bash
# 查询当前credit
cast call <pool-address> \
  "paths(uint32)(uint64,uint64,uint128)" \
  <eid> \
  --rpc-url <rpc>

# 返回: (credit, lkb, targetCredit)
```

⚠️ **警告**: 如果credit低于targetCredit的20%，考虑降低仓位

#### 赎回流动性
```solidity
// 检查credit
uint64 credit = pool.paths(localEid).credit;
require(credit > 0, "Credit exhausted!");

// 赎回
pool.redeem(shares, msg.sender);
```

### 对开发者

#### 集成Stargate Composer
```solidity
// 跨链+调用一体化
IStargateComposer(composer).sendToken{value: fee}(
    SendParam({
        dstEid: 30102,
        to: bytes32(receiver),
        amountLD: 1000e6,
        minAmountLD: 995e6,
        extraOptions: "",
        composeMsg: abi.encode(
            "swap", // 目标链操作
            swapParams
        ),
        oftCmd: ""
    }),
    refundAddress
);
```

#### 处理组合调用
```solidity
contract MyDApp {
    function lzCompose(
        address _from,
        bytes32 _guid,
        bytes calldata _message,
        address _executor,
        bytes calldata _extraData
    ) external payable {
        // 1. 验证调用者
        require(msg.sender == stargateComposer);

        // 2. 解码消息
        (string memory action, bytes memory params) =
            abi.decode(_message, (string, bytes));

        // 3. 执行操作
        if (keccak256(bytes(action)) == keccak256("swap")) {
            _executeSwap(params);
        }
    }
}
```

### 最佳实践

#### ✅ DO

1. **监控Credit状态**
   - 定期检查paths[eid].credit
   - Credit<20% targetCredit时警惕

2. **设置合理滑点**
   - 考虑费用波动
   - 建议最小1-2%

3. **使用可信DVN配置**
   - 验证Stargate的UlnConfig
   - 确保至少3个required DVNs

4. **分散LP风险**
   - 不要将所有资金放在单一Pool
   - 分散到多个资产和链

#### ❌ DON'T

1. **忽略Credit警告**
   - Credit低时仍大量存入
   - 可能被长期锁定

2. **在市场恐慌时大额操作**
   - 单向流动可能耗尽credit
   - 等待市场稳定

3. **盲目信任Owner**
   - 关注Owner地址变更
   - 关注FeeLib更新

4. **忽略安全公告**
   - 关注官方Twitter
   - 订阅安全通知

---

## 相关资源

### 官方资源
- 🔗 [Stargate Docs](https://docs.stargate.finance/)
- 🔗 [GitHub](https://github.com/stargate-protocol/stargate-v2)
- 🔗 [GitBook](https://stargateprotocol.gitbook.io/stargate/)
- 🔗 [官网](https://stargate.finance/)

### 审计报告
- 📄 [Stargate完整审计](../../analysis/04-Stargate-Complete-Audit.md)
- 🔗 [Zellic审计](https://github.com/stargate-protocol/stargate-v2/blob/main/audits/)
- 🔗 [Ottersec审计](https://github.com/stargate-protocol/stargate-v2/blob/main/audits/)

### Bug Bounty
- 🔗 [Immunefi Program](https://immunefi.com/bug-bounty/stargate/)
- 💰 最高奖励: $10,000,000

### 社区
- 🔗 [Discord](https://discord.gg/stargate)
- 🔗 [Twitter](https://twitter.com/StargateFinance)
- 🔗 [Forum](https://stargate.discourse.group/)

---

**文档版本**: v1.0
**最后更新**: 2025-10-14
**维护者**: Claude (AI Security Researcher)
**反馈**: https://github.com/cyhhao/layerzero-stargate-audit/issues
