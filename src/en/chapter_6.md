# Assembly tricks

- [1. Using assembly to revert with an error message](#1-using-assembly-to-revert-with-an-error-message)
- [2. Calling functions via interface incurs memory expansion costs, so use assembly to re-use data already in memory](#2-calling-functions-via-interface-incurs-memory-expansion-costs-so-use-assembly-to-re-use-data-already-in-memory)
- [3. Common math operations, like min and max have gas efficient alternatives](#3-common-math-operations-like-min-and-max-have-gas-efficient-alternatives)
- [4. Use SUB or XOR instead of ISZERO(EQ()) to check for inequality (more efficient in certain scenarios)](#4-use-sub-or-xor-instead-of-iszeroeq-to-check-for-inequality-more-efficient-in-certain-scenarios)
- [5. Use inline assembly to check for address(0)](#5-use-inline-assembly-to-check-for-address0)
- [6. selfbalance is cheaper than address(this).balance (more efficient in certain scenarios)](#6-selfbalance-is-cheaper-than-addressthisbalance-more-efficient-in-certain-scenarios)
- [7. Use assembly to perform operations on data of size 96 bytes or less: hashing and unindexed data in events](#7-use-assembly-to-perform-operations-on-data-of-size-96-bytes-or-less-hashing-and-unindexed-data-in-events)
- [8. Use assembly to reuse memory space when making more than one external call.](#8-use-assembly-to-reuse-memory-space-when-making-more-than-one-external-call)
- [9. Use assembly to reuse memory space when creating more than one contract.](#9-use-assembly-to-reuse-memory-space-when-creating-more-than-one-contract)
- [10. Test if a number is even or odd by checking the last bit instead of using a modulo operator](#10-Test-if-a-number-is-even-or-odd-by-checking-the-last-bit-instead-of-using-a-modulo-operator)



You should not assume that writing assembly will automatically lead to more efficient code. We’ve listed areas where writing assembly usually works better, but you should always test the non-assembly version.

## 1. Using assembly to revert with an error message

When reverting in solidity code, it is common practice to use a require or revert statement to revert execution with an error message. This can in most cases be further optimized by using assembly to revert with the error message.

Here’s an example;

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

From the example above we can see that we get a gas saving of over 300 gas when reverting wth the same error message with assembly against doing so in solidity. This gas savings come from the memory expansion costs and extra type checks the solidity compiler does under the hood.

## 2. Calling functions via interface incurs memory expansion costs, so use assembly to re-use data already in memory

When calling a function on a contract B from another contract A, it’s most convenient to use the interface, create an instance of B with an address and call the function we wish to call. This works very well, but due to how solidity compiles our code, it stores the data to send to contract B in a new memory location thereby expanding memory, sometimes unnecessarily.

With inline assembly, we can optimize our code better and save some gas by using previously used memory locations that we don’t need again or (if the calldata contract B  expects is less than 64 bytes) in the scratch space to store our calldata.

Here’s an example comparing the two;

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

We can see that calling set(uint256) on Assembly costs 220 gas less than it would cost if we used solidity.

Do note that when using inline assembly to make external calls, it’s important to check if the address we are calling has code deployed to it using extcodesize(addr) and revert if this returns 0. This is important because calling an address that has no code deployed to it always returns true which can be devastating for our contract logic in most scenarios.

## 3. Common math operations, like min and max have gas efficient alternatives

**Unoptimized**

```
function max(uint256 x, uint256 y) public pure returns (uint256 z) {
	z = x > y ? x : y;
}
```

**Optimized**

```
function max(uint256 x, uint256 y) public pure returns (uint256 z) {
    /// @solidity memory-safe-assembly
    assembly {
        z := xor(x, mul(xor(x, y), gt(y, x)))
    }
}
```

The code above is taken from the math section of the [Solady Library](https://github.com/Vectorized/solady/blob/main/src/utils/FixedPointMathLib.sol), more math operations can be found there. It is worth exploring the library to see what gas efficient operations are available to you.

The reason the above example is more gas efficient is because the ternary operator (and in general, code with conditionals in it) contain conditional jumps in the opcodes, which are more costly.

## 4. Use SUB or XOR instead of ISZERO(EQ()) to check for inequality (more efficient in certain scenarios)

When using inline assembly to compare the equality of two values (e.g if owner is the same as caller()), It is sometimes more efficient to do

```
if sub(caller, sload(owner.slot)) {
    revert(0x00, 0x00)
}
```

over doing this

```
if eq(caller, sload(owner.slot)) {
    revert(0x00, 0x00)
}
```

xor can accomplish the same thing, but be aware that xor will consider a value with all the bits flipped to be equal also, so make sure that this cannot become an attack vector.

This trick will depend on the compiler version used and the context of the code.

## 5. Use inline assembly to check for address(0)

Writing contracts in inline assembly is generally considered gas optimized. we can manipulate memory directly and use fewer opcodes instead of leaving it to the Solidity compiler.

Authentication mechanism is one example where using inline assembly is good, like implementing address zero check.

Here’s an example below:

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

## 6. selfbalance is cheaper than address(this).balance (more efficient in certain scenarios)

The solidity code address(this).balance can sometimes be done more efficiently with the selfbalance() function from yul, but be aware the compiler is sometimes smart enough to use this trick under the hood, so test both ways.

## 7. Use assembly to perform operations on data of size 96 bytes or less: hashing and unindexed data in events

Solidity always writes to memory by expanding it which sometimes is not efficient. We can optimize memory operations on data that are 96 bytes in size or less by utilizing inline-assembly.

Solidity reserves it’s first 64 bytes of memory (mem[0x00:0x40]) as scratch space that devs can use to perform any operation with guarantees that it won’t be overwritten or read from unexpectedly. The next 32 bytes of memory (mem[0x40:0x60]) is where solidity stores, reads and updates the free memory pointer from. This is how solidity keeps track of the next memory offset to write new data to. The next 32 bytes of memory (mem[0x60:0x80]) is called the zero slot. It is where uninitialized dynamic memory data (bytes memory, string memory, T[] memory(where T is any valid type)) points to. Since these values are uninitialized, solidity advances that the slot they point to (0x60) remain 0x00.

Note: Structs stored in memory even when dynamic (i.e have a dynamic value within themselves), when uninitialized do not point to the zero slot.

Note: Uninitialized dynamic memory data still point to the zero slot even if they’re nested within a struct.

If we can utilize the scratch space in performing operations in memory that the compiler would usually expand memory to perform if it did so itself, then we can optimize our code. So we have 64 bytes of cheaper memory to work with now.

The free memory pointer space can also be used as long as we update it before exiting the assembly block too. We can store it on the stack temporarily for this.

Let’s see some examples.

- Using assembly to log up to 96 bytes of unindexed data

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

The example above shows how we can save almost 2,000 gas by using memory to store the data we wish to emit in the BlockData event.

There is no need to update our free memory pointer here because execution ends right after we emit our event and we never step back into solidity code.

Let’s take another example where we would need to update the free memory pointer

- Using assembly to hash up to 96 bytes of data

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

In the above example, similar to the first one, we use assembly to store values in the first 96 bytes of memory which saves us 1,000+ gas. Also notice that in this instance, because we still break back into solidity code, we cached and updated our free memory pointer at the start and end of our assembly block. This is to make sure that the solidity compiler’s assumptions on what is stored in memory remains compatible.

## 8. Use assembly to reuse memory space when making more than one external call.

An operation that causes the solidity compiler to expand memory is making external calls. When making external calls the compiler has to encode the function signature of the function it wishes to call on the external contract alongside it’s arguments in memory. As we know, solidity does not clear or reuse memory memory so it’ll have to store these data in the next free memory pointer which expands memory further.

With inline assembly, we can either use the scratch space and free memory pointer offset to store this data (as above) if the function arguments do not take up more than 96 bytes in memory. Better still, if we are making more than one external call we can reuse the same memory space as the first calls to store the new arguments in memory without expanding memory unnecessarily. Solidity in this scenario would expand memory by as much as the returned data length is. This is because the returned data is stored in memory (in most cases). If the return data is less than 96 bytes, we can use the scratch space to store it to prevent expanding memory.

See the example below;

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

We save approximately 2,000 gas by using the scratch space to store the function selector and it’s arguments and also reusing the same memory space for the second call while storing the returned data in the zero slot thus not expanding memory.

If the arguments of the external function you wish to call is above 64 bytes and if you are making one external call, it wouldn’t save any significant gas writing it in assembly. However, if making more than one call. You can still save gas by reusing the same memory slot for the 2 calls using inline assembly.

Note: Always remember to update the free memory pointer if the offset it points to is already used, to avoid solidity overriding the data stored there or using the value stored there in an unexpected way.

Also note to avoid overwriting the zero slot (0x60 memory offset) if you have undefined dynamic memory values within that call stack. An alternative is to explicitly define dynamic memory values or if used, to set the slot back to 0x00 before exiting the assembly block.

##  

## 9. Use assembly to reuse memory space when creating more than one contract.

Solidity treats contract creation similar to external calls that returns 32 bytes (i.e it returns the address of the created contract or address(0) if the contract creation failed).

From the section on saving gas with external calls, we can immediately see that one way we can optimize this is to store the returned address in the scratch space and avoid expanding memory.

See a similar example below;

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

We saved close to 1,000 gas by using inline assembly.

Note: In the scenario where the two contracts to be deployed are not the same, the second contract’s creation code would need to be mstored manually using inline assembly and not assigned to a variable in solidity to avoid memory expansion.

## 10. Test if a number is even or odd by checking the last bit instead of using a modulo operator

The conventional way to check if a number is even or odd is to do x % 2 == 0 where x is the number in question. You can instead check if x & uint256(1) == 0. where x is assumed to be a uint256. Bitwise and is cheaper than the modulo op code. In binary, the rightmost bit represents "1" whereas all the bits to the are multiples of 2, which are even. Adding "1" to an even number causes it to be odd.

