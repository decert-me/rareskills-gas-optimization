# 概述

Gas 优化手册由 [RareSkills](https://www.rareskills.io/) 编写，[DeCert.me](https://decert.me) 翻译为中文版。

## Solidity gas 优化

在以太坊中，gas 优化是指重写 Solidity 代码，以在以太坊虚拟机（EVM）中消耗更少的 gas 单位，同时实现相同的业务逻辑。

这篇文章超过11,000个字，不包括源代码，是关于 gas 优化的最完整的资料。

要完全理解本教程中的技巧，你需要了解 EVM 的工作原理，你可以通过参加 Rareskills 的[ gas 优化课程](https://hackmd.io/eQJUW4PLQN-6HRrgyxdhsQ?view)，[Yul 课程](https://hackmd.io/eQJUW4PLQN-6HRrgyxdhsQ?view)，以及练习 [Huff Puzzles](https://github.com/RareSkills/huff-puzzles) 来学习。

然而，如果你只想知道代码中可能进行 gas 优化的部分，本文提供了很多可以参考的领域。

## 著作权

RareSkills 研究员 Michael Amadi（[*LinkedIn*](https://www.linkedin.com/in/michael-amadi-2aa2ab23b/)，[*Twitter*](https://twitter.com/@AmadiMichaels)）和 Jesse Raymond（[*LinkedIn*](https://www.linkedin.com/in/jesse-raymond4/)，[*Twitter*](https://twitter.com/Jesserc_)）对此工作做出了重要贡献。