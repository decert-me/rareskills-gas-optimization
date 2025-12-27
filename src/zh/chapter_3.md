
## 1. 使用转账 Hook 来处理代币

如果可以使用转账 Hook 来处理代币，而不是从目标智能合约发起Token转账

假设你有一个合约 A，它接受代币 B（一个 [NFT](https://learnblockchain.cn/tags/NFT) 或一个 [ERC1363代币](https://learnblockchain.cn/article/8549)）。

通常的授权下的工作流程如：

1. msg.sender 批准合约 A 接受代币 B
2. msg.sender 调用合约 A 将代币从 msg.sender 转移到 A
3. 合约 A 然后调用代币 B 进行转移
4. 代币 B 执行转移，并在合约 A 中调用 onTokenReceived()
5. 合约 A 从 onTokenReceived() 返回一个值给代币 B
6. 代币 B 将执行返回给合约 A

这种方式非常低效。更好的方式是让 msg.sender 调用代币 B 进行转账，代币 B 再调用合约 A 中的 tokenReceived 钩子。

请注意：

- 所有 ERC1155 代币都包含一个转账钩子
- [ERC721](https://learnblockchain.cn/tags/ERC721?map=EVM) 中的 safeTransfer 和 safeMint 具有转账钩子
- ERC1363 具有 transferAndCall
- ERC777 具有转账钩子，但已被弃用。如果你需要可替代的代币，请使用 [ERC1363](https://learnblockchain.cn/article/8549) 或 ERC1155

如果你需要向合约 A 传递参数，只需使用 data 字段，并在合约 A 中解析该字段。

## 2. 仅转移以太币时，使用 receive 代替 deposit()

与上述类似，你可以在“仅转移”以太币给合约，并让其响应转移，而不需要使用可支付函数。当然，这取决于合约的其余架构。

**[AAVE](https://learnblockchain.cn/tags/AAVE?map=EVM) 中的存款示例**

```
contract AddLiquidity{

    receive() external payable {
      IWETH(weth).deposit{msg.value}();
      AAVE.deposit(weth, msg.value, msg.sender, REFERRAL_CODE)
    }
}
```

fallback 函数能够接收字节数据，可以使用 abi.decode 进行解析。这是一种替代向存款函数提供参数的方法。

## 3. 预热存储槽

在进行跨合约调用以预热存储槽和合约地址时，如果事先知道将访问的地址 / slot，可以考虑使用 ERC2930 访问列表交易。

ERC-2930 访问列表允许交易发起者在交易开始前声明将访问的账户地址和存储槽，从而避免冷访问带来的额外 gas 成本。

在能够准确预测访问目标、且存在多次或深层状态访问的复杂交易中，访问列表可能带来可观的 gas 优化。

要注意：不过由于 access list 本身也会带来增加 calldata / tx size 成本的增加，对于大多数跨合约调用，尤其是基于 proxy 或 clone 的 delegatecall 模式，访问列表通常不会带来显著收益，也不应被默认使用。

EIP-2930 的细节可访问 [EIP-2930 – 以太坊访问列表
](https://learnblockchain.cn/article/11283) 

## 4. 缓存调用结果

在有意义的情况下缓存对外部合约的调用（例如缓存来自 Chainlink Oracle 的返回数据）以避免在单个执行过程中多次使用相同的数据时出现内存重复。

明显的例子是，如果你需要进行多个操作，比如使用从 [Chainlink](https://learnblockchain.cn/tags/Chainlink) 获取的 ETH 价格。你可以将价格存储在内存中，而不是再次进行昂贵的外部调用。

## 5. 在类似路由器的合约中实现 multicall

这是一种常见的功能，例如 [Uniswap](https://learnblockchain.cn/tags/Uniswap?map=EVM) 路由器和 Compound Bulker。

如果你希望用户进行一系列的调用，请使用 [multicall](https://learnblockchain.cn/article/22679) 将它们批量处理在一起。

## 6. 使用单体合约来避免合约间调用

在 EVM 中，每一次跨合约调用都存在不可避免的固定 gas 成本。因此，在极端性能敏感的执行路径中，不必要的合约拆分可能会引入额外的 gas 开销和系统复杂性。

同时我们也要了解优化是权衡：在现实合约中，架构合理性、安全隔离和可维护性远比减少一次合约调用更重要；是否采用单体或模块化设计，应基于具体使用场景权衡。