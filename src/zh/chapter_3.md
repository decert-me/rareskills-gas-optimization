# 跨合约调用

- [1. 使用转账钩子来处理代币，而不是从目标智能合约发起转账](#1-use-transfer-hooks-for-tokens-instead-of-initiating-a-transfer-from-the-destination-smart-contract)
- [2. 在转移以太币时，使用fallback或receive而不是deposit()](#2-use-fallback-or-receive-instead-of-deposit-when-transferring-ether)
- [3. 在进行跨合约调用时，使用ERC2930访问列表事务来预热存储槽和合约地址](#3-use-erc2930-access-list-transactions-when-making-cross-contract-calls-to-pre-warm-storage-slots-and-contract-addresses)
- [4. 在有意义的情况下缓存对外部合约的调用（例如缓存来自Chainlink Oracle的返回数据）](#4-cache-calls-to-external-contracts-where-it-makes-sense-like-caching-return-data-from-chainlink-oracle)
- [5. 在类似路由器的合约中实现multicall](#5-implement-multicall-in-router-like-contracts)
- [6. 通过构建单体架构来避免合约调用](#6-avoid-contract-calls-by-making-the-architecture-monolithic)


## 1. 使用转账钩子来处理代币，而不是从目标智能合约发起转账

假设您有一个合约A，它接受代币B（一个NFT或一个ERC1363代币）。简单的工作流程如下：

1. msg.sender批准合约A接受代币B
2. msg.sender调用合约A将代币从msg.sender转移到A
3. 合约A然后调用代币B进行转移
4. 代币B执行转移，并在合约A中调用onTokenReceived()
5. 合约A从onTokenReceived()返回一个值给代币B
6. 代币B将执行返回给合约A

这种方式非常低效。更好的方式是让msg.sender调用合约B进行转账，合约B再调用合约A中的tokenReceived钩子。

请注意：

- 所有ERC1155代币都包含一个转账钩子
- ERC721中的safeTransfer和safeMint具有转账钩子
- ERC1363具有transferAndCall
- ERC777具有转账钩子，但已被弃用。如果您需要可替代的代币，请使用ERC1363或ERC1155

如果您需要向合约A传递参数，只需使用data字段，并在合约A中解析该字段。

## 2. 在转移以太币时，使用fallback或receive代替deposit()

与上述类似，您可以“只是转移”以太币给合约，并让其响应转移，而不是使用可支付函数。当然，这取决于合约的其余架构。

**AAVE中的存款示例**

```
contract AddLiquidity{

    receive() external payable {
      IWETH(weth).deposit{msg.value}();
      AAVE.deposit(weth, msg.value, msg.sender, REFERRAL_CODE)
    }
}
```

fallback函数能够接收字节数据，可以使用abi.decode进行解析。这是一种替代向存款函数提供参数的方法。

## 3. 在进行跨合约调用以预热存储槽和合约地址时，使用ERC2930访问列表交易

访问列表交易允许您预付一些存储和调用操作的燃气费用，并享受200燃气折扣。这可以节省进一步状态或存储访问的燃气费用，这些费用将作为热访问支付。

如果您的交易将进行跨合约调用，几乎肯定应该使用访问列表交易。

当调用克隆或代理时，这总是涉及通过delegatecall进行跨合约调用时，应将交易设置为访问列表交易。

我们在这方面有一篇专门的博文，访问 https://www.rareskills.io/post/eip-2930-optional-access-list-ethereum 了解更多信息。

## 4. 在有意义的情况下缓存对外部合约的调用（例如缓存来自Chainlink Oracle的返回数据）

通常建议缓存数据，以避免在单个执行过程中多次使用相同的数据时出现内存重复。

明显的例子是，如果您需要进行多个操作，比如使用从Chainlink获取的ETH价格。您可以将价格存储在内存中，而不是再次进行昂贵的外部调用。

## 5. 在类似路由器的合约中实现multicall

这是一种常见的功能，例如Uniswap路由器和Compound Bulker。

如果您希望用户进行一系列的调用，请使用multicall将它们批量处理在一起。

## 6. 通过使架构成为单体结构来避免合约调用

合约调用是昂贵的，节省燃气的最佳方法是根本不使用它们。这其中存在一种自然的权衡，但是拥有多个相互通信的合约有时可能会增加燃气和复杂性，而不是管理它们。

I'm sorry, but you haven't provided the Markdown content that needs to be translated. Please paste the Markdown content here so that I can proceed with the translation.