# 危险技术

- [1. 使用 gasprice() 或 msg.value 传递信息](#1-use-gasprice-or-msgvalue-to-pass-information)
- [2. 操纵环境变量，如 coinbase() 或 block.number（如果测试允许）](#2-manipulate-environment-variables-like-coinbase-or-blocknumber-if-the-tests-allow-it)
- [3. 使用 gasleft() 在关键点进行分支决策](#3-use-gasleft-to-branch-decisions-at-key-points)
- [4. 使用 send() 转移以太币，但不检查成功与否](#4-use-send-to-move-ether-but-dont-check-for-success)
- [5. 使所有函数可支付](#5-make-all-functions-payable)
- [6. 外部库跳转](#6-external-library-jumping)
- [7. 在合约末尾添加字节码以创建高度优化的子程序](#7-append-bytecode-to-the-end-of-the-contract-to-create-a-highly-optimized-subroutine)

如果您参加了一个燃气优化竞赛，那么这些不寻常的设计模式可以帮助您，但在生产环境中使用它们是极不推荐的，或者至少应该极度谨慎。

## 1. 使用 gasprice() 或 msg.value 传递信息

将参数传递给函数至少会增加 128 gas，因为每个 calldata 的零字节都会消耗 4 gas。然而，您可以免费设置 gasprice 或 msg.value 来传递数字。当然，在生产环境中这是行不通的，因为 msg.value 需要真正的以太币，如果您的燃气价格太低，交易将无法进行，或者会浪费加密货币。

## 2. 操纵环境变量，如 coinbase() 或 block.number（如果测试允许）

当然，在生产环境中这是行不通的，但它可以作为一种侧信道来修改智能合约的行为。

## 3. 使用 gasleft() 在关键点进行分支决策

随着执行的进行，燃气会被消耗掉，因此如果您想在某个特定点之后终止循环或在执行的后面部分更改行为，您可以使用 gasprice() 功能来进行分支决策。gasleft() 的减量是“免费”的，因此可以节省燃气。

## 4. 使用 send() 转移以太币，但不检查成功与否

send和transfer之间的区别在于，如果转账失败，transfer会回滚，而send会返回false。然而，你可以忽略send的返回值，这样可以减少操作码的数量。忽略返回值是一种非常糟糕的做法，遗憾的是编译器并不会阻止你这样做。在生产系统中，由于燃气限制，你根本不应该使用send()。

## 5. 将所有函数设为可支付函数

这是一种有争议的优化方式，因为它允许在交易中发生意外的状态变化，并且并不能节省太多的燃气。但在燃气竞赛的背景下，将所有函数设为可支付函数可以避免额外的操作码来检查msg.value是否为非零值。

正如前面所提到的，将构造函数或管理员函数设为可支付函数是一种合理的节省燃气的方式，因为部署者和管理员应该知道自己在做什么，并且可以执行比发送以太币更具破坏性的操作。

## 6. 外部库跳转

Solidity传统上使用4个字节和跳转表来确定要使用的函数。然而，你可以（非常不安全地！）将跳转目标作为calldata参数提供，将“函数选择器”减少到一个字节，并完全避免跳转表。更多信息可以在这个[tweet](https://twitter.com/AmadiMichaels/status/1697405235948310627)中看到。

## 7. 在合约末尾添加字节码以创建高度优化的子程序

一些计算密集型算法，例如哈希函数，最好使用原始字节码而不是Solidity甚至Yul来编写。例如，[Tornado Cash](https://www.rareskills.io/post/how-does-tornado-cash-work)将MiMC哈希函数作为一个单独的智能合约，直接以原始字节码编写。通过将该字节码附加到实际合约并在它之间来回跳转，可以避免另一个智能合约的额外2600或100燃气成本（冷或热访问）。这里有一个使用Huff的[概念验证](https://twitter.com/AmadiMichaels/status/1696263027920634044)。
