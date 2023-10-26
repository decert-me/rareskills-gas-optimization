# 部署时节省 gas

- [1. 使用账户 nonce 来预测相互依赖的智能合约的地址，从而避免使用存储变量和地址设置函数](#1-使用账户-nonce-来预测相互依赖的智能合约的地址从而避免使用存储变量和地址设置函数)
- [2. 将构造函数设为可支付](#2-将构造函数设为可支付)
- [3. 通过优化 IPFS 哈希以获得更多的零（或使用 --no-cbor-metadata 编译器选项）来减小部署大小](#3-通过优化-ipfs-哈希以获得更多的零或使用--no-cbor-metadata-编译器选项可以减小部署大小)
- [4. 如果合约只使用一次，可以在构造函数中使用 selfdestruct](#4-如果合约只用于一次性使用则在构造函数中使用-selfdestruct)
- [5. 在选择内部函数和修饰器之间时要了解权衡](#5-在选择内部函数和修饰器之间时要理解权衡)
- [6. 在部署不经常调用的非常相似的智能合约时，可以使用克隆或元代理](#6-在部署不经常调用的非常相似的智能合约时使用克隆或元代理)
- [7. 管理员函数可以接受支付](#7-管理员函数可以接受支付)
- [8. 自定义错误（通常）比 require 语句更小](#8-自定义错误通常比-require-语句更小)
- [9. 使用现有的 create2 工厂而不是部署自己的工厂](#9-使用现有的-create2-工厂而不是部署自己的工厂)


## 1. 使用账户 nonce 来预测相互依赖的智能合约的地址，从而避免使用存储变量和地址设置函数

在传统的合约部署中（[create](https://hackmd.io/eQJUW4PLQN-6HRrgyxdhsQ?view)），智能合约的地址可以根据部署者的地址和他们的 nonce 进行确定性计算。

[Solady 的 LibRLP 库](https://github.com/Vectorized/solady/blob/6c54795ef69838e233020e9ab29f3f6288efdf06/src/utils/LibRLP.sol#L27)可以帮助我们做到这一点。

以下是对应的中文翻译：

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

## 2. 将构造函数设为可支付

```
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.20;

contract A {}

contract B {
    constructor() payable {}
}
```

将构造函数设为可支付在部署时节省了 200 gas。这是因为非可支付函数会在其中插入一个隐式的 require(msg.value == 0)。此外，部署时的字节码较少意味着 calldata 更小，从而减少了 gas 成本。

将常规函数设为非可支付有很好的理由，但通常合约是由特权地址部署的，你可以合理地假设它不会发送以太币。如果是经验不丰富的用户部署合约，则可能不适用。

## 3. 通过优化 IPFS 哈希以获得更多的零（或使用--no-cbor-metadata 编译器选项），可以减小部署大小

我们已经在我们的[智能合约元数据教程](https://www.rareskills.io/post/solidity-metadata)中解释过这一点，但是为了回顾一下，Solidity 编译器会将51个字节的元数据附加到实际的智能合约代码中。由于每个部署字节的成本为200 gas，删除它们可以减少超过10,000 gas 的部署成本。

然而，这并不总是理想的，因为它可能会影响智能合约的验证。相反，开发人员可以寻找使附加的 IPFS 哈希中具有更多零的代码注释。

## 4. 如果合约只用于一次性使用，则在构造函数中使用 selfdestruct

有时，合约用于在一个事务中部署多个合约，这就需要在构造函数中执行。

如果合约的唯一用途是构造函数中的代码，则在操作结束时进行 selfdestruct 将节省 gas。

尽管 selfdestruct 在即将到来的硬分叉中将被删除，但根据 [EIP 6780](https://eips.ethereum.org/EIPS/eip-6780)，它仍将在构造函数中得到支持。

## 5. 在选择内部函数和修饰器之间时要理解权衡

修饰器在使用它的地方注入其实现字节码，而内部函数则跳转到运行时代码中其实现的位置。这给两种选项带来了一些权衡。

- 多次使用修饰器意味着重复性和运行时代码大小的增加，但由于不需要跳转到内部函数执行偏移量并跳回继续执行，这减少了 gas 成本。这意味着如果运行时 gas 成本对你最重要，那么修饰器应该是你的选择，但如果部署 gas 成本和/或减小创建代码的大小对你最重要，那么使用内部函数将是最佳选择。
- 然而，修饰器的权衡是它只能在函数的开头或结尾执行。这意味着没有内部函数的情况下在函数的中间执行它不可能直接实现，而内部函数则破坏了原始目的。这影响了它的灵活性。然而，内部函数可以在函数的任何位置调用。

示例展示了使用修饰器和内部函数的 gas 成本差异

```
// SPDX-License-Identifier: MIT
pragma solidity 0.8.19;

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
| 修饰器             | 195435     | 28367             | 28377             | 28411                 |
| 内部函数           | 159309     | 28391             | 28401             | 28435                 |

从上表可以看出，使用修饰器的合约在部署时比使用内部函数的合约多花费了超过35k 的 gas ，这是因为在3个函数中重复使用了 onlyOwner 功能。

在运行时，我们可以看到每个使用修饰器的函数比使用内部函数的函数少消耗了固定的24 gas 。

## 6. 在部署不经常调用的非常相似的智能合约时，使用克隆或元代理

当部署多个相似的智能合约时， gas 成本可能很高。为了降低这些成本，可以使用最小化的克隆或元代理，它们在其字节码中存储了实现合约的地址，并以代理的方式与其交互。

然而，克隆的运行时成本和部署成本之间存在权衡。由于克隆使用了 delegatecall，与普通合约相比，与其交互的成本更高，因此只有在不需要频繁与其交互时才应使用克隆。例如，Gnosis Safe 合约使用克隆来降低部署成本。

从我们的博客文章中了解更多关于如何使用克隆和元代理来降低部署智能合约的 gas 成本：

- EIP-1167: 最小代理标准
- EIP-3448 元代理克隆

## 7. 管理员函数可以接受支付

我们可以将管理员特定函数设置为可接受支付，以节省 gas ，因为编译器不会检查函数的调用值。

这也会使合约更小，部署成本更低，因为创建和运行时代码中的操作码更少。

## 8. 自定义错误（通常）比 require 语句更小

由于自定义错误的处理方式，它们比使用字符串的 require 语句更便宜。Solidity 只存储错误签名的前4个字节，并且只返回这4个字节。这意味着在回滚时，只需要在内存中存储4个字节。而对于 require 语句中的字符串消息，Solidity 必须至少存储（在内存中）并回滚64个字节。

下面是一个示例。

```
// SPDX-License-Identifier: MIT
pragma solidity 0.8.20;

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

## 9. 使用现有的 create2 工厂而不是部署自己的工厂

标题已经很明确了。如果你需要确定性地址，通常可以重复使用预先部署的地址。
