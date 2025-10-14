# LayerZero V2 OApp 开发指南

> **文档来源**: 基于官方文档和代码分析整理
> **编写时间**: 2025-10-14
> **官方文档**: https://docs.layerzero.network/v2
> **GitHub**: https://github.com/LayerZero-Labs/LayerZero-v2
> **维护者**: Claude (AI Security Researcher)

---

## 目录

1. [OApp 核心概念](#oapp-核心概念)
2. [OApp 基础开发](#oapp-基础开发)
3. [OFT (跨链同质化代币)](#oft-跨链同质化代币)
4. [ONFT (跨链非同质化代币)](#onft-跨链非同质化代币)
5. [高级功能](#高级功能)
6. [安全最佳实践](#安全最佳实践)
7. [常见陷阱与解决方案](#常见陷阱与解决方案)
8. [部署与配置](#部署与配置)

---

## OApp 核心概念

### 什么是 OApp？

**OApp (Omnichain Application)** 是基于 LayerZero V2 构建的**跨链应用**，能够在不同区块链之间发送和接收任意消息。

### 核心特性

```
┌─────────────────────────────────────────────────────────┐
│                    OApp 架构                             │
└─────────────────────────────────────────────────────────┘

       链 A                                    链 B
┌──────────────────┐                  ┌──────────────────┐
│   MyOApp         │                  │   MyOApp         │
│  ┌────────────┐  │                  │  ┌────────────┐  │
│  │ lzSend()   ├──┼─────────────────▶│──┤ lzReceive()│  │
│  └────────────┘  │                  │  └────────────┘  │
│  ┌────────────┐  │                  │  ┌────────────┐  │
│  │lzReceive() │◀─┼──────────────────┼──┤ lzSend()   │  │
│  └────────────┘  │                  │  └────────────┘  │
└──────┬───────────┘                  └───────┬──────────┘
       │                                      │
       ▼                                      ▼
  EndpointV2                            EndpointV2
```

### 统一的开发模型

所有 OApp 只需要实现两个核心方法：

```solidity
// 1. 发送消息
_lzSend(_dstEid, _message, _options, _fee, _refundAddress)

// 2. 接收消息
_lzReceive(_origin, _guid, _message, _executor, _extraData)
```

---

## OApp 基础开发

### 1. 环境准备

#### 创建新项目

```bash
# 使用官方脚手架
npx create-lz-oapp@latest

# 选择 OApp 作为起始模板
? What is your project type? › OApp
```

#### 现有项目集成

```bash
# 安装依赖
npm install @layerzerolabs/oapp-evm
npm install @layerzerolabs/lz-evm-protocol-v2
```

### 2. 基础 OApp 合约

#### 最简单的 OApp 实现

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import { OApp, Origin, MessagingFee, MessagingReceipt } from "@layerzerolabs/oapp-evm/contracts/oapp/OApp.sol";
import { Ownable } from "@openzeppelin/contracts/access/Ownable.sol";

contract SimpleOApp is OApp {
    // 存储接收到的消息
    string public lastMessage;
    uint256 public messageCount;

    // 事件
    event MessageSent(uint32 indexed dstEid, string message);
    event MessageReceived(uint32 indexed srcEid, string message);

    /**
     * @dev 构造函数
     * @param _endpoint LayerZero EndpointV2 地址
     * @param _owner 合约所有者
     */
    constructor(
        address _endpoint,
        address _owner
    ) OApp(_endpoint, _owner) Ownable(_owner) {}

    /**
     * @notice 发送跨链消息
     * @param _dstEid 目标链的 Endpoint ID
     * @param _message 要发送的消息
     * @param _options 执行选项（gas 限制等）
     */
    function sendMessage(
        uint32 _dstEid,
        string memory _message,
        bytes calldata _options
    ) external payable {
        // 编码消息
        bytes memory payload = abi.encode(_message);

        // 发送消息
        _lzSend(
            _dstEid,
            payload,
            _options,
            MessagingFee(msg.value, 0),
            payable(msg.sender)
        );

        emit MessageSent(_dstEid, _message);
    }

    /**
     * @notice 接收跨链消息 (内部函数，由 Endpoint 调用)
     * @param _origin 消息来源信息
     * @param _message 接收到的消息
     */
    function _lzReceive(
        Origin calldata _origin,
        bytes32 /*_guid*/,
        bytes calldata _message,
        address /*_executor*/,
        bytes calldata /*_extraData*/
    ) internal override {
        // 解码消息
        string memory message = abi.decode(_message, (string));

        // 存储消息
        lastMessage = message;
        messageCount++;

        emit MessageReceived(_origin.srcEid, message);
    }

    /**
     * @notice 估算跨链消息费用
     * @param _dstEid 目标链的 Endpoint ID
     * @param _message 要发送的消息
     * @param _options 执行选项
     * @return fee 需要支付的费用
     */
    function quote(
        uint32 _dstEid,
        string memory _message,
        bytes calldata _options
    ) public view returns (MessagingFee memory fee) {
        bytes memory payload = abi.encode(_message);
        return _quote(_dstEid, payload, _options, false);
    }
}
```

### 3. 核心组件详解

#### OAppCore - 基础组件

```solidity
abstract contract OAppCore {
    // EndpointV2 实例
    ILayerZeroEndpointV2 public immutable endpoint;

    // 对等合约地址映射
    mapping(uint32 eid => bytes32 peer) public peers;

    /**
     * @notice 设置对等合约地址
     * @param _eid 目标链的 Endpoint ID
     * @param _peer 对等合约地址（bytes32 格式）
     */
    function setPeer(uint32 _eid, bytes32 _peer) public onlyOwner {
        peers[_eid] = _peer;
        emit PeerSet(_eid, _peer);
    }

    /**
     * @notice 设置 Delegate 地址
     * @param _delegate 可以配置 OApp 的委托地址
     */
    function setDelegate(address _delegate) public onlyOwner {
        endpoint.setDelegate(_delegate);
    }
}
```

#### OAppSender - 发送功能

```solidity
abstract contract OAppSender is OAppCore {
    /**
     * @dev 内部发送函数
     * @param _dstEid 目标链 ID
     * @param _message 消息内容
     * @param _options 执行选项
     * @param _fee 消息费用
     * @param _refundAddress 退款地址
     * @return receipt 消息回执
     */
    function _lzSend(
        uint32 _dstEid,
        bytes memory _message,
        bytes memory _options,
        MessagingFee memory _fee,
        address payable _refundAddress
    ) internal virtual returns (MessagingReceipt memory receipt) {
        // 获取对等合约地址
        bytes32 peer = _getPeerOrRevert(_dstEid);

        // 调用 EndpointV2 发送
        return endpoint.send{value: _fee.nativeFee}(
            MessagingParams({
                dstEid: _dstEid,
                receiver: peer,
                message: _message,
                options: _options,
                payInLzToken: _fee.lzTokenFee > 0
            }),
            _refundAddress
        );
    }

    /**
     * @dev 估算费用
     */
    function _quote(
        uint32 _dstEid,
        bytes memory _message,
        bytes memory _options,
        bool _payInLzToken
    ) internal view virtual returns (MessagingFee memory fee) {
        bytes32 peer = _getPeerOrRevert(_dstEid);
        return endpoint.quote(
            MessagingParams({
                dstEid: _dstEid,
                receiver: peer,
                message: _message,
                options: _options,
                payInLzToken: _payInLzToken
            }),
            address(this)
        );
    }
}
```

#### OAppReceiver - 接收功能

```solidity
abstract contract OAppReceiver is OAppCore {
    /**
     * @notice 接收跨链消息 (必须实现)
     * @param _origin 消息来源信息
     * @param _guid 全局唯一标识符
     * @param _message 消息内容
     * @param _executor 执行器地址
     * @param _extraData 额外数据
     */
    function _lzReceive(
        Origin calldata _origin,
        bytes32 _guid,
        bytes calldata _message,
        address _executor,
        bytes calldata _extraData
    ) internal virtual;

    /**
     * @notice LayerZero Endpoint 调用此函数
     */
    function lzReceive(
        Origin calldata _origin,
        bytes32 _guid,
        bytes calldata _message,
        address _executor,
        bytes calldata _extraData
    ) external payable virtual {
        // 验证调用者是 Endpoint
        require(msg.sender == address(endpoint), "Only endpoint");

        // 验证消息来源
        require(_getPeerOrRevert(_origin.srcEid) == _origin.sender, "Invalid sender");

        // 处理消息
        _lzReceive(_origin, _guid, _message, _executor, _extraData);
    }

    /**
     * @notice 允许路径初始化（重要的安全检查）
     * @param _origin 消息来源
     * @return 是否允许该路径
     */
    function allowInitializePath(Origin calldata _origin)
        public
        view
        virtual
        returns (bool)
    {
        // 只允许已设置的 peer
        return peers[_origin.srcEid] == _origin.sender;
    }
}
```

### 4. 完整示例：跨链计数器

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import { OApp, Origin, MessagingFee } from "@layerzerolabs/oapp-evm/contracts/oapp/OApp.sol";
import { Ownable } from "@openzeppelin/contracts/access/Ownable.sol";
import { OptionsBuilder } from "@layerzerolabs/oapp-evm/contracts/oapp/libs/OptionsBuilder.sol";

/**
 * @title CrossChainCounter
 * @dev 跨链计数器：在任意链上递增计数器，其他链同步更新
 */
contract CrossChainCounter is OApp {
    using OptionsBuilder for bytes;

    // 全局计数器
    uint256 public counter;

    // 每条链的入站和出站计数
    mapping(uint32 => uint256) public inboundCount;
    mapping(uint32 => uint256) public outboundCount;

    // 事件
    event CounterIncremented(uint32 indexed srcEid, uint256 newCount);
    event CounterSynced(uint32 indexed dstEid, uint256 count);

    constructor(
        address _endpoint,
        address _owner
    ) OApp(_endpoint, _owner) Ownable(_owner) {}

    /**
     * @notice 递增计数器并同步到其他链
     * @param _dstEids 目标链 ID 列表
     */
    function incrementAndSync(uint32[] calldata _dstEids)
        external
        payable
    {
        // 递增本地计数器
        counter++;

        // 准备消息
        bytes memory message = abi.encode(counter);

        // 构建选项：200k gas，0 msg.value
        bytes memory options = OptionsBuilder
            .newOptions()
            .addExecutorLzReceiveOption(200000, 0);

        // 向每条目标链发送更新
        uint256 totalFee = 0;
        for (uint256 i = 0; i < _dstEids.length; i++) {
            uint32 dstEid = _dstEids[i];

            // 估算费用
            MessagingFee memory fee = _quote(dstEid, message, options, false);
            require(msg.value >= totalFee + fee.nativeFee, "Insufficient fee");

            // 发送消息
            _lzSend(
                dstEid,
                message,
                options,
                MessagingFee(fee.nativeFee, 0),
                payable(msg.sender)
            );

            outboundCount[dstEid]++;
            totalFee += fee.nativeFee;

            emit CounterSynced(dstEid, counter);
        }

        // 退还多余的费用
        if (msg.value > totalFee) {
            payable(msg.sender).transfer(msg.value - totalFee);
        }
    }

    /**
     * @notice 接收来自其他链的计数器更新
     */
    function _lzReceive(
        Origin calldata _origin,
        bytes32 /*_guid*/,
        bytes calldata _message,
        address /*_executor*/,
        bytes calldata /*_extraData*/
    ) internal override {
        // 解码接收到的计数
        uint256 receivedCount = abi.decode(_message, (uint256));

        // 更新本地计数器（取最大值）
        if (receivedCount > counter) {
            counter = receivedCount;
        }

        inboundCount[_origin.srcEid]++;

        emit CounterIncremented(_origin.srcEid, counter);
    }

    /**
     * @notice 估算批量同步费用
     */
    function quoteBatchSync(uint32[] calldata _dstEids)
        external
        view
        returns (uint256 totalFee)
    {
        bytes memory message = abi.encode(counter);
        bytes memory options = OptionsBuilder
            .newOptions()
            .addExecutorLzReceiveOption(200000, 0);

        for (uint256 i = 0; i < _dstEids.length; i++) {
            MessagingFee memory fee = _quote(_dstEids[i], message, options, false);
            totalFee += fee.nativeFee;
        }
    }
}
```

---

## OFT (跨链同质化代币)

### 1. OFT 核心概念

**OFT (Omnichain Fungible Token)** 是 LayerZero 的**跨链同质化代币标准**，实现代币在多条链之间无缝转移。

#### 工作原理

```
┌──────────────────────────────────────────────────────┐
│            OFT Burn & Mint 机制                       │
└──────────────────────────────────────────────────────┘

       源链 (Chain A)                目标链 (Chain B)
┌─────────────────────┐          ┌─────────────────────┐
│  用户: 100 USDC     │          │  用户: 0 USDC       │
└──────────┬──────────┘          └──────────▲──────────┘
           │                                 │
           │ send(100 USDC)                 │ mint(100 USDC)
           ▼                                 │
┌─────────────────────┐          ┌─────────────────────┐
│   OFT Contract      │          │   OFT Contract      │
│   ┌──────────┐      │          │   ┌──────────┐      │
│   │_debit()  │──────┼─────────▶│───│_credit() │      │
│   │burn(100) │      │ LayerZero│   │mint(100) │      │
│   └──────────┘      │          │   └──────────┘      │
│                     │          │                     │
│  Total: -100        │          │  Total: +100        │
└─────────────────────┘          └─────────────────────┘

全局总供应量保持不变: ΣChain(supply) = constant
```

### 2. OFT vs OFTAdapter

| 特性 | OFT | OFTAdapter |
|------|-----|------------|
| **用途** | 新创建的代币 | 现有的 ERC20 代币 |
| **机制** | Burn & Mint | Lock & Unlock (在源链) <br> Mint & Burn (在其他链) |
| **合约关系** | OFT 本身就是 ERC20 | OFT Adapter 包装现有 ERC20 |
| **审批** | 不需要 | 需要 approve() |
| **部署** | 所有链都部署 OFT | 源链部署 Adapter，其他链部署 OFT |

### 3. 标准 OFT 实现

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import { OFT } from "@layerzerolabs/oft-evm/contracts/OFT.sol";
import { Ownable } from "@openzeppelin/contracts/access/Ownable.sol";

/**
 * @title MyOFT
 * @dev 标准 OFT 实现：新代币
 */
contract MyOFT is OFT {
    /**
     * @param _name 代币名称
     * @param _symbol 代币符号
     * @param _lzEndpoint LayerZero Endpoint 地址
     * @param _delegate 委托地址
     */
    constructor(
        string memory _name,
        string memory _symbol,
        address _lzEndpoint,
        address _delegate
    ) OFT(_name, _symbol, _lzEndpoint, _delegate) Ownable(_delegate) {
        // 可选：在部署链上铸造初始供应量
        // _mint(msg.sender, 1_000_000 * 10**decimals());
    }
}
```

### 4. OFTAdapter 实现

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import { OFTAdapter } from "@layerzerolabs/oft-evm/contracts/OFTAdapter.sol";
import { Ownable } from "@openzeppelin/contracts/access/Ownable.sol";

/**
 * @title MyOFTAdapter
 * @dev 适配现有 ERC20 代币为 OFT
 * @dev ⚠️ 每个全局网格只能有一个 Adapter!
 */
contract MyOFTAdapter is OFTAdapter {
    /**
     * @param _token 现有的 ERC20 代币地址
     * @param _lzEndpoint LayerZero Endpoint 地址
     * @param _delegate 委托地址
     */
    constructor(
        address _token,
        address _lzEndpoint,
        address _delegate
    ) OFTAdapter(_token, _lzEndpoint, _delegate) Ownable(_delegate) {}
}
```

### 5. OFT 核心机制

#### Decimal Conversion (精度转换)

OFT 通过 **Shared Decimals (SD)** 和 **Local Decimals (LD)** 解决不同链上代币精度不一致的问题。

```solidity
/**
 * @dev 精度转换示例
 *
 * 场景：
 * - Chain A: USDC 有 6 位小数
 * - Chain B: USDC 有 18 位小数
 * - Shared Decimals: 6 (最小公共精度)
 *
 * 发送 1.2345 USDC 从 Chain A 到 Chain B:
 *
 * Chain A:
 * - Local Amount: 1.2345 * 10^6 = 1,234,500 (LD)
 * - Remove Dust: 1,234,500 / 10^0 * 10^0 = 1,234,500 (LD)
 * - Convert to SD: 1,234,500 / 10^(6-6) = 1,234,500 (SD)
 *
 * Chain B:
 * - Received SD: 1,234,500
 * - Convert to LD: 1,234,500 * 10^(18-6) = 1,234,500,000,000,000,000 (LD)
 * - Final Amount: 1.2345 * 10^18
 */

abstract contract OFTCore {
    // 精度转换率
    uint256 public immutable decimalConversionRate;

    constructor(uint8 _localDecimals, address _endpoint, address _delegate) {
        // decimalConversionRate = 10^(localDecimals - sharedDecimals)
        decimalConversionRate = 10 ** (_localDecimals - sharedDecimals());
    }

    /**
     * @dev 默认的 sharedDecimals 是 6
     * @dev 最大可表示: uint64.max / 10^6 = 18,446,744,073,709 tokens
     * @dev 如果总供应量超过此值，需要减少 sharedDecimals
     */
    function sharedDecimals() public view virtual returns (uint8) {
        return 6;
    }

    /**
     * @dev LD -> SD
     */
    function _toSD(uint256 _amountLD) internal view returns (uint64 amountSD) {
        return uint64(_amountLD / decimalConversionRate);
    }

    /**
     * @dev SD -> LD
     */
    function _toLD(uint64 _amountSD) internal view returns (uint256 amountLD) {
        return _amountSD * decimalConversionRate;
    }

    /**
     * @dev 移除无法表示的精度损失（dust）
     */
    function _removeDust(uint256 _amountLD) internal view returns (uint256) {
        return (_amountLD / decimalConversionRate) * decimalConversionRate;
    }
}
```

#### Debit & Credit 机制

```solidity
abstract contract OFT is OFTCore, ERC20 {
    /**
     * @dev 发送时扣除（燃烧）代币
     */
    function _debit(
        address _from,
        uint256 _amountLD,
        uint256 _minAmountLD,
        uint32 _dstEid
    ) internal virtual override returns (
        uint256 amountSentLD,
        uint256 amountReceivedLD
    ) {
        // 1. 移除精度损失
        amountSentLD = _removeDust(_amountLD);

        // 2. 检查滑点
        require(amountSentLD >= _minAmountLD, "Slippage exceeded");

        // 3. 燃烧代币
        _burn(_from, amountSentLD);

        // 4. 默认实现：发送量 = 接收量
        amountReceivedLD = amountSentLD;
    }

    /**
     * @dev 接收时增加（铸造）代币
     */
    function _credit(
        address _to,
        uint256 _amountLD,
        uint32 /*_srcEid*/
    ) internal virtual override returns (uint256 amountReceivedLD) {
        // 防止铸造到零地址
        if (_to == address(0x0)) _to = address(0xdead);

        // 铸造代币
        _mint(_to, _amountLD);

        return _amountLD;
    }
}
```

### 6. 发送 OFT 代币

```solidity
/**
 * @dev 用户发送 OFT 的完整流程
 */
contract OFTUsageExample {
    using OptionsBuilder for bytes;

    IOFT public oft;

    /**
     * @notice 发送 OFT 到另一条链
     * @param _dstEid 目标链 ID
     * @param _toAddress 接收者地址
     * @param _amountLD 发送数量（本地精度）
     * @param _minAmountLD 最小接收数量（滑点保护）
     */
    function sendOFT(
        uint32 _dstEid,
        address _toAddress,
        uint256 _amountLD,
        uint256 _minAmountLD
    ) external payable {
        // 1. 构建发送参数
        bytes32 to = bytes32(uint256(uint160(_toAddress)));

        SendParam memory sendParam = SendParam({
            dstEid: _dstEid,
            to: to,
            amountLD: _amountLD,
            minAmountLD: _minAmountLD,
            extraOptions: OptionsBuilder.newOptions().addExecutorLzReceiveOption(200000, 0),
            composeMsg: "",  // 不使用 compose
            oftCmd: ""       // 不使用 OFT 命令
        });

        // 2. 估算费用
        MessagingFee memory fee = oft.quoteSend(sendParam, false);
        require(msg.value >= fee.nativeFee, "Insufficient fee");

        // 3. 发送代币
        oft.send{value: fee.nativeFee}(
            sendParam,
            fee,
            payable(msg.sender)  // 退款地址
        );
    }
}
```

### 7. 高级 OFT：带手续费

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import { OFT } from "@layerzerolabs/oft-evm/contracts/OFT.sol";

/**
 * @title FeeOFT
 * @dev 带转账手续费的 OFT（10% 手续费示例）
 */
contract FeeOFT is OFT {
    uint256 public constant FEE_BP = 1000; // 10% = 1000 / 10000
    address public feeCollector;

    event FeeCollected(address indexed from, uint256 amount);

    constructor(
        string memory _name,
        string memory _symbol,
        address _lzEndpoint,
        address _delegate,
        address _feeCollector
    ) OFT(_name, _symbol, _lzEndpoint, _delegate) {
        feeCollector = _feeCollector;
    }

    /**
     * @dev 覆盖 _debit 以实现手续费逻辑
     */
    function _debit(
        address _from,
        uint256 _amountLD,
        uint256 _minAmountLD,
        uint32 _dstEid
    ) internal override returns (
        uint256 amountSentLD,
        uint256 amountReceivedLD
    ) {
        // 1. 移除精度损失
        amountSentLD = _removeDust(_amountLD);

        // 2. 计算手续费
        uint256 fee = (amountSentLD * FEE_BP) / 10000;
        amountReceivedLD = amountSentLD - fee;

        // 3. 检查滑点
        require(amountReceivedLD >= _minAmountLD, "Slippage exceeded");

        // 4. 燃烧代币（发送数量）
        _burn(_from, amountSentLD);

        // 5. 铸造手续费给收集者
        _mint(feeCollector, fee);

        emit FeeCollected(_from, fee);

        // 6. 返回：发送了多少，对方将收到多少
        return (amountSentLD, amountReceivedLD);
    }
}
```

### 8. OFT 部署流程

```typescript
// Hardhat 部署脚本示例
import { ethers } from "hardhat";

async function deployOFT() {
    // 1. 部署参数
    const name = "My Omnichain Token";
    const symbol = "MOT";

    // LayerZero Endpoint 地址
    const endpoints = {
        ethereum: "0x1a44076050125825900e736c501f859c50fE728c",
        bsc: "0x1a44076050125825900e736c501f859c50fE728c",
        avalanche: "0x1a44076050125825900e736c501f859c50fE728c"
    };

    const [deployer] = await ethers.getSigners();

    // 2. 在每条链上部署 OFT
    const OFT = await ethers.getContractFactory("MyOFT");

    // Ethereum
    const oftEth = await OFT.deploy(
        name,
        symbol,
        endpoints.ethereum,
        deployer.address
    );
    console.log("OFT deployed on Ethereum:", oftEth.address);

    // BSC
    const oftBsc = await OFT.deploy(
        name,
        symbol,
        endpoints.bsc,
        deployer.address
    );
    console.log("OFT deployed on BSC:", oftBsc.address);

    // 3. 设置 Peer 关系
    // Ethereum -> BSC
    await oftEth.setPeer(
        30102, // BSC eid
        ethers.utils.zeroPad(oftBsc.address, 32)
    );

    // BSC -> Ethereum
    await oftBsc.setPeer(
        30101, // Ethereum eid
        ethers.utils.zeroPad(oftEth.address, 32)
    );

    console.log("Peers configured successfully!");

    // 4. 配置 DVN（可选，使用自定义安全配置）
    // 参考: https://docs.layerzero.network/v2/developers/evm/configuration
}

deployOFT().catch(console.error);
```

---

## ONFT (跨链非同质化代币)

### 1. ONFT 核心概念

**ONFT (Omnichain Non-Fungible Token)** 是 LayerZero 的**跨链 NFT 标准**，支持 ERC721 和 ERC1155。

#### ONFT vs ONFT Adapter

```
┌────────────────────────────────────────────────────────┐
│              ONFT: Burn & Mint                         │
└────────────────────────────────────────────────────────┘

用例: 新创建的 NFT 集合，原生支持跨链

       Chain A                          Chain B
┌─────────────────┐              ┌─────────────────┐
│  ONFT #123      │    send()    │  (无 #123)      │
│  burn(#123) ────┼──────────────▶  mint(#123)     │
│                 │   LayerZero  │                 │
└─────────────────┘              └─────────────────┘

┌────────────────────────────────────────────────────────┐
│          ONFT Adapter: Lock & Unlock                   │
└────────────────────────────────────────────────────────┘

用例: 现有的 NFT 集合，扩展跨链能力

     Chain A (源链)                   Chain B
┌─────────────────┐              ┌─────────────────┐
│ Original NFT    │              │  Proxy ONFT     │
│   #123          │    send()    │  (无 #123)      │
│ lock(#123)──────┼──────────────▶  mint(#123)     │
│                 │   LayerZero  │                 │
│ unlock(#123)◀───┼──────────────── burn(#123)     │
│   #123          │   receive()  │                 │
└─────────────────┘              └─────────────────┘
```

### 2. 标准 ONFT721 实现

由于当前代码库中没有 ONFT 合约，以下是基于官方标准的示例：

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import { ONFT721 } from "@layerzerolabs/onft-evm/contracts/onft721/ONFT721.sol";
import { Ownable } from "@openzeppelin/contracts/access/Ownable.sol";

/**
 * @title MyONFT721
 * @dev 标准 ONFT721 实现
 */
contract MyONFT721 is ONFT721 {
    uint256 private _nextTokenId;

    constructor(
        string memory _name,
        string memory _symbol,
        address _lzEndpoint,
        address _delegate
    ) ONFT721(_name, _symbol, _lzEndpoint, _delegate) Ownable(_delegate) {}

    /**
     * @notice 铸造新 NFT
     * @param _to 接收者地址
     */
    function mint(address _to) external onlyOwner {
        uint256 tokenId = _nextTokenId++;
        _safeMint(_to, tokenId);
    }

    /**
     * @notice 批量铸造
     */
    function mintBatch(address _to, uint256 _amount) external onlyOwner {
        for (uint256 i = 0; i < _amount; i++) {
            _safeMint(_to, _nextTokenId++);
        }
    }
}
```

### 3. ONFT Adapter 实现

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import { ONFT721Adapter } from "@layerzerolabs/onft-evm/contracts/onft721/ONFT721Adapter.sol";
import { Ownable } from "@openzeppelin/contracts/access/Ownable.sol";

/**
 * @title MyONFT721Adapter
 * @dev 适配现有 ERC721 为 ONFT
 * @dev ⚠️ 只在源链部署 Adapter!
 */
contract MyONFT721Adapter is ONFT721Adapter {
    constructor(
        address _token,          // 现有的 ERC721 合约地址
        address _lzEndpoint,
        address _delegate
    ) ONFT721Adapter(_token, _lzEndpoint, _delegate) Ownable(_delegate) {}
}
```

### 4. 发送 ONFT

```solidity
/**
 * @dev 用户发送 ONFT 的完整流程
 */
contract ONFTUsageExample {
    using OptionsBuilder for bytes;

    IONFT721 public onft;

    /**
     * @notice 发送 NFT 到另一条链
     * @param _dstEid 目标链 ID
     * @param _toAddress 接收者地址
     * @param _tokenId NFT Token ID
     */
    function sendNFT(
        uint32 _dstEid,
        address _toAddress,
        uint256 _tokenId
    ) external payable {
        // 1. 构建发送参数
        bytes32 to = bytes32(uint256(uint160(_toAddress)));

        SendParam memory sendParam = SendParam({
            dstEid: _dstEid,
            to: to,
            tokenId: _tokenId,
            extraOptions: OptionsBuilder.newOptions().addExecutorLzReceiveOption(200000, 0),
            composeMsg: "",
            onftCmd: ""
        });

        // 2. 估算费用
        MessagingFee memory fee = onft.quoteSend(sendParam, false);
        require(msg.value >= fee.nativeFee, "Insufficient fee");

        // 3. Approve ONFT 合约（如果还没 approve）
        // ERC721(onft).approve(address(onft), _tokenId);

        // 4. 发送 NFT
        onft.send{value: fee.nativeFee}(
            sendParam,
            fee,
            payable(msg.sender)
        );
    }

    /**
     * @notice 批量发送 NFT
     */
    function sendNFTBatch(
        uint32 _dstEid,
        address _toAddress,
        uint256[] calldata _tokenIds
    ) external payable {
        bytes32 to = bytes32(uint256(uint160(_toAddress)));
        bytes memory options = OptionsBuilder.newOptions().addExecutorLzReceiveOption(200000, 0);

        uint256 totalFee = 0;

        for (uint256 i = 0; i < _tokenIds.length; i++) {
            SendParam memory sendParam = SendParam({
                dstEid: _dstEid,
                to: to,
                tokenId: _tokenIds[i],
                extraOptions: options,
                composeMsg: "",
                onftCmd: ""
            });

            MessagingFee memory fee = onft.quoteSend(sendParam, false);
            require(msg.value >= totalFee + fee.nativeFee, "Insufficient fee");

            onft.send{value: fee.nativeFee}(
                sendParam,
                fee,
                payable(address(this))
            );

            totalFee += fee.nativeFee;
        }

        // 退款
        if (msg.value > totalFee) {
            payable(msg.sender).transfer(msg.value - totalFee);
        }
    }
}
```

---

## 高级功能

### 1. Message Composing (消息组合)

**Message Composing** 允许在接收消息后执行额外的逻辑，例如在收到代币后自动执行 swap。

```solidity
/**
 * @dev OFT 接收后自动 Swap 示例
 */
contract AutoSwapOFT is OFT {
    IUniswapV2Router public router;

    constructor(
        string memory _name,
        string memory _symbol,
        address _lzEndpoint,
        address _delegate,
        address _router
    ) OFT(_name, _symbol, _lzEndpoint, _delegate) {
        router = IUniswapV2Router(_router);
    }

    /**
     * @dev 发送代币并附带 compose 消息
     */
    function sendAndSwap(
        uint32 _dstEid,
        address _toAddress,
        uint256 _amountLD,
        address _swapToToken,
        uint256 _minOut
    ) external payable {
        bytes32 to = bytes32(uint256(uint160(_toAddress)));

        // 编码 compose 消息
        bytes memory composeMsg = abi.encode(_swapToToken, _minOut);

        SendParam memory sendParam = SendParam({
            dstEid: _dstEid,
            to: to,
            amountLD: _amountLD,
            minAmountLD: _amountLD,
            extraOptions: OptionsBuilder.newOptions()
                .addExecutorLzReceiveOption(200000, 0)
                .addExecutorLzComposeOption(0, 500000, 0),  // Compose option
            composeMsg: composeMsg,
            oftCmd: ""
        });

        MessagingFee memory fee = quoteSend(sendParam, false);

        send{value: fee.nativeFee}(sendParam, fee, payable(msg.sender));
    }

    /**
     * @notice Compose 回调函数
     * @dev 由 Endpoint 调用
     */
    function lzCompose(
        address _oApp,
        bytes32 /*_guid*/,
        bytes calldata _message,
        address /*_executor*/,
        bytes calldata /*_extraData*/
    ) external payable {
        require(_oApp == address(this), "Invalid OApp");
        require(msg.sender == address(endpoint), "Only endpoint");

        // 解码 compose 消息
        (address swapToToken, uint256 minOut) = abi.decode(_message, (address, uint256));

        // 执行 swap
        _executeSwap(swapToToken, minOut);
    }

    function _executeSwap(address _toToken, uint256 _minOut) internal {
        // 实现 swap 逻辑...
    }
}
```

### 2. Rate Limiting (速率限制)

防止在短时间内转移大量代币，增强安全性。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import { OFT } from "@layerzerolabs/oft-evm/contracts/OFT.sol";
import { RateLimiter } from "@layerzerolabs/oft-evm/contracts/utils/RateLimiter.sol";

/**
 * @title RateLimitedOFT
 * @dev 带速率限制的 OFT
 */
contract RateLimitedOFT is OFT, RateLimiter {
    constructor(
        string memory _name,
        string memory _symbol,
        address _lzEndpoint,
        address _delegate
    ) OFT(_name, _symbol, _lzEndpoint, _delegate) {}

    /**
     * @dev 设置速率限制
     * @param _dstEid 目标链 ID
     * @param _limit 最大数量（在 window 时间内）
     * @param _window 时间窗口（秒）
     */
    function setOutboundLimit(
        uint32 _dstEid,
        uint256 _limit,
        uint256 _window
    ) external onlyOwner {
        _setRateLimit(_dstEid, _limit, _window, true);
    }

    function setInboundLimit(
        uint32 _srcEid,
        uint256 _limit,
        uint256 _window
    ) external onlyOwner {
        _setRateLimit(_srcEid, _limit, _window, false);
    }

    /**
     * @dev 覆盖 _debit 以检查速率限制
     */
    function _debit(
        address _from,
        uint256 _amountLD,
        uint256 _minAmountLD,
        uint32 _dstEid
    ) internal override returns (uint256 amountSentLD, uint256 amountReceivedLD) {
        (amountSentLD, amountReceivedLD) = super._debit(_from, _amountLD, _minAmountLD, _dstEid);

        // 检查速率限制
        _checkAndUpdateRateLimit(_dstEid, amountSentLD, true);

        return (amountSentLD, amountReceivedLD);
    }

    /**
     * @dev 覆盖 _credit 以检查速率限制
     */
    function _credit(
        address _to,
        uint256 _amountLD,
        uint32 _srcEid
    ) internal override returns (uint256 amountReceivedLD) {
        amountReceivedLD = super._credit(_to, _amountLD, _srcEid);

        // 检查速率限制
        _checkAndUpdateRateLimit(_srcEid, amountReceivedLD, false);

        return amountReceivedLD;
    }
}
```

### 3. Ordered Messaging (有序消息)

确保消息按照发送顺序执行。

```solidity
/**
 * @dev 有序消息示例（基于 OmniCounter）
 */
abstract contract OrderedOApp is OApp {
    // 每个源链+发送者的最大已接收 nonce
    mapping(uint32 srcEid => mapping(bytes32 sender => uint64 nonce))
        private maxReceivedNonce;

    bool public orderedNonce;

    /**
     * @notice 启用/禁用有序 nonce
     */
    function setOrderedNonce(bool _orderedNonce) external onlyOwner {
        orderedNonce = _orderedNonce;
    }

    /**
     * @dev 接收消息时检查 nonce
     */
    function _lzReceive(
        Origin calldata _origin,
        bytes32 _guid,
        bytes calldata _message,
        address _executor,
        bytes calldata _extraData
    ) internal override {
        // 检查 nonce 顺序
        _acceptNonce(_origin.srcEid, _origin.sender, _origin.nonce);

        // 处理消息
        _handleMessage(_message);
    }

    /**
     * @dev 验证和更新 nonce
     */
    function _acceptNonce(
        uint32 _srcEid,
        bytes32 _sender,
        uint64 _nonce
    ) internal {
        uint64 currentNonce = maxReceivedNonce[_srcEid][_sender];

        if (orderedNonce) {
            // 有序模式：必须是下一个 nonce
            require(_nonce == currentNonce + 1, "Invalid nonce");
        }

        // 更新最大 nonce
        if (_nonce > currentNonce) {
            maxReceivedNonce[_srcEid][_sender] = _nonce;
        }
    }

    /**
     * @notice 返回期望的下一个 nonce
     */
    function nextNonce(
        uint32 _srcEid,
        bytes32 _sender
    ) public view returns (uint64) {
        if (orderedNonce) {
            return maxReceivedNonce[_srcEid][_sender] + 1;
        } else {
            return 0; // 0 表示无 nonce 限制
        }
    }

    /**
     * @notice 跳过指定 nonce（治理功能）
     */
    function skipInboundNonce(
        uint32 _srcEid,
        bytes32 _sender,
        uint64 _nonce
    ) external onlyOwner {
        endpoint.skip(address(this), _srcEid, _sender, _nonce);

        if (orderedNonce) {
            maxReceivedNonce[_srcEid][_sender]++;
        }
    }
}
```

### 4. ABA Messaging (Ping-Pong)

在接收消息后自动回复源链。

```solidity
/**
 * @dev ABA (Ping-Pong) 消息示例
 */
contract PingPongOApp is OApp {
    using OptionsBuilder for bytes;

    uint256 public pingCount;
    uint256 public pongCount;

    event Ping(uint32 indexed srcEid, uint256 count);
    event Pong(uint32 indexed srcEid, uint256 count);

    constructor(address _endpoint, address _owner)
        OApp(_endpoint, _owner)
        Ownable(_owner)
    {}

    /**
     * @notice 发送 Ping
     */
    function sendPing(uint32 _dstEid) external payable {
        bytes memory message = abi.encode("PING", ++pingCount);
        bytes memory options = OptionsBuilder.newOptions()
            .addExecutorLzReceiveOption(200000, msg.value / 2);  // 预留 gas 用于 Pong

        _lzSend(
            _dstEid,
            message,
            options,
            MessagingFee(msg.value, 0),
            payable(msg.sender)
        );
    }

    /**
     * @dev 接收消息并自动回复
     */
    function _lzReceive(
        Origin calldata _origin,
        bytes32 /*_guid*/,
        bytes calldata _message,
        address /*_executor*/,
        bytes calldata /*_extraData*/
    ) internal override {
        (string memory msgType, uint256 count) = abi.decode(_message, (string, uint256));

        if (keccak256(bytes(msgType)) == keccak256("PING")) {
            emit Ping(_origin.srcEid, count);

            // 自动回复 PONG
            _sendPong(_origin.srcEid);
        } else if (keccak256(bytes(msgType)) == keccak256("PONG")) {
            emit Pong(_origin.srcEid, count);
        }
    }

    function _sendPong(uint32 _srcEid) internal {
        bytes memory message = abi.encode("PONG", ++pongCount);
        bytes memory options = OptionsBuilder.newOptions()
            .addExecutorLzReceiveOption(200000, 0);

        _lzSend(
            _srcEid,
            message,
            options,
            MessagingFee(msg.value, 0),
            payable(address(this))
        );
    }

    // 接收 ETH 用于 Pong
    receive() external payable {}
}
```

---

## 安全最佳实践

### 1. Peer 验证

```solidity
abstract contract SecureOApp is OApp {
    /**
     * ✅ 正确：严格验证 peer
     */
    function _lzReceive(
        Origin calldata _origin,
        bytes32 _guid,
        bytes calldata _message,
        address _executor,
        bytes calldata _extraData
    ) internal override {
        // 1. 验证消息来源是已设置的 peer
        require(
            peers[_origin.srcEid] == _origin.sender,
            "Invalid peer"
        );

        // 2. 处理消息
        _handleMessage(_message);
    }

    /**
     * ✅ 正确：实现 allowInitializePath
     */
    function allowInitializePath(Origin calldata _origin)
        public
        view
        override
        returns (bool)
    {
        // 只允许已配置的 peer 初始化路径
        return peers[_origin.srcEid] == _origin.sender;
    }

    /**
     * ❌ 错误：允许所有路径
     */
    // function allowInitializePath(Origin calldata)
    //     public
    //     pure
    //     override
    //     returns (bool)
    // {
    //     return true;  // 危险！任何人都可以初始化路径
    // }
}
```

### 2. Executor 验证

```solidity
abstract contract ExecutorSecureOApp is OApp {
    // 可信的 Executor 白名单
    mapping(address => bool) public trustedExecutors;

    function setTrustedExecutor(address _executor, bool _trusted)
        external
        onlyOwner
    {
        trustedExecutors[_executor] = _trusted;
    }

    function _lzReceive(
        Origin calldata _origin,
        bytes32 _guid,
        bytes calldata _message,
        address _executor,
        bytes calldata _extraData
    ) internal override {
        // 验证 executor
        require(trustedExecutors[_executor], "Untrusted executor");

        // 或者：验证 extraData 没有被篡改
        // _validateExtraData(_extraData);

        _handleMessage(_message);
    }
}
```

### 3. 重入保护

```solidity
import { ReentrancyGuard } from "@openzeppelin/contracts/security/ReentrancyGuard.sol";

/**
 * @dev 防止重入攻击
 */
contract ReentrancyProtectedOApp is OApp, ReentrancyGuard {
    function sendMessage(uint32 _dstEid, bytes memory _message)
        external
        payable
        nonReentrant  // 防重入
    {
        _lzSend(_dstEid, _message, options, fee, payable(msg.sender));
    }

    function _lzReceive(
        Origin calldata _origin,
        bytes32 _guid,
        bytes calldata _message,
        address _executor,
        bytes calldata _extraData
    ) internal override nonReentrant {  // 防重入
        _handleMessage(_message);
    }
}
```

### 4. 金额验证

```solidity
abstract contract AmountValidationOApp is OApp {
    function _lzReceive(
        Origin calldata _origin,
        bytes32 _guid,
        bytes calldata _message,
        address _executor,
        bytes calldata _extraData
    ) internal override {
        // 如果消息中编码了期望的 msg.value
        uint256 expectedValue = abi.decode(_message, (uint256));

        // ✅ 验证实际收到的 msg.value >= 期望值
        require(msg.value >= expectedValue, "Insufficient value");

        _handleMessage(_message);
    }
}
```

### 5. DVN 配置最佳实践

```solidity
/**
 * @dev 推荐的 DVN 配置
 */
contract SecureDVNConfig {
    function getRecommendedConfig() external pure returns (UlnConfig memory) {
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

        return UlnConfig({
            confirmations: 64,              // Ethereum 推荐值
            requiredDVNCount: 3,
            optionalDVNCount: 5,
            optionalDVNThreshold: 3,        // 至少 3 个 optional
            requiredDVNs: requiredDVNs,
            optionalDVNs: optionalDVNs
        });
    }
}
```

---

## 常见陷阱与解决方案

### 陷阱 1: 精度损失 (Dust Loss)

**问题**：
```solidity
// ❌ 错误：直接发送可能导致精度损失
function sendOFT(uint32 _dstEid, uint256 _amount) external {
    // 如果 _amount = 12345 且 decimalConversionRate = 100
    // 则 12345 / 100 = 123 (整数除法)
    // 接收方收到 123 * 100 = 12300
    // 损失了 45 个单位！
}
```

**解决方案**：
```solidity
// ✅ 正确：使用 _removeDust 预先移除无法表示的精度
function sendOFT(uint32 _dstEid, uint256 _amount) external {
    uint256 cleanAmount = _removeDust(_amount);
    // cleanAmount = (12345 / 100) * 100 = 12300
    // 用户知道实际发送的数量

    _lzSend(_dstEid, abi.encode(cleanAmount), options, fee, refund);
}
```

### 陷阱 2: 未设置 Peer

**问题**：
```solidity
// ❌ 部署后忘记设置 peer
MyOApp public oappA;  // Chain A
MyOApp public oappB;  // Chain B

// oappA.send(chainB, message);  // ❌ 会失败: NoPeer(chainB)
```

**解决方案**：
```solidity
// ✅ 部署后立即设置 peer
oappA.setPeer(chainBEid, bytes32(uint256(uint160(address(oappB)))));
oappB.setPeer(chainAEid, bytes32(uint256(uint160(address(oappA)))));
```

### 陷阱 3: Gas 不足

**问题**：
```solidity
// ❌ 错误：目标链 gas 不足导致消息执行失败
bytes memory options = OptionsBuilder.newOptions()
    .addExecutorLzReceiveOption(50000, 0);  // 太少！
```

**解决方案**：
```solidity
// ✅ 正确：根据实际逻辑设置足够的 gas
bytes memory options = OptionsBuilder.newOptions()
    .addExecutorLzReceiveOption(200000, 0);  // 简单逻辑
    // 或
    .addExecutorLzReceiveOption(500000, 0);  // 复杂逻辑

// 使用 Compose 时需要额外的 gas
bytes memory options = OptionsBuilder.newOptions()
    .addExecutorLzReceiveOption(200000, 0)
    .addExecutorLzComposeOption(0, 500000, 0);  // Compose gas
```

### 陷阱 4: Fee 估算不准确

**问题**：
```solidity
// ❌ 错误：没有估算费用就发送
function sendMessage(uint32 _dstEid, string memory _message) external payable {
    bytes memory payload = abi.encode(_message);
    _lzSend(_dstEid, payload, options, MessagingFee(msg.value, 0), payable(msg.sender));
    // 如果 msg.value 不足，交易会失败
}
```

**解决方案**：
```solidity
// ✅ 正确：先估算费用，再检查
function sendMessage(uint32 _dstEid, string memory _message) external payable {
    bytes memory payload = abi.encode(_message);

    // 估算费用
    MessagingFee memory fee = _quote(_dstEid, payload, options, false);

    // 检查用户支付的费用是否足够
    require(msg.value >= fee.nativeFee, "Insufficient fee");

    _lzSend(_dstEid, payload, options, fee, payable(msg.sender));
}

// 或者：提供 quote 函数让前端估算
function quote(uint32 _dstEid, string memory _message)
    external
    view
    returns (uint256 nativeFee)
{
    bytes memory payload = abi.encode(_message);
    MessagingFee memory fee = _quote(_dstEid, payload, options, false);
    return fee.nativeFee;
}
```

### 陷阱 5: 忘记处理 msg.value

**问题**：
```solidity
// ❌ 错误：消息中要求 msg.value 但没有验证
function _lzReceive(...) internal override {
    (address recipient, uint256 requiredValue) = abi.decode(_message, (address, uint256));

    // 直接使用，没有检查 msg.value
    payable(recipient).transfer(requiredValue);  // 可能失败
}
```

**解决方案**：
```solidity
// ✅ 正确：验证 msg.value
function _lzReceive(...) internal override {
    (address recipient, uint256 requiredValue) = abi.decode(_message, (address, uint256));

    // 检查实际收到的 msg.value
    require(msg.value >= requiredValue, "Insufficient value");

    payable(recipient).transfer(requiredValue);

    // 退还多余的
    if (msg.value > requiredValue) {
        payable(msg.sender).transfer(msg.value - requiredValue);
    }
}
```

### 陷阱 6: OFTAdapter 的 Transfer Fee 代币

**问题**：
```solidity
// ❌ 默认 OFTAdapter 假设无损转账（1:1）
// 如果底层 ERC20 有转账手续费，会出问题

// 用户发送 100 tokens
// 实际到账 Adapter: 90 tokens (10% fee)
// Adapter 认为锁定了 100 tokens
// 在目标链铸造 100 tokens
// 结果：全局供应量增加了 10 tokens！
```

**解决方案**：
```solidity
// ✅ 自定义 OFTAdapter，使用余额差值
contract FeeTokenOFTAdapter is OFTAdapter {
    function _debit(
        address _from,
        uint256 _amountLD,
        uint256 _minAmountLD,
        uint32 _dstEid
    ) internal override returns (uint256 amountSentLD, uint256 amountReceivedLD) {
        // 记录转账前余额
        uint256 balanceBefore = innerToken.balanceOf(address(this));

        // 执行转账
        innerToken.safeTransferFrom(_from, address(this), _amountLD);

        // 记录转账后余额
        uint256 balanceAfter = innerToken.balanceOf(address(this));

        // 实际收到的数量 = 余额差值
        amountSentLD = balanceAfter - balanceBefore;
        amountReceivedLD = amountSentLD;

        // 检查滑点
        require(amountReceivedLD >= _minAmountLD, "Slippage exceeded");
    }
}
```

---

## 部署与配置

### 1. 使用 Hardhat/Foundry 部署

#### Hardhat 部署脚本

```typescript
// scripts/deploy-oapp.ts
import { ethers } from "hardhat";
import { Endpoint } from "./endpoints";

async function main() {
    const [deployer] = await ethers.getSigners();
    console.log("Deploying with:", deployer.address);

    // 1. 获取 Endpoint 地址
    const endpoint = Endpoint.ethereum;

    // 2. 部署 OApp
    const OApp = await ethers.getContractFactory("SimpleOApp");
    const oapp = await OApp.deploy(endpoint, deployer.address);
    await oapp.deployed();

    console.log("OApp deployed to:", oapp.address);

    // 3. 配置（如果需要）
    // await configureOApp(oapp);
}

main().catch(console.error);
```

#### Foundry 部署脚本

```solidity
// script/DeployOApp.s.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "forge-std/Script.sol";
import "../src/SimpleOApp.sol";

contract DeployOApp is Script {
    function run() external {
        uint256 deployerPrivateKey = vm.envUint("PRIVATE_KEY");
        address endpoint = vm.envAddress("LZ_ENDPOINT");

        vm.startBroadcast(deployerPrivateKey);

        SimpleOApp oapp = new SimpleOApp(endpoint, msg.sender);
        console.log("OApp deployed to:", address(oapp));

        vm.stopBroadcast();
    }
}
```

```bash
# 部署到 Ethereum
forge script script/DeployOApp.s.sol:DeployOApp \
    --rpc-url $ETH_RPC_URL \
    --broadcast \
    --verify
```

### 2. 配置 Peer 和 DVN

```typescript
// scripts/configure-oapp.ts
import { ethers } from "hardhat";

async function configure() {
    const oappEth = await ethers.getContractAt("SimpleOApp", "0x...");
    const oappBsc = await ethers.getContractAt("SimpleOApp", "0x...");

    // 1. 设置 Peer
    console.log("Setting peers...");
    await oappEth.setPeer(
        30102,  // BSC eid
        ethers.utils.zeroPad(oappBsc.address, 32)
    );
    await oappBsc.setPeer(
        30101,  // Ethereum eid
        ethers.utils.zeroPad(oappEth.address, 32)
    );

    // 2. 配置 DVN (通过 EndpointV2.setConfig)
    const configType = 2; // ULN_CONFIG_TYPE
    const config = ethers.utils.defaultAbiCoder.encode(
        ["tuple(uint64,uint8,uint8,uint8,address[],address[])"],
        [{
            confirmations: 64,
            requiredDVNCount: 3,
            optionalDVNCount: 5,
            optionalDVNThreshold: 3,
            requiredDVNs: [
                "0x589dedbd...",  // LayerZero Labs
                "0x771d10d0...",  // Chainlink
                "0xf4064220..."   // Nethermind
            ],
            optionalDVNs: [
                "0xd56e4eab...",  // Google Cloud
                "0x8DDf05F9...",  // Polyhedra
                "0x...",          // BCW
                "0x...",          // Axelar
                "0x..."           // Switchboard
            ]
        }]
    );

    const endpoint = await ethers.getContractAt(
        "ILayerZeroEndpointV2",
        "0x1a44076050125825900e736c501f859c50fE728c"
    );

    await endpoint.setConfig(
        oappEth.address,
        "0xc02Ab410f0734EFa3F14628780e6e695156024C2",  // ReceiveUln302
        [{
            eid: 30102,
            configType: configType,
            config: config
        }]
    );

    console.log("Configuration complete!");
}

configure().catch(console.error);
```

### 3. 使用 LayerZero CLI

```bash
# 安装 CLI
npm install -g @layerzerolabs/toolbox-hardhat

# 初始化配置
npx hardhat lz:oapp:config:init

# 编辑 layerzero.config.ts
# 设置 peers, DVNs, executors 等

# 应用配置
npx hardhat lz:oapp:config:set

# 查看当前配置
npx hardhat lz:oapp:config:get

# 发送 OFT
npx hardhat lz:oft:send \
    --from ethereum \
    --to bsc \
    --amount 100 \
    --recipient 0x...
```

### 4. Endpoint 地址

```typescript
// constants/endpoints.ts
export const Endpoints = {
    // Mainnet
    ethereum: "0x1a44076050125825900e736c501f859c50fE728c",
    bsc: "0x1a44076050125825900e736c501f859c50fE728c",
    avalanche: "0x1a44076050125825900e736c501f859c50fE728c",
    polygon: "0x1a44076050125825900e736c501f859c50fE728c",
    arbitrum: "0x1a44076050125825900e736c501f859c50fE728c",
    optimism: "0x1a44076050125825900e736c501f859c50fE728c",
    base: "0x1a44076050125825900e736c501f859c50fE728c",

    // Testnet
    sepolia: "0x6EDCE65403992e310A62460808c4b910D972f10f",
    bscTestnet: "0x6EDCE65403992e310A62460808c4b910D972f10f",
    mumbai: "0x6EDCE65403992e310A62460808c4b910D972f10f",
};

export const EndpointIds = {
    ethereum: 30101,
    bsc: 30102,
    avalanche: 30106,
    polygon: 30109,
    arbitrum: 30110,
    optimism: 30111,
    base: 30184,

    sepolia: 40161,
    bscTestnet: 40102,
    mumbai: 40109,
};
```

---

## 附录

### A. 完整合约示例库

查看 GitHub 仓库获取完整示例：

- **LayerZero V2 官方仓库**: https://github.com/LayerZero-Labs/LayerZero-v2
- **OApp 示例**: `packages/layerzero-v2/evm/oapp/contracts/oapp/examples/`
- **OFT 示例**: `packages/layerzero-v2/evm/oapp/contracts/oft/`
- **本审计项目**: https://github.com/cyhhao/layerzero-stargate-audit

### B. 相关文档

- 🔗 [LayerZero V2 官方文档](https://docs.layerzero.network/v2)
- 🔗 [LayerZero V2 核心概念（中文）](./translations/layerzero-v2-core-concepts-zh.md)
- 🔗 [Stargate V2 核心概念（中文）](./translations/stargate-v2-core-concepts-zh.md)
- 🔗 [LayerZero-Stargate 安全审计报告](../reports/LayerZero-Stargate-Security-Audit-Report.md)
- 🔗 [术语表](./GLOSSARY.md)

### C. 工具和资源

- **LayerZero Scan**: https://layerzeroscan.com/
- **Create LZ OApp**: `npx create-lz-oapp@latest`
- **DVN 地址列表**: https://docs.layerzero.network/v2/developers/evm/technical-reference/dvn-addresses
- **Endpoint 地址列表**: https://docs.layerzero.network/v2/developers/evm/technical-reference/deployed-contracts

### D. 社区和支持

- **Discord**: https://discord.gg/layerzero
- **Twitter**: https://twitter.com/LayerZero_Labs
- **GitHub Issues**: https://github.com/LayerZero-Labs/LayerZero-v2/issues

---

**文档版本**: v1.0
**最后更新**: 2025-10-14
**维护者**: Claude (AI Security Researcher)
**反馈**: https://github.com/cyhhao/layerzero-stargate-audit/issues

---

**⚠️ 免责声明**

本指南仅供教育和参考目的。在生产环境中部署任何智能合约前，请：
1. 进行全面的安全审计
2. 在测试网进行充分测试
3. 遵循最佳安全实践
4. 购买适当的安全保险

LayerZero V2 是强大的跨链协议，但也伴随着复杂的安全考虑。请确保您完全理解协议机制和潜在风险。
