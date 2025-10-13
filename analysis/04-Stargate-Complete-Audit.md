# Stargate V2 完整安全审计

## 1. Stargate概述

### 1.1 基本信息
- **协议名称**: Stargate Finance V2
- **类型**: Omnichain Liquidity Pool + 跨链资产桥
- **基于**: LayerZero V2消息传递协议
- **TVL**: 数亿美元（跨多条链）
- **支持资产**: USDC, USDT, ETH, mETH等

### 1.2 核心创新（V2）

**Bus模式**:
- 批量处理3-5笔交易再发送
- Gas成本从~550k降至~100k（80%+节省）
- 本地结算，延迟发送

**Wrapped Asset Bridge**:
- 使用underlying asset作为POL（protocol owned liquidity）
- 原生资产桥接到Stargate wrapped OFT
- 可跨链转账并在有native pool的地方赎回

---

## 2. Stargate架构分析

### 2.1 合约组件（Ethereum Mainnet）

| 合约 | 地址 | 职责 |
|-----|------|------|
| StargatePoolUSDC | `0xc026395860...583189C7` | USDC流动性池 |
| StargatePoolUSDT | `0x933597a323...72fd5A3973` | USDT流动性池 |
| StargatePoolNative | `0x77b2043768...bcE57931` | ETH流动性池 |
| StargatePoolmETH | `0x268Ca24DAe...cbbB79E931D` | mETH流动性池 |
| TokenMessaging | `0x6d6620eFa7...3f4A980` | 跨链消息管理 |
| StargateStaking | `0xFF551fEDdb...Cb644Bb04A6BD` | 质押奖励 |
| StargateMultiRewarder | `0x5871A7f88b...4Bb04A6BD` | 多重奖励分配 |
| Treasurer | `0x1041D127b2...89502606918` | 资金管理 |

### 2.2 合约继承结构

```
StargatePool
├── StargateBase
│   ├── OApp (LayerZero)
│   ├── ITokenMessagingHandler
│   ├── ICreditMessagingHandler
│   └── Transfer
└── LPToken (ERC20)
```

**关键父合约**:
- `StargateBase`: 核心逻辑，处理跨链转账、费用、credit管理
- `OApp`: LayerZero V2的OmniChain Application标准接口
- `LPToken`: 流动性提供者代币

---

## 3. StargatePool核心功能分析

### 3.1 存款 (deposit)

**源码**: StargatePool.sol:55-69

```solidity
function deposit(address _receiver, uint256 _amountLD)
    external payable nonReentrantAndNotPaused returns (uint256 amountLD)
{
    // 1. 收取用户代币
    _assertMsgValue(_amountLD);
    uint64 amountSD = _inflow(msg.sender, _amountLD);

    // 2. 增加credit和池余额
    _postInflow(amountSD);

    // 3. 铸造LP代币给接收者（1:1）
    amountLD = _sd2ld(amountSD);
    lp.mint(_receiver, amountLD);
    tvlSD += amountSD;

    emit Deposited(msg.sender, _receiver, amountLD);
}
```

**安全机制**:
- ✅ `nonReentrantAndNotPaused`: 防重入 + 暂停检查
- ✅ LP代币1:1铸造，公平分配
- ✅ 增加TVL和poolBalance跟踪

**潜在风险**:
- 🟡 没有存款上限，可能导致池过大
- 🟡 没有最小存款要求，可能被滥用

### 3.2 赎回 (redeem)

**源码**: StargatePool.sol:77-92

```solidity
function redeem(uint256 _amountLD, address _receiver)
    external nonReentrantAndNotPaused returns (uint256 amountLD)
{
    uint64 amountSD = _ld2sd(_amountLD);

    // 1. 减少credit（确保流动性充足）
    paths[localEid].decreaseCredit(amountSD);

    // 2. 去除精度损失，烧毁LP代币
    amountLD = _sd2ld(amountSD);
    lp.burnFrom(msg.sender, amountLD);
    tvlSD -= amountSD;

    // 3. 转账underlying token给接收者
    _safeOutflow(_receiver, amountLD);
    _postOutflow(amountSD);

    emit Redeemed(msg.sender, _receiver, amountLD);
}
```

**安全机制**:
- ✅ 检查credit充足性（`paths[localEid].credit >= amountSD`）
- ✅ 烧毁LP代币防止双花
- ✅ `_safeOutflow`: 如果转账失败会revert

**关键发现** 🔴:
```solidity
paths[localEid].decreaseCredit(amountSD);
```
如果local credit不足，**赎回会失败**！

