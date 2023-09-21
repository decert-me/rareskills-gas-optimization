# 跨合约调用

- [1. 使用转账钩子来处理代币，而不是从目标智能合约发起转账](#1-使用转账钩子来处理代币而不是从目标智能合约发起转账)
- [2. 在转移以太币时，使用 fallback 或 receive 而不是 deposit()](#2-在转移以太币时使用-fallback-或-receive-代替-deposit)
- [3. 在进行跨合约调用时，使用 ERC2930 访问列表事务来预热存储槽和合约地址](#3-在进行跨合约调用以预热存储槽和合约地址时使用-erc2930-访问列表交易)
- [4. 在有意义的情况下缓存对外部合约的调用（例如缓存来自 Chainlink Oracle 的返回数据）](#4-在有意义的情况下缓存对外部合约的调用例如缓存来自-chainlink-oracle-的返回数据)
- [5. 在类似路由器的合约中实现 multicall](#5-在类似路由器的合约中实现-multicall)
- [6. 通过构建单体架构来避免合约调用](#6-通过使架构成为单体结构来避免合约调用)


## 1. 使用转账钩子来处理代币，而不是从目标智能合约发起转账

假设你有一个合约 A，它接受代币 B（一个 NFT 或一个 ERC1363代币）。简单的工作流程如下：

1. msg.sender 批准合约 A 接受代币 B
2. msg.sender 调用合约 A 将代币从 msg.sender 转移到 A
3. 合约 A 然后调用代币 B 进行转移
4. 代币 B 执行转移，并在合约 A 中调用 onTokenReceived()
5. 合约 A 从 onTokenReceived() 返回一个值给代币 B
6. 代币 B 将执行返回给合约 A

这种方式非常低效。更好的方式是让 msg.sender 调用合约 B 进行转账，合约 B 再调用合约 A 中的 tokenReceived 钩子。

请注意：

- 所有 ERC1155 代币都包含一个转账钩子
- ERC721 中的 safeTransfer 和 safeMint 具有转账钩子
- ERC1363 具有 transferAndCall
- ERC777 具有转账钩子，但已被弃用。如果你需要可替代的代币，请使用 ERC1363 或 ERC1155

如果你需要向合约 A 传递参数，只需使用 data 字段，并在合约 A 中解析该字段。

## 2. 在转移以太币时，使用 fallback 或 receive 代替 deposit()

与上述类似，你可以“只是转移”以太币给合约，并让其响应转移，而不是使用可支付函数。当然，这取决于合约的其余架构。

**AAVE 中的存款示例**

```
contract AddLiquidity{

    receive() external payable {
      IWETH(weth).deposit{msg.value}();
      AAVE.deposit(weth, msg.value, msg.sender, REFERRAL_CODE)
    }
}
```

fallback 函数能够接收字节数据，可以使用 abi.decode 进行解析。这是一种替代向存款函数提供参数的方法。

## 3. 在进行跨合约调用以预热存储槽和合约地址时，使用 ERC2930 访问列表交易

访问列表交易允许你预付一些存储和调用操作的 gas 费用，并享受200 gas 折扣。这可以节省进一步状态或存储访问的 gas 费用，这些费用将作为热访问支付。

如果你的交易将进行跨合约调用，几乎肯定应该使用访问列表交易。

当调用克隆或代理时，这总是涉及通过 delegatecall 进行跨合约调用时，应将交易设置为访问列表交易。

我们在这方面有一篇专门的博文，访问 [https://www.rareskills.io/post/eip-2930-optional-access-list-ethereum](https://www.rareskills.io/post/eip-2930-optional-access-list-ethereum) 了解更多信息。

## 4. 在有意义的情况下缓存对外部合约的调用（例如缓存来自 Chainlink Oracle 的返回数据）

通常建议缓存数据，以避免在单个执行过程中多次使用相同的数据时出现内存重复。

明显的例子是，如果你需要进行多个操作，比如使用从 Chainlink 获取的 ETH 价格。你可以将价格存储在内存中，而不是再次进行昂贵的外部调用。

## 5. 在类似路由器的合约中实现 multicall

这是一种常见的功能，例如 Uniswap 路由器和 Compound Bulker。

如果你希望用户进行一系列的调用，请使用 multicall 将它们批量处理在一起。

## 6. 通过使架构成为单体结构来避免合约调用

合约调用是昂贵的，节省 gas 的最佳方法是根本不使用它们。这其中存在一种自然的权衡，但是拥有多个相互通信的合约有时可能会增加 gas 和复杂性，而不是管理它们。