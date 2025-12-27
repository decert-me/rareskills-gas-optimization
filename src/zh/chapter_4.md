
## 1. 批量处理交易

使用 multidelegatecall 批量处理交易，multidelegatecall 允许 msg.sender 在一个合约中调用多个函数，同时保留 msg.sender 和 msg.value 等环境变量。

*注意*：需要注意的是，由于 msg.value 是持久的，当在合约中继承 multidelegatecall 时，可能会导致开发者需要解决的问题。

以下是 Uniswap 实现 multidelegatecall 的示例：

```
function multicall(bytes[] calldata data) public payable override returns (bytes[] memory results) {
        results = new bytes[](data.length);
        for (uint256 i = 0; i < data.length; i++) {
            (bool success, bytes memory result) = address(this).delegatecall(data[i]);

            if (!success) {
                // Next 5 lines from https://ethereum.stackexchange.com/a/83577
                if (result.length < 68) revert();
                assembly {
                    result := add(result, 0x04)
                }
                revert(abi.decode(result, (string)));
            }

            results[i] = result;
        }
    }
```

## 2. 使用 Merkle 树减少存储

在 Airdrop、白名单（Whitelist）等场景中，使用 Merkle 树可以大幅减少链上存储成本。

**传统方式的问题：**

如果要在链上存储 10,000 个白名单地址，需要：
```solidity
// ❌ 传统方式：高昂的存储成本
mapping(address => bool) public whitelist;

// 部署时需要逐个添加地址
function addToWhitelist(address[] calldata addresses) external onlyOwner {
    for (uint256 i = 0; i < addresses.length; i++) {
        whitelist[addresses[i]] = true;  // 每个地址约 20,000 gas
    }
}
// 10,000 个地址 ≈ 200,000,000 gas！
```

**Merkle 树方案：**

使用 Merkle 树，只需在链上存储一个 32 字节的根哈希：

```solidity
// ✅ Merkle 树方式：极低的存储成本
import "@openzeppelin/contracts/utils/cryptography/MerkleProof.sol";

contract MerkleAirdrop {
    bytes32 public immutable merkleRoot;
    mapping(address => bool) public claimed;

    constructor(bytes32 _merkleRoot) {
        merkleRoot = _merkleRoot;  // 仅存储根哈希，约 20,000 gas
    }

    function claim(
        uint256 amount,
        bytes32[] calldata merkleProof
    ) external {
        require(!claimed[msg.sender], "Already claimed");

        // 验证 Merkle 证明
        bytes32 leaf = keccak256(abi.encodePacked(msg.sender, amount));
        require(
            MerkleProof.verify(merkleProof, merkleRoot, leaf),
            "Invalid proof"
        );

        claimed[msg.sender] = true;
        // 执行 airdrop 逻辑...
    }
}
```

**成本对比：**

| 方案 | 部署成本 (10,000 地址) | 用户 claim 成本 |
|------|----------------------|----------------|
| 传统 mapping | ~200,000,000 gas | ~5,000 gas |
| Merkle 树 | ~20,000 gas | ~50,000 gas (含 proof 验证) |

