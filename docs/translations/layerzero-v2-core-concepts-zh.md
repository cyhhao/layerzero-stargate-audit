# LayerZero V2 核心概念 (中文)

> **文档来源**: 基于官方文档和代码分析整理
> **翻译时间**: 2025-10-14
> **原文参考**: https://docs.layerzero.network/v2
> **审计报告**: [LayerZero-Stargate-Security-Audit-Report.md](../../reports/LayerZero-Stargate-Security-Audit-Report.md)

---

## 目录

1. [协议概述](#协议概述)
2. [核心架构](#核心架构)
3. [关键组件](#关键组件)
4. [消息生命周期](#消息生命周期)
5. [安全模型](#安全模型)
6. [配置指南](#配置指南)

---

## 协议概述

### 什么是 LayerZero V2?

LayerZero V2 是一个**全链互操作协议** (Omnichain Interoperability Protocol)，使智能合约能够在不同区块链之间读写状态。

**核心特性**:
- 🔒 **不可变协议核心** - EndpointV2 部署后无法升级
- ⚙️ **可配置边缘合约** - MessageLib 可升级
- 🌐 **去中心化验证** - DVN 网络验证消息
- 🔗 **120+ 网络支持** - EVM, Solana, Move, TON 等
- 🎯 **统一代币标准** - OFT/ONFT 跨链代币

### 与传统跨链桥的区别

| 特性 | LayerZero V2 | 传统跨链桥 |
|------|-------------|-----------|
| **架构** | 不可变核心 + 可配置边缘 | 中心化验证 |
| **安全** | OApp 自选 DVN | 固定验证者 |
| **流动性** | 无需流动性 (消息协议) | 需要流动性池 |
| **灵活性** | 高度可定制 | 受限于桥设计 |
| **升级** | 边缘可升级，核心不变 | 通常整体升级 |

---

## 核心架构

### 整体架构图

```
┌─────────────────────────────────────────────────────────┐
│                LayerZero V2 架构                         │
└─────────────────────────────────────────────────────────┘

源链 (Chain A)                          目标链 (Chain B)
┌─────────────┐                         ┌─────────────┐
│   OApp      │                         │   OApp      │
│  (你的应用)  │                         │  (你的应用)  │
└──────┬──────┘                         └──────▲──────┘
       │                                       │
       │ 1. send()                        7. lzReceive()
       ▼                                       │
┌─────────────┐                         ┌─────────────┐
│ EndpointV2  │ ◀─── 不可变核心 ───▶     │ EndpointV2  │
│  eid: 30101 │                         │  eid: 30102 │
└──────┬──────┘                         └──────▲──────┘
       │                                       │
       │ 2. getSendLibrary()              6. verify()
       ▼                                       │
┌─────────────┐                         ┌─────────────┐
│ SendUln302  │ ◀─── 可升级边缘 ───▶     │ReceiveUln302│
│ (MessageLib)│                         │ (MessageLib)│
└──────┬──────┘                         └──────▲──────┘
       │                                       │
       │ 3. emit PacketSent            5. commitVerification()
       │                                       │
       └───────────────────┐   ┌──────────────┘
                           │   │
                    ┌──────▼───▼──────┐
                    │   DVN Network   │
                    │   (链下验证服务)  │
                    │                 │
                    │  4. verify() ×N │
                    └─────────────────┘
```

### 分层架构

**Layer 1: 不可变协议层**
- **EndpointV2**: 所有链上的统一入口点
- 部署后永不升级
- 保证长期稳定性和安全性

**Layer 2: 可配置边缘层**
- **MessageLib**: SendLib & ReceiveLib
- 可升级以支持新功能
- Grace Period 机制平滑过渡

**Layer 3: 去中心化验证层**
- **DVN Network**: 多个独立验证者
- OApp 可自选 DVN 组合
- M-of-N 验证模型

**Layer 4: 应用层**
- **OApp**: 基于 LayerZero 的跨链应用
- **OFT/ONFT**: 标准化跨链代币

---

## 关键组件

### 1. EndpointV2

**定义**: LayerZero V2 的不可变核心合约

**主要职责**:
```solidity
interface ILayerZeroEndpointV2 {
    // 发送跨链消息
    function send(
        MessagingParams calldata _params,
        address _refundAddress
    ) external payable returns (MessagingReceipt memory);

    // 验证跨链消息
    function verify(
        Origin calldata _origin,
        address _receiver,
        bytes32 _payloadHash
    ) external;

    // 执行跨链消息
    function lzReceive(
        Origin calldata _origin,
        address _receiver,
        bytes32 _guid,
        bytes calldata _message,
        bytes calldata _extraData
    ) external payable;
}
```

**关键特性**:
- ✅ **不可变性**: 部署后无法修改
- ✅ **多链统一**: 所有链使用相同地址 (通过 CREATE2)
- ✅ **权限管理**: Owner, Delegate 角色
- ✅ **消息路由**: 基于 eid (Endpoint ID)

**Ethereum 地址**: `0x1a44076050125825900e736c501f859c50fE728c`

### 2. MessageLib (消息库)

**定义**: 可升级的消息处理边缘合约

**类型**:

#### SendLib (发送库)
```solidity
// 主要实现: SendUln302
interface ISendLib {
    // 发送消息
    function send(
        Packet calldata _packet,
        bytes calldata _options,
        bool _payInLzToken
    ) external returns (MessagingFee memory, bytes memory);

    // 计算费用
    function quote(
        Packet calldata _packet,
        bytes calldata _options,
        bool _payInLzToken
    ) external view returns (MessagingFee memory);
}
```

**职责**:
- 💰 计算跨链费用 (DVN 费用 + Executor 费用)
- 📡 发出 PacketSent 事件
- ⚙️ 处理消息选项 (Options)

**Ethereum SendUln302**: `0xbB2Ea70C9E858123480642Cf96acbcCE1372dCe1`

#### ReceiveLib (接收库)
```solidity
// 主要实现: ReceiveUln302
interface IReceiveLib {
    // DVN 提交验证
    function verify(
        bytes calldata _packetHeader,
        bytes32 _payloadHash,
        uint64 _confirmations
    ) external;

    // 提交验证到 Endpoint
    function commitVerification(
        bytes calldata _packetHeader,
        bytes32 _payloadHash
    ) external;

    // 检查是否可验证
    function verifiable(
        UlnConfig memory _config,
        bytes32 _headerHash,
        bytes32 _payloadHash
    ) external view returns (bool);
}
```

**职责**:
- ✅ 接收 DVN 验证
- 🔍 检查验证条件 (M-of-N)
- 📝 提交验证到 Endpoint
- 🗑️ 清理存储回收 Gas

**Ethereum ReceiveUln302**: `0xc02Ab410f0734EFa3F14628780e6e695156024C2`

### 3. DVN (去中心化验证网络)

**定义**: 独立运行的链下服务，验证跨链消息真实性

**工作流程**:
```
1. 监听源链 PacketSent 事件
   ↓
2. 等待 confirmations 个区块
   ↓
3. 验证交易最终性 (Finality)
   ↓
4. 在目标链调用 ReceiveUln302.verify()
   ↓
5. 记录验证: hashLookup[header][payload][dvn] = ✓
```

**主要 DVN 提供商**:

| DVN | 地址 (Ethereum) | 类型 | 运营者 |
|-----|----------------|------|--------|
| **LayerZero Labs** | `0x589dedbd...` | Push | LayerZero 官方 |
| **Google Cloud** | `0xd56e4eab...` | Push | 运营者待确认⚠️ |
| **Chainlink** | `0x771d10d0...` | Push | Chainlink |
| **Nethermind** | `0xf4064220...` | Pull | Nethermind |
| **Polyhedra** | `0x8DDf05F9...` | Push | Polyhedra |

**DVN 类型**:

**Push DVN**:
- 链下服务主动提交验证
- Gas 效率高
- 需要维护链下基础设施

**Pull DVN** (lzRead):
- 通过链上读取验证
- 无需持续运行链下服务
- Gas 成本较高

### 4. UlnConfig (验证配置)

**定义**: OApp 的 DVN 验证配置

```solidity
struct UlnConfig {
    uint64 confirmations;           // 区块确认数
    uint8 requiredDVNCount;         // 必需 DVN 数量
    uint8 optionalDVNCount;         // 可选 DVN 数量
    uint8 optionalDVNThreshold;     // 可选 DVN 阈值
    address[] requiredDVNs;         // 必需 DVN 列表
    address[] optionalDVNs;         // 可选 DVN 列表
}
```

**验证逻辑**:
```
消息验证通过 ⟺
    (ALL required DVNs 已验证)
    AND
    (至少 optionalDVNThreshold 个 optional DVNs 已验证)
```

**配置示例**:

**默认配置** (Ethereum → BSC):
```solidity
UlnConfig({
    confirmations: 20,
    requiredDVNs: [
        0x589dedbd... (LayerZero Labs),
        0xd56e4eab... (Google Cloud)
    ],
    optionalDVNs: [],
    optionalDVNThreshold: 0
})
```
⚠️ **风险**: 仅 2 个 required DVNs，攻破任意 1 个即可伪造消息

**推荐配置**:
```solidity
UlnConfig({
    confirmations: 64,  // Ethereum 推荐值
    requiredDVNs: [
        LayerZero Labs,
        Chainlink,
        Nethermind
    ],  // 3 个独立实体
    optionalDVNs: [
        Google Cloud,
        Polyhedra,
        BCW,
        Axelar,
        Switchboard
    ],  // 5 个可选
    optionalDVNThreshold: 3  // 至少 3 个
})
```
✅ **安全性**: 需要攻破 3 个 required + 3 个 optional = 6 个 DVN

---

## 消息生命周期

### 完整流程

#### 阶段 1: 发送 (源链)

```solidity
// 用户/OApp 发起
OApp.crossChainCall() {
    EndpointV2.send{value: fee}(
        MessagingParams({
            dstEid: 30102,        // BSC
            receiver: bytes32(receiverAddress),
            message: abi.encode(data),
            options: options,
            payInLzToken: false
        }),
        refundAddress
    );
}

// EndpointV2 内部
function _send(...) internal returns (MessagingReceipt) {
    // 1. 生成 GUID
    bytes32 guid = keccak256(abi.encodePacked(
        nonce, srcEid, sender, dstEid, receiver
    ));

    // 2. 递增 nonce
    outboundNonce[sender][dstEid][receiver]++;

    // 3. 获取 SendLib
    address sendLib = getSendLibrary(sender, dstEid);

    // 4. 调用 SendLib
    (MessagingFee fee, bytes encodedPacket) =
        ISendLib(sendLib).send(packet, options, payInLzToken);

    return MessagingReceipt(guid, nonce, fee);
}

// SendUln302.send()
function send(...) external returns (MessagingFee, bytes) {
    // 1. 计算费用
    MessagingFee fee = _quote(packet, options);

    // 2. 收取费用
    _payFee(fee);

    // 3. 发出事件
    emit PacketSent(encodedPacket, options, sendLibrary);

    return (fee, encodedPacket);
}
```

**关键数据**:
- **GUID**: 消息的全局唯一标识符
- **Nonce**: 严格递增的序号
- **PacketSent 事件**: DVN 监听的信号

#### 阶段 2: 验证 (DVN 网络)

```python
# DVN 链下服务伪代码
class DVNService:
    def run(self):
        while True:
            # 1. 监听 PacketSent 事件
            event = listen_to_packet_sent_events()

            # 2. 解析 packet
            packet_header = parse_packet_header(event.encodedPacket)
            payload_hash = keccak256(event.encodedPacket.payload)

            # 3. 检查是否是我负责的路径
            if not self.should_verify(packet_header):
                continue

            # 4. 等待 confirmations 个区块
            wait_for_confirmations(self.config.confirmations)

            # 5. 检查 finality
            if not check_finality(packet_header.blockHash):
                continue  # 链重组，重新等待

            # 6. 幂等性检查
            if already_verified(packet_header, payload_hash):
                continue  # 已提交，跳过

            # 7. 在目标链提交验证
            target_chain.ReceiveUln302.verify(
                packet_header,
                payload_hash,
                self.config.confirmations
            )
```

**DVN 验证记录**:
```solidity
// ReceiveUln302 存储
mapping(bytes32 headerHash =>
    mapping(bytes32 payloadHash =>
        mapping(address dvn => Verification)
    )
) public hashLookup;

struct Verification {
    bool submitted;           // 是否已提交
    uint64 confirmations;     // 确认数
}
```

#### 阶段 3: 提交验证 (目标链)

```solidity
// 任何人都可以调用
ReceiveUln302.commitVerification(
    bytes packetHeader,
    bytes32 payloadHash
) external {
    // 1. 检查是否满足验证条件
    require(_checkVerifiable(
        config,
        keccak256(packetHeader),
        payloadHash
    ), "Not verifiable");

    // 2. 删除验证记录（回收 Gas）
    _verifyAndReclaimStorage(config, headerHash, payloadHash);

    // 3. 调用 Endpoint.verify()
    ILayerZeroEndpointV2(endpoint).verify(
        origin,
        receiver,
        payloadHash
    );
}

// EndpointV2.verify()
function verify(
    Origin calldata _origin,
    address _receiver,
    bytes32 _payloadHash
) external {
    // 1. 检查调用者是否是 receiver 的 ReceiveLib
    require(isValidReceiveLibrary(_receiver, _origin.srcEid, msg.sender));

    // 2. 检查路径是否可初始化
    require(_initializable(_origin, _receiver));

    // 3. 检查 nonce 是否可验证
    require(_verifiable(_origin, _receiver));

    // 4. 存储 payload hash
    inboundPayloadHash[_receiver][_origin.srcEid][_origin.sender][_origin.nonce]
        = _payloadHash;

    emit PayloadVerified(_receiver, _origin, _payloadHash);
}
```

**验证条件** (_checkVerifiable):
```solidity
function _checkVerifiable(
    UlnConfig memory _config,
    bytes32 _headerHash,
    bytes32 _payloadHash
) internal view returns (bool) {
    // 1. 所有 required DVNs 必须验证
    for (uint i = 0; i < _config.requiredDVNCount; i++) {
        if (!_verified(
            _config.requiredDVNs[i],
            _headerHash,
            _payloadHash,
            _config.confirmations
        )) {
            return false;
        }
    }

    // 2. 如果没有 optional DVNs，直接返回 true
    if (_config.optionalDVNCount == 0) {
        return true;
    }

    // 3. 至少 threshold 个 optional DVNs 验证
    uint verifiedCount = 0;
    for (uint i = 0; i < _config.optionalDVNCount; i++) {
        if (_verified(
            _config.optionalDVNs[i],
            _headerHash,
            _payloadHash,
            _config.confirmations
        )) {
            verifiedCount++;
            if (verifiedCount >= _config.optionalDVNThreshold) {
                return true;
            }
        }
    }

    return false;
}
```

#### 阶段 4: 执行 (目标链)

```solidity
// Executor 调用 (任何人)
EndpointV2.lzReceive{value: executionGas}(
    Origin({
        srcEid: 30101,
        sender: bytes32(senderAddress),
        nonce: 42
    }),
    receiverAddress,
    guid,
    message,
    extraData
) external payable {
    // 1. 清除 payload (防重入)
    _clearPayload(origin, receiver, guid, message);

    // 2. 调用 OApp 的 lzReceive
    ILayerZeroReceiver(receiver).lzReceive{value: msg.value}(
        origin,
        guid,
        message,
        msg.sender,  // executor address
        extraData
    );
}

function _clearPayload(
    Origin calldata _origin,
    address _receiver,
    bytes32 _guid,
    bytes calldata _message
) internal {
    // 1. 计算期望的 payload hash
    bytes32 expectedHash = keccak256(
        abi.encodePacked(_guid, _message)
    );

    // 2. 获取存储的 payload hash
    bytes32 storedHash = inboundPayloadHash[
        _receiver
    ][_origin.srcEid][_origin.sender][_origin.nonce];

    // 3. 验证匹配
    require(expectedHash == storedHash, "Invalid payload");

    // 4. 删除 payload hash
    delete inboundPayloadHash[_receiver][_origin.srcEid][_origin.sender][_origin.nonce];

    // 5. 更新 lazy nonce
    if (_origin.nonce == lazyInboundNonce[_receiver][_origin.srcEid][_origin.sender] + 1) {
        lazyInboundNonce[_receiver][_origin.srcEid][_origin.sender]++;
    }
}

// OApp 实现
contract MyOApp is ILayerZeroReceiver {
    function lzReceive(
        Origin calldata _origin,
        bytes32 _guid,
        bytes calldata _message,
        address _executor,
        bytes calldata _extraData
    ) external payable override {
        // 1. 验证调用者是 Endpoint
        require(msg.sender == endpoint, "Only endpoint");

        // 2. 验证源链和发送者
        require(
            trustedRemotes[_origin.srcEid] == _origin.sender,
            "Untrusted source"
        );

        // 3. 处理消息
        _handleMessage(_message);
    }
}
```

**安全要点**:
- ✅ **防重入**: 先 `_clearPayload`，再调用外部合约
- ✅ **Payload 验证**: 通过 hash 确保消息完整性
- ✅ **Nonce 管理**: 延迟加载，支持乱序验证但顺序执行

---

## 安全模型

### M-of-N 验证模型

**定义**: 需要 M 个验证者中的 N 个确认

**LayerZero 实现**:
```
验证通过 = (ALL required DVNs) AND (M-of-N optional DVNs)
```

**安全级别对比**:

| 配置 | 验证条件 | 攻击成本 | 安全等级 |
|------|---------|---------|---------|
| 1 required, 0 optional | 1-of-1 | 攻破 1 个 | ⭐ (最低) |
| 2 required, 0 optional | 2-of-2 (AND) | 攻破任意 1 个 | ⭐⭐ |
| 3 required, 0 optional | 3-of-3 (AND) | 攻破任意 1 个 | ⭐⭐ |
| 0 required, 3-of-5 optional | 3-of-5 (OR) | 攻破 3 个 | ⭐⭐⭐ |
| 3 required, 3-of-5 optional | (3-of-3) AND (3-of-5) | 攻破 1个required OR 3个optional | ⭐⭐⭐⭐ |
| 3 required, 5-of-7 optional | (3-of-3) AND (5-of-7) | 攻破 1个required OR 5个optional | ⭐⭐⭐⭐⭐ (最高) |

**推荐配置**:
```solidity
// 高安全级别 (推荐用于高价值应用)
UlnConfig({
    confirmations: 64,
    requiredDVNs: [LZ Labs, Chainlink, Nethermind],  // 3个
    optionalDVNs: [Google, Polyhedra, BCW, Axelar, Switchboard, Horizen, StableOrbit],  // 7个
    optionalDVNThreshold: 5  // 需要5个
})

// 攻击成本: 需要攻破 (3个required中的1个) OR (7个optional中的5个)
```

### 区块确认数 (Confirmations)

**作用**: 防止链重组攻击

**推荐值**:

| 链 | 推荐值 | 理由 |
|----|-------|------|
| **Ethereum** | 64 | Casper FFG finality (~13分钟) |
| **BSC** | 15 | 快速但中心化 (~45秒) |
| **Polygon** | 128 | 高重组风险 (~4分钟) |
| **Arbitrum** | 20 | L2 相对安全 |
| **Optimism** | 20 | 同 Arbitrum |
| **Avalanche** | 10 | 快速最终性 |

**链重组攻击场景**:
```
t0: 用户在 Ethereum 发送消息
t1: DVN 在 confirmations=10 时验证
t2: 目标链执行消息
t3: Ethereum 发生 51% 攻击，回滚到 t0
t4: Ethereum 上的交易被回滚，但目标链已执行

结果: 目标链执行了不存在的消息
```

**防御**: 使用足够的 confirmations 确保最终性

### 信任假设

**LayerZero V2 的信任模型**:

1. **Endpoint 不可变性** ✅
   - EndpointV2 部署后不可修改
   - 长期可信

2. **MessageLib 可升级** ⚠️
   - SendLib/ReceiveLib 可升级
   - 依赖 Owner 和 grace period

3. **DVN 诚实性** ⚠️
   - 完全依赖 DVN 网络
   - 无经济惩罚 (Slash) 机制
   - 运营者独立性需验证

4. **Executor 可信** ⚠️
   - 任何人可执行
   - OApp 需验证 executor

5. **Owner 中心化** ⚠️
   - Owner 权限过大
   - 需要治理透明化

**风险等级**:
```
完整安全 = 代码安全 (⭐⭐⭐⭐⭐) × 治理安全 (⭐⭐) × DVN安全 (⭐⭐)
         = ⭐⭐⭐ (2.5/5)
```

---

## 配置指南

### OApp 开发者配置

#### 1. 设置 Delegate (可选)

```solidity
// 使用多签作为 delegate
address multisig = 0x...;
endpointV2.setDelegate(multisig);
```

⚠️ **警告**: Delegate 拥有与 OApp 相同的配置权限！

#### 2. 配置 DVN

```solidity
// 推荐配置
address[] memory requiredDVNs = new address[](3);
requiredDVNs[0] = 0x589dedbd...; // LayerZero Labs
requiredDVNs[1] = 0x771d10d0...; // Chainlink
requiredDVNs[2] = 0xf4064220...; // Nethermind

address[] memory optionalDVNs = new address[](5);
optionalDVNs[0] = 0xd56e4eab...; // Google Cloud
optionalDVNs[1] = 0x8DDf05F9...; // Polyhedra
optionalDVNs[2] = 0x...; // BCW
optionalDVNs[3] = 0x...; // Axelar
optionalDVNs[4] = 0x...; // Switchboard

SetConfigParam[] memory params = new SetConfigParam[](1);
params[0] = SetConfigParam({
    eid: 30102,  // BSC
    configType: uint32(2),  // ULN_CONFIG_TYPE
    config: abi.encode(
        UlnConfig({
            confirmations: 64,
            requiredDVNCount: 3,
            optionalDVNCount: 5,
            optionalDVNThreshold: 3,
            requiredDVNs: requiredDVNs,
            optionalDVNs: optionalDVNs
        })
    )
});

endpointV2.setConfig(address(oapp), receiveLib302, params);
```

#### 3. 实现 allowInitializePath

```solidity
function allowInitializePath(Origin calldata _origin)
    external
    view
    returns (bool)
{
    // 只允许受信任的源链和发送者
    return trustedRemotes[_origin.srcEid] == _origin.sender;
}
```

❌ **错误示例**:
```solidity
// 不安全! 允许任何路径
function allowInitializePath(Origin calldata) external pure returns (bool) {
    return true;
}
```

#### 4. 验证 Executor

```solidity
function lzReceive(
    Origin calldata _origin,
    bytes32 _guid,
    bytes calldata _message,
    address _executor,
    bytes calldata _extraData
) external payable override {
    // 1. 验证 endpoint
    require(msg.sender == endpoint);

    // 2. 验证源
    require(trustedRemotes[_origin.srcEid] == _origin.sender);

    // 3. 可选: 验证 executor
    if (!trustedExecutors[_executor]) {
        // 限制 extraData 或其他防御措施
    }

    _handleMessage(_message);
}
```

### 查询当前配置

```bash
# 查询 SendLib
cast call 0x1a44076050125825900e736c501f859c50fE728c \
  "getSendLibrary(address,uint32)(address)" \
  <your-oapp-address> \
  30102 \
  --rpc-url https://eth.llamarpc.com

# 查询 ReceiveLib
cast call 0x1a44076050125825900e736c501f859c50fE728c \
  "getReceiveLibrary(address,uint32)(address,bool)" \
  <your-oapp-address> \
  30102 \
  --rpc-url https://eth.llamarpc.com

# 查询 UlnConfig
cast call 0xc02Ab410f0734EFa3F14628780e6e695156024C2 \
  "getUlnConfig(address,uint32)(uint64,uint8,uint8,uint8,address[],address[])" \
  <your-oapp-address> \
  30102 \
  --rpc-url https://eth.llamarpc.com
```

---

## 最佳实践

### ✅ DO (应该做)

1. **使用多个独立 DVN**
   - 至少 3 个 required DVNs
   - 来自不同运营者

2. **设置 optional DVNs**
   - 提供冗余和额外安全层
   - Threshold 设为 3-5 个

3. **足够的 confirmations**
   - 根据链的安全特性设置
   - Ethereum: 64, BSC: 15, Polygon: 128

4. **验证消息来源**
   - 实现 `allowInitializePath`
   - 白名单受信任的源

5. **使用多签 Delegate**
   - 不要使用 EOA
   - 至少 3/5 多签

6. **定期审查配置**
   - 检查 DVN 列表
   - 更新 confirmations

### ❌ DON'T (不应该做)

1. **依赖默认配置**
   - 默认配置可能不够安全
   - 特别是高价值应用

2. **使用单个 required DVN**
   - 单点失败风险
   - 至少 3 个

3. **忽略 confirmations**
   - 链重组风险
   - 根据链特性设置

4. **盲目信任 extraData**
   - Executor 可传递任意 extraData
   - 验证或限制使用

5. **EOA 作为 Delegate**
   - 私钥泄露风险
   - 使用多签合约

6. **忽略安全更新**
   - 关注协议公告
   - 及时更新配置

---

## 相关资源

### 官方文档
- 🔗 [LayerZero V2 Docs](https://docs.layerzero.network/v2)
- 🔗 [GitHub](https://github.com/LayerZero-Labs/LayerZero-v2)
- 🔗 [DVN 地址](https://docs.layerzero.network/v2/developers/evm/technical-reference/dvn-addresses)

### 审计报告
- 📄 [完整审计报告](../../reports/LayerZero-Stargate-Security-Audit-Report.md)
- 📄 [DVN 链下分析](../../analysis/03-DVN-OffChain-Analysis.md)
- 🔗 [Paladin 审计](https://paladinsec.co/projects/layerzero/)

### 工具
- 📖 [术语表](../GLOSSARY.md)
- 🔗 [合约地址](../DOCUMENTATION_INDEX.md)

---

**文档版本**: v1.0
**最后更新**: 2025-10-14
**维护者**: Claude (AI Security Researcher)
**反馈**: https://github.com/cyhhao/layerzero-stargate-audit/issues
