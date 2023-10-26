# 设计模式

- [1. 使用 multidelegatecall 批量处理交易](#1-使用-multidelegatecall-批量处理交易)
- [2. 使用 ECDSA 签名替代默克尔树进行 allowlist 和 airdrop](#2-使用-ecdsa-签名替代默克尔树进行-allowlist-和-airdrop)
- [3. 使用 ERC20Permit 在一笔交易中批量处理批准和转账步骤](#3-使用-erc20permit-在一笔交易中批准和转账步骤)
- [4. 对于游戏或其它高吞吐量、低交易价值的应用，使用 L2 消息传递](#4-对于游戏或其它高吞吐量低交易价值的应用使用-l2-消息传递)
- [5. 如适用，使用状态通道](#5-如果适用使用状态通道)
- [6. 使用投票委托作为节省 gas 的措施](#6-使用投票委托作为节省-gas-的措施)
- [7. ERC1155 是比 ERC721 更便宜的非同质化代币](#7-erc1155-是比-erc721-更便宜的非同质化代币)
- [8. 使用一个 ERC1155 或 ERC6909 代币代替多个 ERC20 代币](#8-使用一个-erc1155-或-erc6909-代币代替多个-erc20-代币)
- [9. UUPS 升级模式对用户来说比透明升级代理更节省 gas](#9-uups-升级模式对用户来说比透明升级代理更节省-gas)
- [10. 考虑使用 OpenZeppelin 的替代方案](#10-考虑使用-openzeppelin-的替代方案)


## 1. 使用 multidelegatecall 批量处理交易

multidelegatecall 允许 msg.sender 在一个合约中调用多个函数，同时保留 msg.sender 和 msg.value 等环境变量。

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

## 2. 使用 ECDSA 签名替代默克尔树进行 allowlist 和 airdrop

默克尔树使用大量的 calldata，并且随着默克尔证明的大小增加而增加成本。通常情况下，使用数字签名在 gas 方面比默克尔证明更便宜。

## 3. 使用 ERC20Permit 在一笔交易中批准和转账步骤

ERC20 Permit 有一个额外的函数，可以接受代币持有者的数字签名，以增加对另一个地址的批准。这样，批准的接收者可以提交许可证交易和转账交易，并合并为一笔交易。授予许可证的用户不需要支付任何 gas 费用，而许可证的接收者可以将许可证和 transferFrom 交易合并为一笔交易。

## 4. 对于游戏或其它高吞吐量、低交易价值的应用，使用 L2 消息传递

[Etherorcs](https://github.com/EtherOrcsOfficial/etherOrcs-contracts) 是这种模式的早期先驱之一，因此你可以参考他们的 Github（上面链接的地址）获取灵感。这个想法是，以太坊上的资产可以通过消息传递“桥接”到其它链，如 Polygon、Optimism 或 Arbitrum，并且游戏可以在那里进行，其中交易成本较低。

## 5. 如果适用，使用状态通道

状态通道可能是以太坊最古老但仍可用的可扩展性解决方案。与 L2 不同，它们是应用程序特定的。用户不是将他们的交易提交到链上，而是将资产提交到智能合约，然后彼此共享绑定签名作为状态转换。当操作结束时，他们将最终结果提交到链上。

如果参与者中有人不诚实，那么诚实的参与者可以使用对方的签名来强制智能合约释放他们的资产。

## 6. 使用投票委托作为节省 gas 的措施

我们在 [ERC20 Votes](https://www.rareskills.io/post/erc20-votes-erc5805-and-erc6372) 的教程中更详细地描述了这种模式。与其让每个代币持有者都进行投票，不如只让代表进行投票，从而减少了投票交易的数量。

## 7. ERC1155 是比 ERC721 更便宜的非同质化代币

ERC721 的 balanceOf 函数在实践中很少使用，但在进行铸造和转账时会增加存储开销。ERC1155 按 id 跟踪余额，并使用相同的余额来跟踪 id 的所有权。如果每个 id 的最大供应量为一，则该代币对于每个 id 都是非同质化的。

## 8. 使用一个 ERC1155 或 ERC6909 代币代替多个 ERC20 代币

这是 ERC1155 代币的最初目的。每个单独的代币的行为类似于 ERC20，但只需要部署一个合约。

这种方法的缺点是这些代币将与大多数 DeFi 交换原语不兼容。

ERC1155 在所有转账方法上使用回调。如果不需要这样的功能，可以使用 [ERC6909](https://eips.ethereum.org/EIPS/eip-6909)。

## 9. UUPS 升级模式对用户来说比透明升级代理更节省 gas

透明升级代理模式需要在每次交易发生时从存储槽（由 ERC-1967 定义）中读取管理员地址，以查看调用者是否为管理员。这会引入额外的存储读取。

## 10. 考虑使用 OpenZeppelin 的替代方案

OpenZeppelin 是一个很棒且受欢迎的智能合约库，但还有其它值得考虑的替代方案。这些替代方案提供更好的 gas 效率，并经过开发者的测试和推荐。

其中两个替代方案的例子是 [Solmate](https://github.com/transmissions11/solmate) 和 [Solady](https://github.com/Vectorized/solady)。

Solmate 是一个库，提供了一些常见智能合约模式的高效实现。Solady 是另一个高效的库，强调使用汇编语言。
