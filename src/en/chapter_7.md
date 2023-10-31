# Solidity Compiler Related

- [1. Prefer strict inequalities over non-strict inequalities, but test both alternatives](#1-prefer-strict-inequalities-over-non-strict-inequalities-but-test-both-alternatives)
- [2. Split require statements that have boolean expressions](#2-split-require-statements-that-have-boolean-expressions)
- [3. Split revert statements](#3-split-revert-statements)
- [4. Always use Named Returns](#4-always-use-named-returns)
- [5. Invert if-else statements that have a negation](#5-invert-if-else-statements-that-have-a-negation)
- [6. Use ++i instead of i++ to increment](#6-use-i-instead-of-i-to-increment)
- [7. Use unchecked math where appropriate](#7-use-unchecked-math-where-appropriate)
- [8. Write gas-optimal for-loops](#8-write-gas-optimal-for-loops)
- [9. Do-While loops are cheaper than for loops](#9-do-while-loops-are-cheaper-than-for-loops)
- [10. Avoid Unnecessary Variable Casting, variables smaller than uint256 (including boolean and address) are less efficient unless packed](#10-avoid-unnecessary-variable-casting-variables-smaller-than-uint256-including-boolean-and-address-are-less-efficient-unless-packed)
- [11. Short-circuit booleans](#11-short-circuit-booleans)
- [12. Don’t make variables public unless it is necessary to do so](#12-dont-make-variables-public-unless-it-is-necessary-to-do-so)
- [13. Prefer very large values for the optimizer](#13-prefer-very-large-values-for-the-optimizer)
- [14. Heavily used functions should have optimal names](#14-heavily-used-functions-should-have-optimal-names)
- [15. Bitshifting is cheaper than multiplying or dividing by a power of two](#15-bitshifting-is-cheaper-than-multiplying-or-dividing-by-a-power-of-two)
- [16. It is sometimes cheaper to cache calldata](#16-it-is-sometimes-cheaper-to-cache-calldata)
- [17. Use branchless algorithms as a replacement for conditionals and loops](#17-use-branchless-algorithms-as-a-replacement-for-conditionals-and-loops)
- [18. Internal functions only used once can be inlined to save gas](#18-internal-functions-only-used-once-can-be-inlined-to-save-gas)
- [19. Compare array equality and string equality by hashing them if they are longer than 32 bytes](#19-compare-array-equality-and-string-equality-by-hashing-them-if-they-are-longer-than-32-bytes)
- [20. Use lookup tables when computing powers and logarithms](#20-use-lookup-tables-when-computing-powers-and-logarithms)
- [Precompiled contracts may be useful for some multiplication or memory operations.](#precompiled-contracts-may-be-useful-for-some-multiplication-or-memory-operations)




The following tricks are known to improve gas efficiency in the Solidity compiler. However, it is expected that the Solidity compiler will improve over time making these tricks less useful or even counterproductive.

You shouldn’t blindly use the tricks listed here, but benchmark both alternatives. 

Some of these tricks are already incorporated by the compiler when using the --via-ir compiler flag, and may even make the code less efficient when that flag is used.

Benchmark. Always benchmark.


## 1. Prefer strict inequalities over non-strict inequalities, but test both alternatives

It is generally recommended to use strict inequalities (<, >) over non-strict inequalities (<=, >=). This is because the compiler will sometimes change a > b to be !(a < b) to accomplish the non-strict inequality. The EVM does not have an opcode for checking less-than-or-equal to or greater-than-or-equal to.

However, you should try both comparisons, because it is not always the case that using the strict inequality will save gas. This is very dependent on the context of the surrounding opcodes.

## 2. Split require statements that have boolean expressions

When we split require statements, we are essentially saying that each statement must be true for the function to continue executing.

If the first statement evaluates to false, the function will revert immediately and the following require statements will not be examined. This will save the gas cost rather than evaluating the next require statement.

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

## 3. Split revert statements

Similar to splitting require statements, you will usually save some gas by not having a boolean operator in the if statement.

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

## 4. Always use Named Returns

The solidity compiler outputs more efficient code when the variable is declared in the return statement. There seem to be very few exceptions to this in practice, so if you see an anonymous return, you should test it with a named return instead to determine which case is most efficient.

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

## 5. Invert if-else statements that have a negation

This is the same example we gave at the beginning of the article. In the code snippet below, the second function avoids an unnecessary negation. In theory, the extra ! increases the computational cost. But as we noted at the top of the article, you should benchmark both methods because the compiler is can sometimes optimize this.

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

## 6. Use ++i instead of i++ to increment

The reason behind this is in way ++i and i++ are evaluated by the compiler.

i++ returns i(its old value) before incrementing i to a new value. This means that 2 values are stored on the stack for usage whether you wish to use it or not. ++i on the other hand, evaluates the ++ operation on i (i.e it increments i) then returns i (its incremented value) which means that only one item needs to be stored on the stack.


## 7. Use unchecked math where appropriate

Solidity uses checked math (i.e it reverts if the result of a math operation overflows the type of the result variable) by default, but there are some situations where overflow is infeasible to occur.

- for loops which have natural upper bounds
- math where the input to the function is already sanitized into reasonable ranges
- variables that start at a low number and then each transaction adds one or a small number to it (like a counter)

Whenever you see arithmetic in code, ask yourself if there is a natural guard to overflow or underflow in the context (keep in mind the type of the variable holding the number too). If so, add an unchecked block.


## 8. Write gas-optimal for-loops

This is what a gas-optimal for loop looks like, if you combine the two tricks above:

```
for (uint256 i; i < limit; ) {
    
    // inside the loop
    
    unchecked {
        ++i;
    }
}
```

The two differences here from a conventional for loop is that i++ becomes ++i (as noted above), and it is unchecked because the limit variable ensures it won’t overflow.


## 9. Do-While loops are cheaper than for loops

If you want to push optimization at the expense of creating slightly unconventional code, Solidity do-while loops are more gas efficient than for loops, even if you add an if-condition check for the case where the loop doesn’t execute at all.

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

## 10. Avoid Unnecessary Variable Casting, variables smaller than uint256 (including boolean and address) are less efficient unless packed

It is better to use uint256 for integers, except when smaller integers are necessary.

This is because the EVM automatically converts smaller integers to uint256 when they are used. This conversion process adds extra gas cost, so it is more efficient to use uint256 from the start.

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

## 11. Short-circuit booleans

In Solidity, when you evaluate a boolean expression (e.g the || (logical or) or && (logical and) operators), in the case of || the second expression will only be evaluated if the first expression evaluates to false and in the case of && the second expression will only be evaluated if the first expression evaluates to true. This is called short-circuiting.

For example, the expression require(msg.sender == owner || msg.sender == manager) will pass if the first expression msg.sender == owner evaluates to true. The second expression msg.sender == manager will not be evaluated at all.

However, if the first expression msg.sender == owner evaluates to false, the second expression msg.sender == manager will be evaluated to determine whether the overall expression is true or false. Here, by checking the condition that is most likely to pass firstly, we can avoid checking the second condition thereby saving gas in majority of successful calls.

This is similar for the expression require(msg.sender == owner && msg.sender == manager). If the first expression msg.sender == owner evaluates to false, the second expression msg.sender == manager will not be evaluated because the overall expression cannot be true. For the overall statement to be true, both side of the expression must evaluate to true. Here, by checking the condition that is most likely to fail firstly, we can avoid checking the second condition thereby saving gas in majority of call reverts.

Short-circuiting is useful and it’s recommended to place the less expensive expression first, as the more costly one might be bypassed. If the second expression is more important than the first, it might be worth reversing their order so that the cheaper one gets evaluated first.


## 12. Don’t make variables public unless it is necessary to do so

A public storage variable has an implicit public function of the same name. A public function increases the size of the jump table and adds bytecode to read the variable in question. That makes the contract larger.

Remember, private variables aren’t private, it’s not difficult to extract the variable value using [web3.js](https://www.rareskills.io/post/web3-js-tutorial).

This is especially true for constants which are meant to be read by humans rather than smart contracts.


## 13. Prefer very large values for the optimizer

The Solidity optimizer focuses on optimizing two primary aspects:

1. The deployment cost of a smart contract.
2. The execution cost of functions within the smart contract.

There’s a trade-off involved in selecting the runs parameter for the optimizer.

Smaller run values prioritize minimizing the deployment cost, resulting in smaller creation code but potentially unoptimized runtime code. While this reduces gas costs during deployment, it may not be as efficient during execution.

Conversely, larger values of the runs parameter prioritize the execution cost. This leads to larger creation code but an optimized runtime code that is more cheaper to execute. While this may not significantly affect deployment gas costs, it can significantly reduce gas costs during execution.

Considering this trade-off, if your contract will be used frequently it is advisable to use a larger value for the optimizer. As this will save up gas costs in a long term.


## 14. Heavily used functions should have optimal names

The EVM uses a jump table for function calls, and function selectors with lesser hexadecimal order are sorted first over selectors with higher hex order. In other words, if two function selectors, for example, 0x000071c3 and 0xa0712d68, are present in the same contract, the function with the selector 0x000071c3 will be checked before the one with 0xa0712d68 during contract execution.

Hence, if a function is used frequently, it is essential for it to have an optimal name. This optimization increases its chances of being sorted first, thus saving gas costs from further checks (although if there are more than four functions in the contract, the EVM does a binary search for the jump table instead of a linear search).

This also reduces calldata cost (if the function has leading zeros, as zero bytes cost 4 gas, and non-zero bytes cost 16 gas).

Here is a good demo below.

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

In addition, we have a helpful tool called Solidity Zero Finder which was built with Rust that can assist developers in achieving this. It is available at this [GitHub repository](https://github.com/jeffreyscholz/solidity-zero-finder-rust).

## 15. Bitshifting is cheaper than multiplying or dividing by a power of two

In Solidity, it is often more gas efficient to multiply or divide numbers that are powers of two by shifting their bits, rather than using the multiplication or division operators.

For example, the following two expressions are equivalent

```
10 * 2
10 << 1 # shift 10 left by 1
```

and this is also equivalent

```
8 / 4
8 >> 2 # shift 8 right by 2
```

Bit shifting operations opcodes in the EVM, such as shr (shift right) and shl (shift left), cost 5 gas while multiplication and division operations (mul and div) cost 3 gas each.

Majority of gas savings also comes the fact that solidity does no overflow/underflow or division by check for shr and shl operations. It’s important to have this in mind when using these operators so that overflow and underflow bugs don't happen.

## 16. It is sometimes cheaper to cache calldata

Although the calldataload instruction is a cheap opcode, the solidity compiler will sometimes output cheaper code if you cache calldataload. This will not always be the case, so you should test both possibilities.

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

## 17. Use branchless algorithms as a replacement for conditionals and loops

The max code from an earlier section is an example of a branchless algorithm, i.e. it eliminates the JUMP opcode, which is more costly than arithmetic opcodes in general.

For loops have jumps built into them, so you may want to consider [loop unrolling](https://en.wikipedia.org/wiki/Loop_unrolling) to save gas.

Loops don’t have to be unrolled all the way. For example, you can execute a loop two items at a time and cut the number of jumps in half.

This is a very extreme optimization, but you should be aware that conditional jumps and loops introduce a slightly more expensive opcode.

## 18. Internal functions only used once can be inlined to save gas

It is okay to have internal functions, however they introduce additional jump labels to the bytecode.

Hence, in a case where it is only used by one function, it is better to inline the logic of the internal function inside the function it is being used. This will save some gas by avoiding jumps during the function execution.

## 19. Compare array equality and string equality by hashing them if they are longer than 32 bytes

This is a trick you will rarely use, but looping over the arrays or strings is a lot costlier than hashing them and comparing the hashes.

## 20. Use lookup tables when computing powers and logarithms

If you need to take logarithms or powers where the base or the power is a fraction, it may be preferable to precompute a table if either the base or the power is fixed.

Consider the [Bancor Formula](https://github.com/AragonBlack/fundraising/blob/master/apps/bancor-formula/contracts/BancorFormula.sol#L293) and [Uniswap V3 Tick Math](https://github.com/Uniswap/v3-core/blob/main/contracts/libraries/TickMath.sol#L23) as examples.


## Precompiled contracts may be useful for some multiplication or memory operations.

The [Ethereum precomipled contracts](https://www.rareskills.io/post/solidity-precompiles) provide operations primarily for cryptography, but if you need to multiply large numbers over a modulus or copy significant chunk of memory, consider using the precompiles. Be aware that this might make your application incompatible with some layer 2s.