**风险场景**:
```
1. 池中有1000 USDC
2. 但local credit只有500（另外500被发送到其他链）
3. 用户想赎回600 USDC
4. 虽然池有足够余额，但credit不足
5. 赎回失败！

结果: 流动性锁定，用户无法提款
```

### 3.3 跨链转账 (sendToken)

**源码**: StargateBase.sol:247-281

```solidity
function sendToken(SendParam calldata _sendParam, MessagingFee calldata _fee, address _refundAddress)
    public payable nonReentrantAndNotPaused
    returns (MessagingReceipt memory msgReceipt, OFTReceipt memory oftReceipt, Ticket memory ticket)
{
    // Step 1: 收取代币并扣费
    (bool isTaxi, uint64 amountInSD, uint64 amountOutSD) = _inflowAndCharge(_sendParam);

    // Step 2: 生成OFT收据
    oftReceipt = OFTReceipt(_sd2ld(amountInSD), _sd2ld(amountOutSD));

    // Step 3: 验证消息费用
    MessagingFee memory messagingFee = _assertMessagingFee(_fee, oftReceipt.amountSentLD);

    // Step 4: 发送（Taxi或Bus模式）
    if (isTaxi) {
        msgReceipt = _taxi(_sendParam, messagingFee, amountOutSD, _refundAddress);
    } else {
        (msgReceipt, ticket) = _rideBus(_sendParam, messagingFee, amountOutSD, _refundAddress);
    }

    emit OFTSent(msgReceipt.guid, _sendParam.dstEid, msg.sender, oftReceipt.amountSentLD, oftReceipt.amountReceivedLD);
}
```

**费用机制**:
```solidity
// _inflowAndCharge()
amountInSD = _inflow(msg.sender, _sendParam.amountLD);  // 收取代币
FeeParams memory feeParams = _buildFeeParams(_sendParam.dstEid, amountInSD, isTaxi);
amountOutSD = _chargeFee(feeParams, _ld2sd(_sendParam.minAmountLD));  // 扣费/奖励

// 费用可能是正（收费）或负（奖励）
if (amountOutSD < amountInSD) {
    // 收费
    treasuryFee += amountInSD - amountOutSD;
} else if (amountOutSD > amountInSD) {
    // 奖励（从treasury扣除）
    (amountOutSD, reward) = _capReward(amountOutSD, reward);
    treasuryFee -= reward;
}
```

**Credit机制**:
```solidity
paths[_sendParam.dstEid].decreaseCredit(amountOutSD);  // 减少目标链credit
_postInflow(amountOutSD);  // 增加本地poolBalance
```

**两种发送模式**:

#### Taxi模式（立即发送）
- 单笔交易立即发送
- Gas成本高（~550k）
- 适合急需的转账
- 支持composeMsg（目标链上的回调）

#### Bus模式（批量发送）
- 等待3-5笔交易凑齐再发送
- Gas成本低（~100k per tx）
- 本地结算，延迟发送
- 不支持composeMsg

### 3.4 接收代币 (receiveToken)

**Taxi模式** (StargateBase.sol:344-374):
```solidity
function receiveTokenTaxi(
    Origin calldata _origin,
    bytes32 _guid,
    address _receiver,
    uint64 _amountSD,
    bytes calldata _composeMsg
) external nonReentrantAndNotPaused onlyCaller(tokenMessaging) {
    uint256 amountLD = _sd2ld(_amountSD);

    // 尝试转账
    bool success = _outflow(_receiver, amountLD);
    if (success) {
        _postOutflow(_amountSD);
        // 如果有composeMsg，发送到endpoint
        if (_composeMsg.length > 0) {
            endpoint.sendCompose(_receiver, _guid, 0, composeMsg);
        }
        emit OFTReceived(_guid, _origin.srcEid, _receiver, amountLD);
    } else {
        // 转账失败，缓存到unreceivedTokens
        unreceivedTokens[_guid][0] = keccak256(...);
        emit UnreceivedTokenCached(_guid, 0, _origin.srcEid, _receiver, amountLD, composeMsg);
    }
}
```

**Bus模式** (StargateBase.sol:321-341):
- 类似逻辑，但使用seatNumber索引
- 不支持composeMsg

**失败处理**:
- 如果`_outflow()`失败（如gas不足、接收合约revert），代币被缓存
- 用户可以调用`retryReceiveToken()`重试

---

## 4. 安全机制评估

### 4.1 防重入保护 ✅

**实现**: StargateBase.sol:67-77
```solidity
modifier nonReentrantAndNotPaused() {
    if (status != NOT_ENTERED) {
        if (status == ENTERED) revert Stargate_ReentrantCall();
        revert Stargate_Paused();
    }
    status = ENTERED;
    _;
    status = NOT_ENTERED;
}
```

