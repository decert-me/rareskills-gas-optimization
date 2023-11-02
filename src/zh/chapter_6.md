# 汇编技巧

- [1. 使用汇编来回滚并附带错误消息](#1-使用汇编来回滚并附带错误消息)
- [2. 通过接口调用函数会产生内存扩展成本，因此使用汇编来重用已经存在于内存中的数据](#2-通过接口调用函数会产生内存扩展成本因此使用汇编来重用已存在于内存中的数据)
- [3. 常见的数学运算，如最小值和最大值，有更节省 gas 的替代方法](#3-常见的数学运算如最小值和最大值有更节省-gas-的替代方法)
- [4. 使用 SUB 或 XOR 而不是 ISZERO(EQ())来检查不等式（在某些情况下更高效）](#4-使用-sub-或-xor-而不是-iszeroeq-来检查不等式在某些情况下更高效)
- [5. 使用内联汇编来检查地址是否为0](#5-使用内联汇编来检查-address0)
- [6. selfbalance 比 address(this).balance 更便宜（在某些情况下更高效）](#6-selfbalance-比-addressthisbalance-更便宜在某些情况下更高效)
- [7. 使用汇编来处理大小为96字节或更小的数据：哈希和事件中的非索引数据](#7-使用汇编来处理大小为96字节或更小的数据哈希和事件中的非索引数据)
- [8. 在进行多个外部调用时，使用汇编来重用内存空间](#8-使用汇编在进行多个外部调用时重用内存空间)
- [9. 在创建多个合约时，使用汇编来重用内存空间](#9-使用汇编在创建多个合约时重用内存空间)
- [10. 通过检查最后一位而不是使用模运算符来测试一个数是偶数还是奇数](#10-通过检查最后一位而不是使用模运算符来测试一个数是偶数还是奇数)




不要假设编写汇编代码会自动导致更高效的代码。我们列出了编写汇编通常效果更好的领域，但你应始终测试非汇编版本。

## 1. 使用汇编来回滚并附带错误消息

在 Solidity 代码中进行回滚时，通常使用 require 或 revert 语句来回滚执行并附带错误消息。通过使用汇编来回滚并附带错误消息，可以进一步优化。

以下是一个示例：
```
/// calling restrictedAction(2) with a non-owner address: 24042
contract SolidityRevert {
    address owner;
    uint256 specialNumber = 1;

    constructor() {
        owner = msg.sender;
    }

    function restrictedAction(uint256 num)  external {
        require(owner == msg.sender, "caller is not owner");
        specialNumber = num;
    }
}

/// calling restrictedAction(2) with a non-owner address: 23734
contract AssemblyRevert {
    address owner;
    uint256 specialNumber = 1;

    constructor() {
        owner = msg.sender;
    }

    function restrictedAction(uint256 num)  external {
        assembly {
            if sub(caller(), sload(owner.slot)) {
                mstore(0x00, 0x20) // store offset to where length of revert message is stored
                mstore(0x20, 0x13) // store length (19)
                mstore(0x40, 0x63616c6c6572206973206e6f74206f776e657200000000000000000000000000) // store hex representation of message
                revert(0x00, 0x60) // revert with data
            }
        }
        specialNumber = num;
    }
}
```
从上面的示例中，我们可以看到，使用汇编与使用 Solidity 相比，在出现相同错误消息时，使用汇编进行回滚可以节省超过300个 gas。这种节省是由于 Solidity 编译器在内部执行的内存扩展成本和额外类型检查所导致的。

## 2. 通过接口调用函数会产生内存扩展成本，因此使用汇编来重用已存在于内存中的数据

当从合约 A 调用合约 B 的函数时，使用接口并创建一个 B 的实例，并调用我们希望调用的函数是最方便的。这种方法非常有效，但由于 Solidity 编译我们的代码的方式，它会将要发送给合约 B 的数据存储在一个新的内存位置中，从而扩展内存，有时是不必要的。

通过内联汇编，我们可以更好地优化我们的代码，并通过使用先前使用过的不再需要的内存位置或（如果 calldata 合约 B 期望的字节数小于64字节）在临时空间中存储我们的 calldata 来节省一些 gas。

下面是一个比较两者的示例：

```
/// 30570
contract Sol {
    function set(address addr, uint256 num) external {
        Callme(addr).setNum(num);
    }
}

/// 30350
contract Assembly {
    function set(address addr, uint256 num) external {
        assembly {
            mstore(0x00, hex"cd16ecbf")
            mstore(0x04, num)

            if iszero(extcodesize(addr)) {
                revert(0x00, 0x00) // revert if address has no code deployed to it
            }

            let success := call(gas(), addr, 0x00, 0x00, 0x24, 0x00, 0x00)
            
            if iszero(success) {
                revert(0x00, 0x00)
            }
        }
    }
}


contract Callme {
    uint256 num = 1;

    function setNum(uint256 a) external {
        num = a;
    }
}
```

我们可以看到，在 Assembly 上调用 set(uint256) 的成本比使用 Solidity 少220个 gas。

请注意，当使用内联汇编进行外部调用时，重要的是使用`extcodesize(addr)`检查我们要调用的地址是否部署了代码，并在返回0时回滚。这很重要，因为调用一个没有部署代码的地址总是返回 true，这在大多数情况下可能对我们的合约逻辑造成灾难性的影响。

## 3. 常见的数学运算，如最小值和最大值，有更节省 gas 的替代方法

**未优化**

```
function max(uint256 x, uint256 y) public pure returns (uint256 z) {
	z = x > y ? x : y;
}
```

**优化后**

```
function max(uint256 x, uint256 y) public pure returns (uint256 z) {
    /// @solidity memory-safe-assembly
    assembly {
        z := xor(x, mul(xor(x, y), gt(y, x)))
    }
}
```

上面的代码摘自 [Solady Library](https://github.com/Vectorized/solady/blob/main/src/utils/FixedPointMathLib.sol) 的数学部分，更多的数学运算可以在那里找到。值得探索该库，看看有哪些节省 gas 的操作可供使用。

上面的示例之所以更节省 gas，是因为三元运算符（以及一般情况下带有条件的代码）在操作码中包含条件跳转，这是更昂贵的操作。

## 4. 使用 SUB 或 XOR 而不是 ISZERO(EQ()) 来检查不等式（在某些情况下更高效）

当使用内联汇编来比较两个值的相等性（例如，如果所有者与调用者相同），有时使用以下方式更高效：

```
if sub(caller, sload(owner.slot)) {
    revert(0x00, 0x00)
}
```

而不是这样做：

```
if eq(caller, sload(owner.slot)) {
    revert(0x00, 0x00)
}
```

XOR 也可以实现相同的效果，但要注意 XOR 也会将所有位翻转的值视为相等，因此请确保这不会成为攻击的入口。

这个技巧取决于所使用的编译器版本和代码的上下文。

## 5. 使用内联汇编来检查 address(0)

使用内联汇编编写合约通常被认为是优化了的。我们可以直接操作内存，使用更少的操作码，而不是交给 Solidity 编译器处理。

身份验证机制是使用内联汇编的一个例子，比如实现地址零检查。

下面是一个示例：

```
// SPDX-License-Identifier: MIT
pragma solidity 0.8.20;

contract NormalAddressZeroCheck {
    function check(address _caller) public pure returns (bool) {
        require(_caller != address(0x00), "Zero address");
        return true;
    }
}

contract AddressZeroCheckAssembly {
    // Saves about 90 gasfunction checkOptimized(address _caller) public pure returns (bool) {
        assembly {
            if iszero(_caller) {
                mstore(0x00, 0x20)
                mstore(0x20, 0x0c)
                mstore(0x40, 0x5a65726f20416464726573730000000000000000000000000000000000000000) // load hex of "Zero Address" to memory
                revert(0x00, 0x60)
            }
        }

        return true;
    }
}
```

## 6. selfbalance 比 address(this).balance 更便宜（在某些情况下更高效）

Solidity 代码中的 address(this).balance 有时可以使用 yul 中的 selfbalance() 函数更高效地完成，但要注意编译器有时会在幕后使用这个技巧，所以请测试两种方式。

## 7. 使用汇编来处理大小为96字节或更小的数据：哈希和事件中的非索引数据

Solidity 总是通过扩展内存来写入，这有时不是高效的。我们可以通过使用内联汇编来优化对大小为96字节或更小的数据的内存操作。

Solidity 将其前64字节的内存（mem[0x00:0x40]）保留为临时空间，开发人员可以使用它来执行任何操作，并保证不会意外地被覆盖或读取。接下来的32字节的内存（mem[0x40:0x60]）是 Solidity 用于存储、读取和更新自由内存指针的位置。这是 Solidity 用于跟踪下一个内存偏移以写入新数据的方式。接下来的32字节的内存（mem[0x60:0x80]）称为零槽。它是未初始化的动态内存数据（bytes memory、string memory、T[] memory（其中 T 是任何有效类型））指向的位置。由于这些值是未初始化的，Solidity 假设它们指向的槽（0x60）保持为0x00。

注意：即使动态结构体（即在自身内部具有动态值）未初始化，它们在内存中仍然不指向零槽。

注意：即使未初始化的动态内存数据嵌套在结构体中，它们仍然指向零槽。

如果我们可以利用临时空间来执行内存操作，而编译器通常会扩展内存来执行这些操作，那么我们可以优化我们的代码。因此，我们现在有64字节的更便宜的内存可供使用。

只要我们在退出汇编块之前更新它，空闲内存指针空间也可以使用。我们可以将其暂时存储在堆栈上。

让我们看一些例子。

- 使用汇编记录最多96字节的非索引数据

```
contract ExpensiveLogger {
    event BlockData(uint256 blockTimestamp, uint256 blockNumber, uint256 blockGasLimit);

    // cost: 22790
    function returnBlockData() external {
        emit BlockData(block.timestamp, block.number, block.gaslimit);
    }
}


contract CheapLogger {
    event BlockData(uint256 blockTimestamp, uint256 blockNumber, uint256 blockGasLimit);

    // cost: 26145
    function returnBlockData() external {
        assembly {
            mstore(0x00, timestamp())
            mstore(0x20, number())
            mstore(0x40, gaslimit())

            log1(0x00, 
                0x60,
                0x9ae98f1999f57fc58c1850d34a78f15d31bee81788521909bea49d7f53ed270b // event hash of BlockData
                )
        }
    }
}
```

上面的示例显示了我们如何通过使用内存来存储我们希望在 BlockData 事件中发出的数据来节省近2000个 gas。

在这里不需要更新我们的空闲内存指针，因为执行在我们发出事件后立即结束，我们不会再返回到 Solidity 代码中。

让我们再看一个例子，在这个例子中我们需要更新空闲内存指针

- 使用汇编对最多96字节的数据进行哈希计算

```
contract ExpensiveHasher {
    bytes32 public hash;
    struct Values {
        uint256 a;
        uint256 b;
        uint256 c;
    }
    Values values;

    // cost: 113155function setOnchainHash(Values calldata _values) external {
        hash = keccak256(abi.encode(_values));
        values = _values;
    }
}


contract CheapHasher {
    bytes32 public hash;
    struct Values {
        uint256 a;
        uint256 b;
        uint256 c;
    }
    Values values;

    // cost: 112107
    function setOnchainHash(Values calldata _values) external {
        assembly {
            // cache the free memory pointer because we are about to override it 
            let fmp := mload(0x40)
            
            // use 0x00 to 0x60
            calldatacopy(0x00, 0x04, 0x60)
            sstore(hash.slot, keccak256(0x00, 0x60))

            // restore the cache value of free memory pointer
            mstore(0x40, fmp)
        }

        values = _values;
    }
}
```

在上面的示例中，与第一个示例类似，我们使用汇编将值存储在内存的前96字节中，这样可以节省1000多个 gas。还要注意，在这种情况下，因为我们仍然返回到 Solidity 代码中，我们在汇编块的开头和结尾缓存并更新了我们的空闲内存指针。这是为了确保 Solidity 编译器对存储在内存中的内容的假设保持兼容。

## 8. 使用汇编在进行多个外部调用时重用内存空间

导致 Solidity 编译器扩展内存的操作之一是进行外部调用。在进行外部调用时，编译器必须将要调用的外部合约的函数签名以及其参数编码到内存中。正如我们所知，Solidity 不会清除或重用内存，因此它将不得不将这些数据存储在下一个空闲内存指针中，从而进一步扩展内存。

使用内联汇编，我们可以使用临时空间和自由内存指针偏移来存储这些数据（如上所示），如果函数参数在内存中不超过96字节。更好的是，如果我们进行多次外部调用，我们可以重用与第一次调用相同的内存空间，将新的参数存储在内存中，而不会不必要地扩展内存。在这种情况下，Solidity 将根据返回数据的长度扩展内存。这是因为返回的数据通常存储在内存中。如果返回的数据少于96字节，我们可以使用临时空间将其存储起来，以防止扩展内存。

请参考下面的示例：

```
contract Called {
    function add(uint256 a, uint256 b) external pure returns(uint256) {
        return a + b;
    }
}


contract Solidity {
    // cost: 7262
    function call(address calledAddress) external pure returns(uint256) {
        Called called = Called(calledAddress);
        uint256 res1 = called.add(1, 2);
        uint256 res2 = called.add(3, 4);

        uint256 res = res1 + res2;
        return res;
    }
}


contract Assembly {
    // cost: 5281
    function call(address calledAddress) external view returns(uint256) {
        assembly {
            // check that calledAddress has code deployed to it
            if iszero(extcodesize(calledAddress)) {
                revert(0x00, 0x00)
            }

            // first call
            mstore(0x00, hex"771602f7")
            mstore(0x04, 0x01)
            mstore(0x24, 0x02)
            let success := staticcall(gas(), calledAddress, 0x00, 0x44, 0x60, 0x20)
            if iszero(success) {
                revert(0x00, 0x00)
            }
            let res1 := mload(0x60)

            // second call
            mstore(0x04, 0x03)
            mstore(0x24, 0x4)
            success := staticcall(gas(), calledAddress, 0x00, 0x44, 0x60, 0x20)
            if iszero(success) {
                revert(0x00, 0x00)
            }
            let res2 := mload(0x60)

            // add results
            let res := add(res1, res2)

            // return data
            mstore(0x60, res)
            return(0x60, 0x20)
        }
    }
}
```

通过使用临时空间存储函数选择器及其参数，并在第二次调用时重用相同的内存空间，同时将返回的数据存储在零槽中，我们可以节省大约2000个 gas，从而避免扩展内存。

如果你希望调用的外部函数的参数超过64字节，并且只进行一次外部调用，则使用汇编语言编写不会节省任何显著的 gas。然而，如果进行多次调用，你仍然可以通过使用内联汇编将相同的内存槽用于两次调用来节省 gas。

注意：始终记得更新自由内存指针，如果它指向的偏移已经被使用，以避免 Solidity 覆盖存储在那里的数据或以意外的方式使用存储在那里的值。

还要注意，如果在调用堆栈中存在未定义的动态内存值，请避免覆盖零槽（0x60内存偏移）。一个替代方法是显式定义动态内存值，或者在退出汇编块之前将槽设置回0x00。


## 9. 使用汇编在创建多个合约时重用内存空间

Solidity 将合约创建视为返回32字节的外部调用（即返回创建的合约地址，如果合约创建失败，则返回 address(0)）。

从关于通过外部调用节省 gas 的部分，我们可以立即看到一种优化的方法是将返回的地址存储在临时空间中，避免扩展内存。

看下面一个类似的例子：

```
contract Solidity {
    // cost: 261032
    function call() external returns (Called, Called) {
        Called called1 = new Called();
        Called called2 = new Called();
        return (called1, called2);
    }
}


contract Assembly {
    // cost: 260210
    function call() external returns(Called, Called) {
        bytes memory creationCode = type(Called).creationCode;
        assembly {
            let called1 := create(0x00, add(0x20, creationCode), mload(creationCode))
            let called2 := create(0x00, add(0x20, creationCode), mload(creationCode))

            // revert if either called1 or called2 returned address(0)
            if iszero(and(called1, called2)) {
                revert(0x00, 0x00)
            }

            mstore(0x00, called1)
            mstore(0x20, called2)

            return(0x00, 0x40)
        }
    }
}



contract Called {
    function add(uint256 a, uint256 b) external pure returns(uint256) {
        return a + b;
    }
}
```

通过使用内联汇编，我们节省了近1000个 gas。

注意：在两个要部署的合约不相同的情况下，第二个合约的创建代码需要使用内联汇编手动存储，而不是在 Solidity 中分配给一个变量，以避免内存扩展。

## 10. 通过检查最后一位而不是使用模运算符来测试一个数是偶数还是奇数

传统的方法是通过执行 x % 2 == 0 来检查一个数是偶数还是奇数，其中 x 是待检查的数。你可以改为检查 x & uint256(1) == 0。其中 x 假设为 uint256。位与运算比模运算更加高效。在二进制中，最右边的位表示 "1"，而所有位向左都是 2 的倍数，即偶数。将一个偶数加上 "1" 会使其变为奇数。


