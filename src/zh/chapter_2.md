## 1. 关于部署gas 
**CREATE 和 CREATE2 合约部署成本：**

```
部署成本 = 基础成本 + 代码存储成本 + 初始化成本
```

基础成本：
CREATE: 32,000 Gas
CREATE2: 32,000 Gas（相同）

代码存储成本：
200 Gas/字节

示例：
合约字节码大小：10,000 字节

成本计算：
- 基础：32,000 Gas
- 代码存储：10,000 × 200 = 2,000,000 Gas
- 构造函数执行：变量（例如 500,000 [Gas](https://learnblockchain.cn/tags/Gas?map=EVM)）
总计：约 2,532,000 [Gas](https://learnblockchain.cn/tags/Gas?map=EVM)

实际成本（50 gwei）：
2,532,000 × 50 gwei = 0.1266 ETH ≈ $250

## 2. 大型合约优化

根据 EIP-170 合约大小限制最大为：24,576 字节 (24 KB)
超出限制的解决方案：

### 将合约分开

将合约分开应该永远你的首要方法。如何把合约分成多个小合约？一般来说，这迫使你为你的合约想出一个好的架构。从代码可读性的角度来看，较小的合约总是首选。

对于拆分合约，要问自己这几个问题：

哪些函数应该在一起？每一组函数可能最好放在自己的合约中。
哪些函数不需要读取合约的状态，或者只需要读取状态的一个特定子集？
你能把存储和函数分开吗？

拆分合约几个可能的方案：

1. Diamond Pattern（钻石模式）
   - 拆分功能到多个 Facet
   - 通过 Diamond Proxy 聚合

2. Proxy Pattern（代理模式）
   - 逻辑合约可随时更换
   - 单个逻辑合约 < 24 KB
 
3. 使用库（Library）

### 代码优化

1. **移除未使用代码**

这一点应该是很明显的，更多函数会将合约规模提高不少。可以删除内部/私有函数，只要该函数只被调用一次，就可以简单地内联代码。

2. **优化器设置**

**runs 设置**:

优化器的设置 run 默认值为200，表示它试图在一个函数被调用200次的情况下优化字节码。如果你把它改为1，你基本上是告诉优化器为每个函数只运行一次的情况进行优化。一个只运行一次的优化函数意味着它对部署本身进行了优化。请注意，这增加了运行函数的Gas成本。

**IR (中间表示优化)**:

通过启用 `viaIR` 编译选项，Solidity 编译器会使用基于 Yul 的中间表示(IR)来优化代码，这可以显著减小合约部署大小。

**IR 优化的好处：**
- 更激进的代码优化，生成更紧凑的字节码
- 某些情况下可以减少 10-20% 的合约大小
- 对接近 24KB 限制的大型合约特别有用
- 可能降低运行时 gas 成本（取决于具体代码）

**配置方式：**

Foundry 配置（foundry.toml）：
```toml
[profile.default]
solc_version = "0.8.24"
optimizer = true
optimizer_runs = 1
via_ir = true  # 启用 IR 优化
```

**注意事项：**
- IR 优化会显著增加编译时间（可能慢 2-5 倍）
- 某些边缘情况下可能产生不同的 gas 消耗
- 不是所有合约都能从 IR 优化中受益，需要测试对比

**权衡考虑：**
- 如果主要目标是减小部署大小：`runs: 1` + `viaIR: true`
- 如果主要目标是降低运行时 gas：`runs: 200`（或更高）+ `viaIR: true`
- 优先考虑将合约拆分，IR 优化作为辅助手段

3. 链下存储

使用 IPFS 存储元数据， 链上仅存哈希


## 3. 预测地址

使用账户 nonce 来预测相互依赖的智能合约的地址，从而避免使用存储变量和地址设置函数

在传统的合约部署中（[create](https://learnblockchain.cn/article/22619?course_id=93#%E4%BD%BF%E7%94%A8%20create%20%E5%88%9B%E5%BB%BA%E5%90%88%E7%BA%A6)），智能合约的地址可以根据部署者的地址和他们的 nonce 进行确定性计算。

[Solady 的 LibRLP 库](https://github.com/Vectorized/solady/blob/6c54795ef69838e233020e9ab29f3f6288efdf06/src/utils/LibRLP.sol#L27)可以帮助我们做到这一点。


假设有以下示例场景：

StorageContract 只允许 Writer 设置存储变量 x，这意味着它需要知道 Writer 的地址。但是，为了让 Writer 写入 StorageContract，它也需要知道 StorageContract 的地址。

下面的实现是对这个问题的一种简单方法。它通过一个设置器函数在部署后设置存储变量。但是存储变量是昂贵的，我们更愿意避免使用它们。

```
contract StorageContract {
    address immutable public writer;
    uint256 public x;
    
    constructor(address _writer) {
        writer = _writer;
    }

    function setX(uint256 x_) external {
        require(msg.sender == address(writer), "only writer can set");
        x = x_;
    }
}

contract Writer {
    StorageContract public storageContract;

    // cost: 49291
    function set(uint256 x_) external {
        storageContract.setX(x_);
    }

    function setStorageContract(address _storageContract) external {
        storageContract = StorageContract(_storageContract);
    }
}
```

这种方法在部署和运行时都更加昂贵。它涉及部署 Writer，然后将已部署的 Writer 地址设置为 writer，再将 Writer 的 StorageContract 变量设置为新创建的 StorageContract。这涉及很多步骤，并且由于我们将 StorageContract 存储在存储器中，所以可能很昂贵。调用 Writer.setX() 的成本为 49k gas。

更高效的方法是在部署之前计算 StorageContract 和 Writer 将要部署到的地址，并在它们的构造函数中设置这些地址。

以下是一个示例：

```
import {LibRLP} from "https://github.com/vectorized/solady/blob/main/src/utils/LibRLP.sol";

contract StorageContract {
    address immutable public writer;
    uint256 public x;
    
    constructor(address _writer) {
        writer = _writer;
    }

    // cost: 47158
    function setX(uint256 x_) external {
        require(msg.sender == address(writer), "only writer can set");
        x = x_;
    }
}

contract Writer {
    StorageContract immutable public storageContract;
    
    constructor(StorageContract _storageContract) {
        storageContract = _storageContract;
    }

    function set(uint256 x_) external {
        storageContract.setX(x_);
    }
}

// one time deployer.
contract BurnerDeployer {
    using LibRLP for address;

    function deploy() public returns(StorageContract storageContract, address writer) {
        StorageContract storageContractComputed = StorageContract(address(this).computeAddress(2)); // contracts nonce start at 1 and only increment when it creates a contract
        writer = address(new Writer(storageContractComputed)); // first creation happens here using nonce = 1
        storageContract = new StorageContract(writer); // second create happens here using nonce = 2
        require(storageContract == storageContractComputed, "false compute of create1 address"); // sanity check
    }
}
```

在这里，调用 Writer.setX() 的成本为 47k gas。通过在部署 StorageContract 之前预先计算其部署地址，并在部署 Writer 时使用该地址，我们节省了 2k+ 的 gas，因此不需要使用设置器函数。

不需要使用单独的合约来使用这种技术，你可以在部署脚本中完成。

如果你希望进一步了解，我们提供了由 [Philogy](https://twitter.com/real_philogy) 制作的 [地址预测视频教程](https://www.youtube.com/watch?v=eb3qtUc4UE4)。

## 4. 将构造函数设为可支付

```
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.24;

contract A {}

contract B {
    constructor() payable {}
}
```

将构造函数设为可支付在部署时节省了 200 gas。这是因为非可支付函数会在其中插入一个隐式的 require(msg.value == 0)。此外，部署时的字节码较少意味着 calldata 更小，从而减少了 gas 成本。

将常规函数设为非可支付有很好的理由，但通常合约是由特权地址部署的，你可以合理地假设它不会发送以太币。如果是经验不丰富的用户部署合约，则可能不适用。

## 5. 优化 IPFS 哈希

通过优化 IPFS 哈希以获得更多的零（或使用--no-cbor-metadata 编译器选项），可以减小部署大小

我们已经在我们的[智能合约元数据教程](https://learnblockchain.cn/article/5879)中解释过这一点，但是为了回顾一下，[Solidity](https://learnblockchain.cn/course/93) 编译器会将51个字节的元数据附加到实际的智能合约代码中。由于每个部署字节的成本为200 gas，删除它们可以减少超过10,000 gas 的部署成本。

然而，这并不总是理想的，因为它可能会影响智能合约的验证。相反，开发人员可以寻找使附加的 IPFS 哈希中具有更多零的代码注释。

## 6. selfdestruct 一次性合约

如果合约只用于一次性使用，则在构造函数中使用 selfdestruct

有时，合约用于在一个交易中部署多个合约，这就需要在构造函数中执行。

如果合约的唯一用途是构造函数中的代码，则在操作结束时进行 selfdestruct 将节省 gas。

> **⚠️ 重要提示（2024 Cancun 升级 - EIP-6780）**：

根据 [EIP-6780](https://learnblockchain.cn/docs/eips/EIPS/eip-6780/)（2024年3月 Cancun 升级），`SELFDESTRUCT` 的行为已被严格限制：

**在 Cancun 升级之前：** `SELFDESTRUCT` 会删除合约代码，清除所有存储，转移 ETH 余额，退还部分 gas（24,000 gas）

**在 Cancun 升级之后：**
- **仅在同一交易内创建并销毁的合约**才保留完整的 `SELFDESTRUCT` 功能
- 对于已存在的合约调用 `SELFDESTRUCT`， **不再**删除合约代码， **不再**清除存储，仍然转移 ETH 余额，**不再**退还 gas

不过在在构造函数中使用 SELFDESTRUCT是仍然有效的：


```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

contract ConstructorSelfDestruct {
    constructor() payable {
        // 执行一些初始化逻辑
        _initialize();

        // ✅ 在构造函数中自毁（同一交易内）
        selfdestruct(payable(tx.origin));
        // 合约代码和存储会被完全删除，节省部署成本
    }

    function _initialize() internal {
        // 初始化逻辑
    }
}

contract Factory {
    function deployAndExecute() external payable {
        // ✅ 部署并立即执行逻辑，然后自毁
        new ConstructorSelfDestruct{value: msg.value}();
        // 合约被完全删除，节省存储成本
    }
}
```


## 7. 权衡内部函数与修改器
在选择内部函数和[修改器(Modifier)](https://learnblockchain.cn/article/22555)之间时要理解权衡.

修改器在使用它的地方注入其实现字节码，而内部函数则跳转到运行时代码中其实现的位置。这给两种选项带来了一些权衡。

- 多次使用修改器意味着重复性和运行时代码大小的增加，但由于不需要跳转到内部函数执行偏移量并跳回继续执行，这减少了 gas 成本。这意味着如果运行时 gas 成本对你最重要，那么修改器应该是你的选择，但如果部署 gas 成本和/或减小创建代码的大小对你最重要，那么使用内部函数将是最佳选择。
- 然而，修改器的权衡是它只能在函数的开头或结尾执行。这意味着没有内部函数的情况下在函数的中间执行它不可能直接实现，而内部函数则破坏了原始目的。这影响了它的灵活性。然而，内部函数可以在函数的任何位置调用。

示例展示了使用修改器和内部函数的 gas 成本差异

```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/** deployment gas cost: 195435
    gas per call:
              restrictedAction1: 28367
              restrictedAction2: 28377
              restrictedAction3: 28411
 */
 contract Modifier {
    address owner;
    uint256 val;

    constructor() {
        owner = msg.sender;
    }

    modifier onlyOwner() {
        require(msg.sender == owner);
        _;
    }

    function restrictedAction1() external onlyOwner {
        val = 1;
    }

    function restrictedAction2() external onlyOwner {
        val = 2;
    }

    function restrictedAction3() external onlyOwner {
        val = 3;
    }
}



/** deployment gas cost: 159309
    gas per call:
              restrictedAction1: 28391
              restrictedAction2: 28401
              restrictedAction3: 28435
 */
 contract InternalFunction {
    address owner;
    uint256 val;

    constructor() {
        owner = msg.sender;
    }

    function onlyOwner() internal view {
        require(msg.sender == owner);
    }

    function restrictedAction1() external {
        onlyOwner();
        val = 1;
    }

    function restrictedAction2() external {
        onlyOwner();
        val = 2;
    }

    function restrictedAction3() external {
        onlyOwner();
        val = 3;
    }
}
```

| 操作               | 部署       | 受限操作1         | 受限操作2         | **受限操作3**         |
| ------------------ | ---------- | ----------------- | ----------------- | --------------------- |
| 修改器             | 195435     | 28367             | 28377             | 28411                 |
| 内部函数           | 159309     | 28391             | 28401             | 28435                 |

从上表可以看出，使用修改器的合约在部署时比使用内部函数的合约多花费了超过35k 的 gas ，这是因为在3个函数中重复使用了 onlyOwner 功能。

在运行时，我们可以看到每个使用修改器的函数比使用内部函数的函数少消耗了固定的24 gas 。

## 8. 使用克隆或元代理

当部署多个相似的[智能合约](https://learnblockchain.cn/tags/%E6%99%BA%E8%83%BD%E5%90%88%E7%BA%A6)时， gas 成本可能很高。为了降低这些成本，可以使用最小化的克隆或元代理，它们在其字节码中存储了实现合约的地址，并以代理的方式与其交互。

然而，克隆的运行时成本和部署成本之间存在权衡。由于交互时，使用克隆会多一部克隆本身的加载，因此交互gas 会高一点点(几十到一两百 gas 量级)，通常来说，这个成本在普通的函数里都可以忽略。尤其是相比大幅下降部署成本来说。
但如果是非常高频的调用，这部分多出的 gas 成本需要考虑。

例如，Gnosis Safe 合约使用克隆来降低部署成本。

使用 EIP-1167最小代理标准可参考，Solidity 教程中的[这篇文章](https://learnblockchain.cn/article/11333)


## 9. 管理员函数可以接受支付

我们可以将管理员特定函数设置为可接受支付，以节省 gas ，因为编译器不会检查函数的调用值。

这也会使合约更小，部署成本更低，因为创建和运行时代码中的操作码更少。

## 10. 使用自定义错误

[自定义错误](https://learnblockchain.cn/article/22557?course_id=93#revert()%20-%20%E7%81%B5%E6%B4%BB%E7%9A%84%E9%94%99%E8%AF%AF%E5%A4%84%E7%90%86)（通常）比 require 语句更小，由于自定义错误的处理方式，它们比使用字符串的 require 语句更便宜。Solidity 只存储错误签名的前4个字节，并且只返回这4个字节。这意味着在回滚时，只需要在内存中存储4个字节。而对于 require 语句中的字符串消息，[Solidity](https://learnblockchain.cn/course/93) 必须至少存储（在内存中）并回滚64个字节。

下面是一个示例。

```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

contract CustomError {
    error InvalidAmount();

    function withdraw(uint256 _amount) external pure {
        if (_amount > 10 ether) revert InvalidAmount();
    }
}

// This uses more gas than the above contract
contract NoCustomError {
    function withdraw(uint256 _amount) external pure {
        require(_amount <= 10 ether, "Error: Pass in a valid amount");
    }
}
```

## 11. 使用现有的 create2 工厂

使用现有的 create2 工厂而不是部署自己的工厂.

如果你需要确定性地址，通常可以重复使用预先部署的地址。