**评价**: ✅ 正确实现，所有关键函数都使用

### 4.2 权限管理

**角色定义**:
```solidity
address public owner;       // 合约拥有者
address public planner;     // 运营计划者（可暂停）
address public treasurer;   // 资金管理者（可提取费用）
address public tokenMessaging;     // 跨链消息合约
address public creditMessaging;    // credit管理合约
address public feeLib;      // 费用计算库
```

**权限检查**:
```solidity
modifier onlyCaller(address _caller) {
    if (msg.sender != _caller) revert Stargate_Unauthorized();
    _;
}
```

**关键权限**:

| 角色 | 权限 | 风险 |
|-----|------|------|
| **owner** | setAddressConfig (设置所有角色)<br>setOFTPath (设置OFT路径) | 🔴 Critical |
| **treasurer** | withdrawTreasuryFee (提取费用)<br>addTreasuryFee (增加费用)<br>recoverToken (恢复误发代币) | 🟡 Medium |
| **planner** | setPause (暂停/恢复)<br>withdrawPlannerFee (提取gas费)<br>setDeficitOffset (Pool特有) | 🟡 Medium |
| **tokenMessaging** | receiveTokenTaxi/Bus (接收代币) | 🟡 Medium |
| **creditMessaging** | sendCredits / receiveCredits | 🟢 Low |

### 4.3 暂停机制 ✅

**实现**: StargateBase.sol:208-212
```solidity
function setPause(bool _paused) external onlyCaller(planner) {
    if (status == ENTERED) revert Stargate_ReentrantCall();
    status = _paused ? PAUSED : NOT_ENTERED;
    emit PauseSet(_paused);
}
```

**评价**: ✅ 正确实现，可以紧急暂停

---

## 5. 主要安全风险

### 5.1 🔴 Critical: Owner权限过大

**问题**:
Owner可以通过`setAddressConfig()`更换所有关键角色。

```solidity
function setAddressConfig(AddressConfig calldata _config) external onlyOwner {
    feeLib = _config.feeLib;           // ← 可以设置恶意费用计算
    planner = _config.planner;         // ← 可以暂停合约
    treasurer = _config.treasurer;     // ← 可以提取所有费用
    tokenMessaging = _config.tokenMessaging;  // ← 可以伪造接收
    creditMessaging = _config.creditMessaging;
    lzToken = _config.lzToken;
}
```

**攻击场景**:
```
1. 恶意Owner设置恶意feeLib
2. 恶意feeLib返回amountOutSD = 0（全部扣为费用）
3. 用户发送1000 USDC
4. 目标链只收到0 USDC，1000 USDC进入treasuryFee
5. Owner通过treasurer提取treasuryFee
6. 结果: 窃取用户资金
```

**缓解建议**:
- Owner应该是多签（至少3/5）
- 关键操作应该有Timelock（7天）
- 角色更换应该有延迟生效

### 5.2 🔴 Critical: Credit枯竭导致流动性锁定

**问题**:
赎回依赖local credit充足，但credit可能被发送到其他链。

**源码**: StargatePool.sol:79
```solidity
paths[localEid].decreaseCredit(amountSD);  // ← 如果credit不足会revert
```

**攻击/风险场景**:
```
初始状态:
- poolBalance = 1000 USDC
- paths[Ethereum].credit = 1000

操作:
1. 用户A发送800 USDC到BSC
   - paths[BSC].credit -= 800
   - paths[Ethereum].credit += 800 (本地增加)
   - poolBalance = 1000 (不变)

2. 用户B发送300 USDC到Arbitrum
   - paths[Arbitrum].credit -= 300
   - paths[Ethereum].credit += 300
   - poolBalance = 1000

当前状态:
- poolBalance = 1000 USDC
- paths[Ethereum].credit = 1600
- paths[BSC].credit = 0
- paths[Arbitrum].credit = 0

问题:
3. 用户C想赎回700 USDC
   - 池有1000 USDC（足够）
   - 但需要decreaseCredit(700)
   - paths[Ethereum].credit = 1600 → 1600-700 = 900 ✅成功

4. 但如果用户D想赎回1100 USDC（有1100 LP）
   - 池有1000 USDC（不够！）
   - paths[Ethereum].credit = 900（不够！）
   - 赎回失败！
```

**真实风险**:
- 如果大量资金被发送到其他链
- Local credit可能耗尽
- 即使池中有资金，也无法赎回

**缓解措施** (Stargate已实现):
- **Credit Messaging**: 可以从其他链"买回"credit
- **Deficit Offset**: Planner可以调整赤字偏移量
- **Fee Mechanism**: 动态费用激励平衡流动性

