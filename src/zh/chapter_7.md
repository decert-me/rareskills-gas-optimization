# Solidity 编译器相关

- [1. 更喜欢使用严格不等式而不是非严格不等式，但是测试两种选择](#1-更喜欢使用严格不等式而不是非严格不等式但要测试两种选择)
- [2. 将具有布尔表达式的 require 语句拆分](#2-将具有布尔表达式的-require-语句拆分)
- [3. 拆分 revert 语句](#3-拆分-revert-语句)
- [4. 始终使用命名返回](#4-始终使用命名返回)
- [5. 反转具有否定的 if-else 语句](#5-反转具有否定的-if-else-语句)
- [6. 使用 ++i 而不是 i++ 进行递增](#6-使用-i-而不是-i-进行递增)
- [7. 在适当的情况下使用未检查的数学运算](#7-在适当的情况下使用无溢出的数学运算)
- [8. 编写 gas 优化的 for 循环](#8-编写最优化的-gas-for-循环)
- [9. Do-While 循环比 for 循环更便宜](#9-do-while-循环比-for-循环更省-gas)
- [10. 避免不必要的变量转换，小于 uint256 的变量（包括布尔值和地址）效率较低，除非打包](#10-避免不必要的变量转换小于-uint256-的变量包括布尔值和地址效率较低除非进行打包)
- [11. 短路布尔运算](#11-短路布尔运算)
- [12. 除非有必要，否则不要将变量设为 public](#12-除非必要不要将变量设为公开)
- [13. 优化器的值应尽量设置得很大](#13-优化器应选择非常大的值)
- [14. 频繁使用的函数应具有最佳名称](#14-频繁使用的函数应具有最佳名称)
- [15. 位移比乘法或除法更便宜，特别是 2 的幂次方](#15-位移比乘法或除法更省-gas)
- [16. 有时缓存 calldata 更便宜](#16-有时缓存-calldata-更便宜)
- [17. 使用无分支算法替代条件语句和循环](#17-使用无分支算法替代条件语句和循环)
- [18. 只使用一次的内部函数可以内联以节省 gas ](#18-只使用一次的内部函数可以内联以节省-gas)
- [19. 如果数组或字符串长度超过 32 字节，则通过哈希比较它们的相等性](#19-如果数组或字符串的长度超过-32-字节则通过哈希比较它们的相等性)
- [20. 在计算幂和对数时使用查找表](#20-在计算幂和对数时使用查找表)
- [预编译合约可能对某些乘法或内存操作有用。](#预编译合约可能对一些乘法或内存操作有用)

以下的技巧被认为可以提高 Solidity 编译器的 gas 效率。然而，预计随着时间的推移，Solidity 编译器会不断改进，使得这些技巧变得不那么有用甚至适得其反。

你不应该盲目地使用这里列出的技巧，而是应该对两种选择进行基准测试。

当使用 `--via-ir` 编译器标志时，编译器已经将其中一些技巧纳入考虑，但在使用该标志时，这些技巧甚至可能使代码效率降低。

进行基准测试。始终进行基准测试。


## 1. 更喜欢使用严格不等式而不是非严格不等式，但要测试两种选择

通常建议使用严格不等式(<, >)而不是非严格不等式(<=, >=)。这是因为编译器有时会将 a > b 改为 !(a < b) 来实现非严格不等式。EVM 没有用于检查小于等于或大于等于的操作码。

然而，你应该尝试两种比较，因为并不总是使用严格不等式会节省 gas 。这在很大程度上取决于周围操作码的上下文。

## 2. 将具有布尔表达式的 require 语句拆分

当我们拆分 require 语句时，实际上是在说每个语句必须为真，函数才能继续执行。

如果第一个语句计算结果为 false，函数将立即回滚，后续的 require 语句将不会被检查。这将节省 gas 成本，而不是评估下一个 require 语句。

```
// SPDX-License-Identifier: MIT
pragma solidity 0.8.20;

contract Require {
    function dontSplitRequireStatement(uint256 x, uint256 y) external pure returns (uint256) {
        require(x > 0 && y > 0); // both conditon would be evaluated, before reverting or notreturn x * y;
    }
}

contract RequireTwo {
    function splitRequireStatement(uint256 x, uint256 y) external pure returns (uint256) {
        require(x > 0); // if x <= 0, the call reverts and "y > 0" is not checked. 
        require(y > 0);

        return x * y;
    }
}
```

## 3. 拆分 revert 语句

与拆分 require 语句类似，通过在 if 语句中不使用布尔运算符，通常可以节省一些 gas 。

```
contract CustomErrorBoolLessEfficient {
    error BadValue();

    function requireGood(uint256 x) external pure {
        if (x < 10 || x > 20) {
            revert BadValue();
        }
    }
}

contract CustomErrorBoolEfficient {
    error TooLow();
    error TooHigh();

    function requireGood(uint256 x) external pure {
        if (x < 10) {
            revert TooLow();
        }
        if (x > 20) {
            revert TooHigh();
        }
    }
}
```

## 4. 始终使用命名返回

当变量在返回语句中声明时，Solidity 编译器会输出更高效的代码。实际上，很少有例外情况，所以如果你看到一个匿名返回，你应该尝试使用命名返回来确定哪种情况更高效。

```
// SPDX-License-Identifier: MIT
pragma solidity 0.8.20;

contract NamedReturn {
    function myFunc1(uint256 x, uint256 y) external pure returns (uint256) {
        require(x > 0);
        require(y > 0);

        return x * y;
    }
}

contract NamedReturn2 {
    function myFunc2(uint256 x, uint256 y) external pure returns (uint256 z) {
        require(x > 0);
        require(y > 0);

        z = x * y;
    }
}
```

## 5. 反转具有否定的 if-else 语句

这是我们在文章开头给出的相同示例。在下面的代码片段中，第二个函数避免了不必要的否定。理论上，额外的 ! 会增加计算成本。但正如我们在文章开头提到的，你应该对这两种方法进行基准测试，因为编译器有时可以对此进行优化。

```
function cond() public {
    if (!condition) {
        action1();
    }
    else {
        action2();
    }
}

function cond() public {
    if (condition) {
        action2();
    }
    else {
        action1();
    }
}
```

## 6. 使用 ++i 而不是 i++ 进行递增

这是因为编译器对 ++i 和 i++ 的评估方式不同。

i++ 在递增 i 到新值之前返回 i（即它的旧值）。这意味着无论你是否希望使用它，都会在堆栈上存储两个值供使用。而 ++i 则在 i 上评估 ++ 操作（即递增 i），然后返回 i（即它的递增值），这意味着只需要在堆栈上存储一个项目。

## 7. 在适当的情况下使用无溢出的数学运算

Solidity 默认使用有溢出检查的数学运算（即如果数学运算的结果超出结果变量的类型，则回滚），但有些情况下溢出是不可行的。

- 具有自然上限的 for 循环
- 输入函数的数学运算已经被限制在合理范围内
- 变量从一个较小的数字开始，然后每个事务都会增加一个或多个数字（例如计数器）

每当你在代码中看到算术运算时，问问自己是否在上下文中存在自然的溢出或下溢的保护（还要记住保存数字的变量的类型）。如果有，添加一个无溢出的代码块。

## 8. 编写最优化的 gas-for 循环

如果将上述两个技巧结合起来，这就是一个最优化的 gas-for 循环的样子：

```
for (uint256 i; i < limit; ) {
    
    // inside the loop
    
    unchecked {
        ++i;
    }
}
```

与传统的 for 循环相比，这里有两个不同之处：i++ 变成了 ++i（如上所述），并且它是无溢出的，因为限制变量确保它不会溢出。

## 9. do-while 循环比 for 循环更省 gas

如果你想以优化为代价创建稍微不常规的代码，Solidity 的 do-while 循环比 for 循环更节省 gas，即使你为循环不执行的情况添加了 if 条件检查。

```
// SPDX-License-Identifier: MIT
pragma solidity 0.8.20;

// times == 10 in both tests
contract Loop1 {
    function loop(uint256 times) public pure {
        for (uint256 i; i < times;) {
            unchecked {
                ++i;
            }
        }
    }
}

contract Loop2 {
    function loop(uint256 times) public pure {
        if (times == 0) {
            return;
        }

        uint256 i;

        do {
            unchecked {
                ++i;
            }
        } while (i < times);
    }
}
```

## 10. 避免不必要的变量转换，小于 uint256 的变量（包括布尔值和地址）效率较低，除非进行打包

在整数方面，最好使用 uint256，除非需要较小的整数。

这是因为当使用较小的整数时，EVM 会自动将其转换为 uint256。这个转换过程会增加额外的 gas 成本，因此最好从一开始就使用 uint256。

```
// SPDX-License-Identifier: MIT
pragma solidity 0.8.20;

contract Unnecessary_Typecasting {
    uint8 public num;

    function incrementNum() public {
        num += 1;
    }
}

// Uses less gas
contract NoTypecasting {
    uint256 public num;

    function incrementNumCheap() public {
        num += 1;
    }
}
```

## 11. 短路布尔运算

在 Solidity 中，当你评估一个布尔表达式（例如 ||（逻辑或）或 &&（逻辑与）运算符）时，在 || 的情况下，只有在第一个表达式评估为 false 时才会评估第二个表达式，在 && 的情况下，只有在第一个表达式评估为 true 时才会评估第二个表达式。这被称为短路。

例如，表达式 require(msg.sender == owner || msg.sender == manager)将在第一个表达式 msg.sender == owner 评估为 true 时通过。第二个表达式 msg.sender == manager 根本不会被评估。

然而，如果第一个表达式 msg.sender == owner 评估为 false，则第二个表达式 msg.sender == manager 将被评估以确定整个表达式是 true 还是 false。在这里，通过首先检查最有可能通过的条件，我们可以避免检查第二个条件，从而在大多数成功的调用中节省 gas 。

对于表达式 require(msg.sender == owner && msg.sender == manager) 也是类似的。如果第一个表达式 msg.sender == owner 评估为 false，则不会评估第二个表达式 msg.sender == manager，因为整个表达式不能为 true。为了使整个语句为 true，两边的表达式都必须评估为 true。在这里，通过首先检查最有可能失败的条件，我们可以避免检查第二个条件，从而在大多数调用失败时节省 gas 。

短路运算很有用，建议将较便宜的表达式放在前面，因为较昂贵的表达式可能会被绕过。如果第二个表达式比第一个更重要，可能值得颠倒它们的顺序，以便先评估更便宜的表达式。

## 12. 除非必要，不要将变量设为公开

公开的存储变量会有一个同名的隐式公开函数。公开函数会增加跳转表的大小，并添加字节码以读取相关变量。这会使合约变得更大。

请记住，私有变量并不是真正的私有，使用 [web3.js](https://www.rareskills.io/post/web3-js-tutorial) 很容易提取变量的值。

对于常量来说尤其如此，它们是供人类阅读而不是供智能合约使用的。


## 13. 优化器应选择非常大的值

Solidity 优化器主要关注两个方面的优化：

1. 智能合约的部署成本。
2. 智能合约内部函数的执行成本。

在选择优化器的 `runs` 参数时需要权衡利弊。

较小的 `runs` 值优先考虑最小化部署成本，从而得到较小的创建代码，但可能会导致未经优化的运行时代码。虽然这会减少部署时的 gas 成本，但在执行时可能不够高效。

相反，较大的 `runs` 参数值优先考虑执行成本。这会导致较大的创建代码，但会得到经过优化的运行时代码，执行成本更低。虽然这可能不会显著影响部署时的 gas 成本，但在执行时可以显著降低 gas 成本。

考虑到这种权衡，如果你的合约将经常使用，建议选择较大的优化器值。这将在长期节省 gas 成本。


## 14. 频繁使用的函数应具有最佳名称

EVM 在函数调用时使用跳转表，并且具有较低十六进制顺序的函数选择器会优先排序，而不是具有较高十六进制顺序的选择器。换句话说，如果同一个合约中存在两个函数选择器，例如 0x000071c3 和 0xa0712d68，那么在合约执行时，具有选择器 0x000071c3 的函数将在具有选择器 0xa0712d68 的函数之前被检查。

因此，如果一个函数经常被使用，它必须具有最佳名称。这种优化会增加它被首先排序的机会，从而节省进一步检查的 gas 成本（尽管如果合约中的函数超过四个，EVM 会使用二分搜索而不是线性搜索来查找跳转表）。

这还可以减少 calldata 的成本（如果函数有前导零，因为零字节的成本为4 gas，非零字节的成本为16 gas）。

下面是一个很好的演示。

```
// SPDX-License-Identifier: MIT
pragma solidity 0.8.20;

contract FunctionWithLeadingZeros {
    uint256 public totalSupply;
    
    // selector = 0xa0712d68
    function mint(uint256 amount) public {
        totalSupply += amount;
    }
    
    // selector = 0x000071c3 (this cheaper than the above function)
    function mint_184E17(uint256 amount) public {
        totalSupply += amount;
    }
}
```

此外，我们还有一个有用的工具，名为 Solidity Zero Finder，它是用 Rust 构建的，可以帮助开发人员实现这一点。它可以在这个 [GitHub 存储库](https://github.com/jeffreyscholz/solidity-zero-finder-rust)中找到。

## 15. 位移比乘法或除法更省 gas

在 Solidity 中，通过位移操作来乘以或除以二的幂次方的数字通常比使用乘法或除法运算符更节省 gas。

例如，下面两个表达式是等价的：

```
10 * 2
10 << 1 # 将10左移1位
```

这也是等价的：

```
8 / 4
8 >> 2 # 将8右移2位
```

EVM 中的位移操作码（如 shr（右移）和 shl（左移））的成本为5 gas，而乘法和除法操作（mul 和 div）的成本为每个3 gas。

大部分的节省来自于 Solidity 不会对 shr 和 shl 操作进行溢出/下溢或除零检查。在使用这些运算符时，要牢记这一点，以避免发生溢出和下溢的错误。

## 16. 有时缓存 calldata 更便宜

尽管 calldataload 指令是一个廉价的操作码，但 solidity 编译器有时会输出更便宜的代码，如果你缓存 calldataload。这并不总是这样，所以你应该测试两种可能性。

```
contract LoopSum {
    function sumArr(uint256[] calldata arr) public pure returns (uint256 sum) {
        uint256 len = arr.length;
        for (uint256 i = 0; i < len; ) {
            sum += arr[i];
            unchecked {
                ++i;
            }
        }
    }
}
```

## 17. 使用无分支算法替代条件语句和循环

前面一节中的 max 代码是无分支算法的一个例子，即它消除了 `JUMP` 操作码，而 `JUMP` 操作码通常比算术操作码更昂贵。

循环中内置了跳转，因此你可能希望考虑[展开循环](https://en.wikipedia.org/wiki/Loop_unrolling)以节省 gas。

循环不必完全展开。例如，你可以一次执行两个项目的循环，并将跳转次数减半。

这是一个非常极端的优化，但你应该知道条件跳转和循环会引入稍微昂贵的操作码。

## 18. 只使用一次的内部函数可以内联以节省 gas

使用内部函数是可以的，但它们会在字节码中引入额外的跳转标签。

因此，在只被一个函数使用的情况下，最好将内部函数的逻辑内联到使用它的函数中。这样可以通过避免函数执行期间的跳转来节省一些 gas。

## 19. 如果数组或字符串的长度超过 32 字节，则通过哈希比较它们的相等性

这是一个你很少使用的技巧，但是循环遍历数组或字符串比哈希它们并比较哈希值要昂贵得多。

## 20. 在计算幂和对数时使用查找表

如果需要计算底数或指数为分数的对数或幂，如果底数或指数是固定的，可以预先计算一个查找表。

考虑 [Bancor Formula](https://github.com/AragonBlack/fundraising/blob/master/apps/bancor-formula/contracts/BancorFormula.sol#L293) 和 [Uniswap V3 Tick Math](https://github.com/Uniswap/v3-core/blob/main/contracts/libraries/TickMath.sol#L23) 这些例子。

## 预编译合约可能对一些乘法或内存操作有用。

[Ethereum 预编译合约](https://www.rareskills.io/post/solidity-precompiles) 主要提供密码学操作，但如果你需要在模数上乘以大数或复制大块内存，请考虑使用预编译合约。请注意，这可能会导致你的应用与某些 L2 不兼容。