查看 [以太坊智能合约中的 Merkle 树](https://learnblockchain.cn/article/22678) 可以深入了解 Merkle 树如何构建与验证。

## 3. ECDSA 签名替代默克尔树

使用 ECDSA 签名替代默克尔树进行 allowlist 和 airdrop，默克尔树使用大量的 calldata，并且随着默克尔证明的大小增加而增加成本。通常情况下，使用数字签名在 gas 方面比默克尔证明更便宜。

### ECDSA 签名方案

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/utils/cryptography/ECDSA.sol";
import "@openzeppelin/contracts/utils/cryptography/MessageHashUtils.sol";

contract SignatureAirdrop {
    using ECDSA for bytes32;
    using MessageHashUtils for bytes32;

    address public immutable signer;
    mapping(address => bool) public claimed;

    constructor(address _signer) {
        signer = _signer;
    }

    function claim(
        uint256 amount,
        bytes calldata signature
    ) external {
        require(!claimed[msg.sender], "Already claimed");

        // 构建消息哈希
        bytes32 messageHash = keccak256(abi.encodePacked(msg.sender, amount));
        bytes32 ethSignedMessageHash = messageHash.toEthSignedMessageHash();

        // 验证签名
        address recoveredSigner = ethSignedMessageHash.recover(signature);
        require(recoveredSigner == signer, "Invalid signature");

        claimed[msg.sender] = true;
        // 执行 airdrop 逻辑...
    }
}
```

### Merkle 树 vs ECDSA 签名：如何选择？

两种方案各有优劣，应根据具体场景选择：

#### Gas 成本对比

| 方案 | 部署成本 | 用户 claim 成本 | Calldata 大小 |
|------|---------|----------------|---------------|
| **Merkle 树** | ~20,000 gas | ~50,000 gas | 约 14×32 字节 (10k 用户) |
| **ECDSA 签名** | ~20,000 gas | ~35,000 gas | 65 字节 (固定) |

#### 选择 Merkle 树的场景：

✅ **去中心化要求高** - Merkle 树数据可以公开验证，任何人都可以生成证明
✅ **大规模用户** - 当用户数量超过百万级别时，链下管理签名可能不现实
✅ **无需信任中心化服务器** - 完整的 Merkle 树可以存储在 IPFS 或公开的地方
✅ **需要批量验证** - 可以一次性验证多个 proof
✅ **透明度要求** - 所有合格地址可以公开列表

#### 选择 ECDSA 签名的场景：

✅ **用户体验优先** - 签名固定 65 字节，calldata 成本更低
✅ **动态白名单** - 可以随时签发新的签名，无需重新部署
✅ **Gas 成本敏感** - 签名验证比 Merkle proof 验证便宜(与层级有关)
✅ **中小规模用户** - 10k 以下用户，链下签名管理可行
✅ **链下灵活性** - 可以为不同用户签发不同的参数（金额、到期时间等）


#### 安全性考虑

| 考虑因素 | Merkle 树 | ECDSA 签名 |
|---------|----------|-----------|
| **私钥泄露风险** | 低 - 无私钥 | 高 - 需要保护签名私钥 |
| **可验证性** | 高 - 完全透明 | 中 - 依赖签名者 |
| **抗审查性** | 高 - 数据可公开 | 中 - 依赖签名服务 |
| **签名重放** | 不适用 | 需要防护（nonce/过期时间） |


#### 混合方案

对于某些场景，可以结合两种方案的优势：

```solidity
contract HybridAirdrop {
    bytes32 public merkleRoot;
    address public signer;
    mapping(address => bool) public claimed;

    // 白名单用户使用 Merkle 树
    function claimWithMerkle(
        uint256 amount,
        bytes32[] calldata proof
    ) external {
        // Merkle 验证逻辑
    }

    // 动态分配使用签名
    function claimWithSignature(
        uint256 amount,
        uint256 deadline,
        bytes calldata signature
    ) external {
        require(block.timestamp <= deadline, "Signature expired");
        // 签名验证逻辑
    }
}
```

## 4. 使用 ERC20Permit 

ERC20Permit 可以在一笔交易中实现授权和转账步骤。

[ERC20 Permit](https://learnblockchain.cn/article/1496) 有一个额外的函数 permit，可以接受代币持有者的数字签名，以支持对某个地址的授权。这样，被授权者可以提交授权签名和转账交易，并合并为一笔交易。授予授权的用户不需要支付任何 gas 费用，而被授权人可以将许可和 transferFrom 交易合并为一笔交易。

## 5. 迁移到 Layer2

以太坊经过多次扩容，手续费下降了很多，但是对于游戏或其它高吞吐量、低交易价值的应用，成本也就较高，这时可以考虑使用 Layer2 （Optimism 、 Arbitrum、Base）或其他侧链（如 BNB Chain 、Polygon），在那里进行交易成本更低。

可以使用跨链桥进行 Token 和数据的桥接


## 6. 如果适用，使用状态通道

状态通道可能是[以太坊](https://learnblockchain.cn/tags/以太坊?map=EVM)最古老但仍可用的可扩展性解决方案。与 L2 不同，它们是应用程序特定的。用户不是将他们的交易提交到链上，而是将资产提交到智能合约，然后彼此共享绑定签名作为状态转换。当操作结束时，他们将最终结果提交到链上。

如果参与者中有人不诚实，那么诚实的参与者可以使用对方的签名来强制智能合约释放他们的资产。

## 7. 使用投票委托

在 [ERC20 Votes](https://learnblockchain.cn/article/11274) 的教程中更详细地描述了这种模式。与其让每个代币持有者都进行投票，不如只让代表进行投票，从而减少了投票交易的数量。

## 8. 使用 ERC1155 代替 ERC721

如果你要跟踪 NFT 的数量，ERC1155 可能是更好的选择。

ERC721 的 balanceOf 函数在实践中很少使用，但在进行铸造和转账时会增加存储开销。ERC1155 按 id 跟踪余额，并使用相同的余额来跟踪 id 的所有权。如果每个 id 的最大供应量为一，则该代币对于每个 id 都是非同质化的。

## 9. 使用 ERC1155 或 ERC6909 代替多个 ERC20

这是 ERC1155 代币的最初目的。每个单独的代币的行为类似于 ERC20，但只需要部署一个合约。

这种方法的缺点是这些代币将与大多数 DeFi 交换原语不兼容。

ERC1155 在所有转账方法上使用回调。如果不需要这样的功能，可以使用 [ERC6909](https://learnblockchain.cn/docs/eips/EIPS/eip-6909)。

## 10. UUPS 升级比透明代理更节省 gas

透明升级代理模式需要在每次交易发生时从存储槽（由 ERC-1967 定义）中读取管理员地址，以查看调用者是否为管理员。这会引入额外的存储读取。

## 11. 考虑使用 OpenZeppelin 的替代方案

OpenZeppelin 是一个很棒且受欢迎的智能合约库，但还有其它值得考虑的替代方案。这些替代方案提供更好的 gas 效率，并经过开发者的测试和推荐。

### 推荐替代库 Solady

[Solady](https://github.com/Vectorized/solady) 是目前最高效的 [Solidity](https://learnblockchain.cn/course/93) 库，强调使用汇编语言进行极致优化。

Solady 同样经过多次审计和实战验证，大量使用汇编，Gas 效率极高， 覆盖 ERC20、ERC721、ERC1155、权限控制、数学库等

**Gas 对比（2024 数据）：**

| 功能 | [OpenZeppelin](https://learnblockchain.cn/tags/OpenZeppelin?map=EVM) | Solady | 节省 |
|------|-------------|--------|------|
| ERC20 Transfer | 677 gas | 546 gas | **19%** |
| ERC721 Mint | 2,800 gas | 2,200 gas | **21%** |
| MulDivDown | 674 gas | 504 gas | **25%** |
| Sqrt | 1,146 gas | 683 gas | **40%** |


### 使用示例对比

**[OpenZeppelin](https://learnblockchain.cn/tags/OpenZeppelin?map=EVM) vs Solady**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

// ❌ OpenZeppelin (标准但 gas 较高)
import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract TokenOZ is ERC20 {
    constructor() ERC20("MyToken", "MTK") {
        _mint(msg.sender, 1000000 * 10**18);
    }
}

// ✅ Solady (更高效)
import "solady/tokens/ERC20.sol";

contract TokenSolady is ERC20 {
    function name() public pure override returns (string memory) {
        return "MyToken";
    }

    function symbol() public pure override returns (string memory) {
        return "MTK";
    }

    constructor() {
        _mint(msg.sender, 1000000 * 10**18);
    }
}
// Transfer gas: 546 vs 677 (节省 19%)
```

当 Gas 优化是首要目标（如高频调用的合约）， 且团队有一定的 [Solidity](https://learnblockchain.cn/course/93) 和汇编经验，应该考虑使用 Solady 。

- [Solady 官方仓库](https://github.com/Vectorized/solady)
- [Solady 文档](https://github.com/Vectorized/solady/tree/main/docs)


## 使用可迭代映射

数组是可迭代的，但是数组的维护成本很高，例如要从数组中间插入和删除元素，gas 性能将非常差，因为涉及到很多元素的移动。

原生的 Solidity 的 mapping 当前是不可以迭代的，但是我们将通过扩展映射数据结构来实现一个链表，来使迭代成为可能，从而以最小的 gas 成本开销支持迭代功能。

我们可参考[O(1)可迭代映射](https://learnblockchain.cn/article/1632) 和 [如何维护排序列表](https://learnblockchain.cn/article/1638) .