### 5.3 🟡 Medium: Treasurer权限

**问题**:
Treasurer可以:
1. 提取treasuryFee（所有累积的费用）
2. 提取误发的代币
3. 增加treasuryFee（从自己账户）

**风险**:
- 如果treasurer被攻破，累积的费用可能被盗
- 但无法盗取用户的本金（poolBalance受保护）

**保护机制** (StargatePool.sol:285-296):
```solidity
function recoverToken(address _token, address _to, uint256 _amount)
    public virtual override onlyCaller(treasurer) returns (uint256)
{
    if (_token == token) {
        // 只能提取超出poolBalance + treasuryFee的部分
        uint256 cap = _thisBalance() - _sd2ld(poolBalanceSD + treasuryFee);
        _amount = _amount > cap ? cap : _amount;
    }
    return super.recoverToken(_token, _to, _amount);
}
```

**评价**: ✅ 良好保护，无法盗取用户本金

### 5.4 🟡 Medium: FeeLib信任

**问题**:
Stargate完全信任feeLib计算费用。

**源码**: StargateBase.sol:538-554
```solidity
function _chargeFee(FeeParams memory _feeParams, uint64 _minAmountOutSD) internal returns (uint64 amountOutSD) {
    // 完全信任feeLib的返回值
    amountOutSD = IStargateFeeLib(feeLib).applyFee(_feeParams);

    // 如果feeLib返回0，用户什么都收不到！
    if (amountOutSD < _minAmountOutSD || amountOutSD == 0) revert Stargate_SlippageTooHigh();
}
```

**攻击场景**（如果feeLib恶意）:
```
1. 恶意feeLib.applyFee()返回amountOutSD = 0
2. 用户的minAmountLD检查: 如果设置为0也能通过
3. 用户发送1000 USDC，目标链收到0
4. 1000 USDC进入treasuryFee
```

**缓解措施**:
- FeeLib应该经过严格审计
- Owner更换feeLib应该有Timelock
- 用户应该设置合理的minAmountLD（slippage保护）

### 5.5 🟡 Medium: LayerZero依赖

**问题**:
Stargate的安全性**完全依赖**LayerZero。

**风险**:
- 如果LayerZero的DVN被攻破 → Stargate被攻破
- 如果LayerZero的Endpoint有漏洞 → Stargate受影响
- Stargate继承了LayerZero的所有风险（见Phase 1报告）

**缓解**:
- 使用高安全性的DVN配置
- 定期审查LayerZero的安全更新
- 考虑实施额外的应用层安全措施

### 5.6 🟢 Low: Bus模式DoS

**问题**:
Bus模式需要等待3-5笔交易凑齐。

**DoS场景**:
```
1. 攻击者发送1笔到Bus
2. Bus等待更多交易（2-5笔）
3. 攻击者不再发送
4. 其他用户的交易被延迟（等待Bus凑齐）
```

**实际影响**: 低，因为用户可以选择Taxi模式

### 5.7 🟢 Low: unreceivedTokens攻击

**问题**:
如果接收失败，代币被缓存到`unreceivedTokens`。

**潜在攻击**:
```
1. 攻击者发送到一个会revert的合约
2. 代币被缓存
3. 攻击者稍后调用retryReceiveToken()
4. 如果在此期间receiver合约逻辑改变，可能导致意外行为
```

**实际影响**: 低，因为receiver是攻击者自己

---

## 6. 与LayerZero集成安全性

### 6.1 OApp实现

Stargate实现了LayerZero的OApp标准：

**关键函数**:
```solidity
// 继承自StargateBase，实现ILayerZeroReceiver
function lzReceive(
    Origin calldata _origin,
    bytes32 _guid,
    bytes calldata _message,
    address _executor,
    bytes calldata _extraData
) external payable {
    // LayerZero EndpointV2会调用此函数
    // StargateBase路由到receiveTokenTaxi/Bus或receiveCredits
}
```

**集成要点**:
1. ✅ 正确实现`lzReceive()`
2. ✅ 使用`onlyCaller(tokenMessaging)`保护
3. ✅ 处理执行失败（缓存到unreceivedTokens）

### 6.2 DVN配置

**问题**: Stargate使用什么DVN配置？

**查询方法**:
```solidity
// 查询Stargate USDC Pool的DVN配置
endpoint.getReceiveLibrary(STARGATE_USDC_POOL, srcEid);
// 然后查询该库的UlnConfig
```

**推荐**: Stargate应该使用增强的DVN配置（见Phase 2报告）

---

## 7. 经济模型分析

### 7.1 费用结构

**费用计算** (由FeeLib决定):
```
amountOut = FeeLib.applyFee(FeeParams({
    sender: msg.sender,
    dstEid: dstEid,
    amountInSD: amountInSD,
    deficitSD: deficit,  // 池赤字
    isOFTPath: paths[dstEid].isOFTPath(),
    isTaxi: isTaxi
}))

Fee = amountInSD - amountOutSD (可能为负，即奖励)
```

**费用机制**:
- **Deficit-based**: 池赤字越大，费用越高（或奖励越低）
- **Path-based**: OFT路径和Pool路径费用不同
- **Mode-based**: Taxi和Bus模式可能费用不同

**目的**:
- 激励流动性平衡
- 赤字大的池收费高，吸引反向流动
- 盈余大的池给奖励，鼓励资金流出

### 7.2 LP奖励

**来源**:
1. **交易费用**: treasuryFee累积
2. **Protocol奖励**: 来自StargateStaking合约
3. **MultiRewarder**: 多种代币奖励

**分配**:
- LP代币持有者按比例分享
- 通过StargateStaking合约质押可获得额外奖励

### 7.3 经济安全性

**好的设计** ✅:
- 动态费用平衡流动性
- Reward上限为treasuryFee（不会过度支付）
- LP代币1:1铸造，公平

**潜在风险** 🟡:
- 如果FeeLib配置不当，可能导致池失衡
- 高费用可能降低竞争力
- 需要持续监控和调整

---

## 8. 审计结论

### 8.1 总体评分

| 维度 | 评分 | 说明 |
|-----|------|------|
| 代码质量 | ⭐⭐⭐⭐ | 清晰的模块化设计 |
| 安全机制 | ⭐⭐⭐ | 防重入、权限管理良好，但Owner权限大 |
| 经济模型 | ⭐⭐⭐⭐ | 动态费用机制设计合理 |
| LayerZero集成 | ⭐⭐⭐ | 正确实现，但继承LayerZero风险 |
| 文档完整性 | ⭐⭐⭐ | 代码注释好，缺少详细文档 |

**综合评分**: ⭐⭐⭐ (3.5/5)

### 8.2 关键发现总结

| 风险 | 等级 | 描述 |
|-----|------|------|
| Owner权限过大 | 🔴 Critical | 可更换所有角色，需多签+Timelock |
| Credit枯竭风险 | 🔴 Critical | 可能导致流动性锁定，已有缓解措施 |
| LayerZero依赖 | 🟡 Medium | 继承LayerZero所有风险 |
| Treasurer权限 | 🟡 Medium | 可提取费用，但无法盗取本金 |
| FeeLib信任 | 🟡 Medium | 完全信任费用计算，需审计 |

### 8.3 建议措施

#### 立即实施:
1. **Owner多签化**
   - 迁移到至少3/5多签
   - 关键操作实施7天Timelock

2. **增强DVN配置**
   - 使用至少3个required DVNs
   - 增加optional DVNs

3. **监控Credit状态**
   - 实时监控各池的credit余额
   - 及时通过Credit Messaging平衡

#### 中期改进:
1. **FeeLib审计**
   - 深度审计费用计算逻辑
   - 实施费用上限保护

2. **紧急响应计划**
   - 制定安全事件响应流程
   - 建立安全委员会

3. **保险覆盖**
   - 考虑购买DeFi保险
   - 建立安全储备金

#### 长期目标:
1. **去中心化治理**
   - 将Owner权限转移到DAO
   - 社区投票决定重大变更

2. **多链部署审计**
   - 审计所有链上的部署
   - 确保配置一致性

---

## 9. 与竞品对比

| 特性 | Stargate V2 | Hop Protocol | Across | Synapse |
|-----|-------------|--------------|--------|---------|
| 基础协议 | LayerZero V2 | Ethereum L2 | UMA Optimistic Oracle | LayerZero V1 |
| 流动性模型 | 统一池 | 分散池 | Relayer池 | 分散池 |
| 费用模型 | 动态Deficit-based | 固定 | 竞价 | 固定 |
| Bus模式 | ✅ (V2创新) | ❌ | ❌ | ❌ |
| Gas效率 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ |
| 安全性 | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ |
| TVL | 数亿美元 | 数千万美元 | 数千万美元 | 数千万美元 |

**Stargate优势**:
- Bus模式大幅降低Gas成本
- 统一流动性池，Capital效率高
- 基于LayerZero V2，支持120+链

**Stargate劣势**:
- 安全性依赖LayerZero
- Owner权限较大
- Credit机制复杂

---

**文档版本**: v1.0
**最后更新**: 2025-10-13
**审计状态**: Phase 2完成
