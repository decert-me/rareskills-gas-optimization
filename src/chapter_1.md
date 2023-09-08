[The RareSkills Book of Gas Optimization](##Solidity%20Gas%20Optimization)

- [Gas optimization tricks do not always work](##Gas%20optimization%20tricks%20do%20not%20always%20work)
- [Beware of complexity and readability](##Beware%20of%20complexity%20and%20readability)
- [Comprehensive treatment of each topic isn’t possible here](##Comprehensive%20treatment%20of%20each%20topic%20isn’t%20possible%20here)
- [We do not discuss application-specific tricks](##We%20do%20not%20discuss%20application-specific%20tricks)
- [1. Most important: avoid zero to one storage writes where possible](##1.%20Most%20important:%20avoid%20zero%20to%20one%20storage%20writes%20where%20possible)
- [2. Cache storage variables: write and read storage variables exactly once](##2.%20Cache%20storage%20variables:%20write%20and%20read%20storage%20variables%20exactly%20once)
- [3. Pack related variables](##3.%20Pack%20related%20variables)
- [4. Pack structs](##4.%20Pack%20structs)
- [5. Keep strings smaller than 32 bytes](##5.%20Keep%20strings%20smaller%20than%2032%20bytes)
- [6. Variables that are never updated should be immutable or constant](##6.%20Variables%20that%20are%20never%20updated%20should%20be%20immutable%20or%20constant)
- [7. Using mappings instead of arrays to avoid length checks](##7.%20Using%20mappings%20instead%20of%20arrays%20to%20avoid%20length%20checks)
- [8. Using unsafeAccess on arrays to avoid redundant length checks](##8.%20Using%20unsafeAccess%20on%20arrays%20to%20avoid%20redundant%20length%20checks)
- [9. Use bitmaps instead of bools when a significant amount of booleans are used](##9.%20Use%20bitmaps%20instead%20of%20bools%20when%20a%20significant%20amount%20of%20booleans%20are%20used)
- [10. Use SSTORE2 or SSTORE3 to store a lot of data](##10.%20Use%20SSTORE2%20or%20SSTORE3%20to%20store%20a%20lot%20of%20data)
- [11. Use storage pointers instead of memory where appropriate](##11.%20Use%20storage%20pointers%20instead%20of%20memory%20where%20appropriate)
- [12. Avoid having ERC20 token balances go to zero, always keep a small amount](##12.%20Avoid%20having%20ERC20%20token%20balances%20go%20to%20zero,%20always%20keep%20a%20small%20amount)
- [13. Count from n to zero instead of counting from zero to n](##13.%20Count%20from%20n%20to%20zero%20instead%20of%20counting%20from%20zero%20to%20n)


## Solidity Gas Optimization

Gas optimization in Ethereum is re-writing Solidity code to accomplish the same business logic while consuming fewer gas units in the Ethereum Virtual Machine (EVM).

Clocking in at over 11,000 words, not including source code, this article is the most complete treatment of gas optimization available.

To fully understand the tricks in this tutorial, you’ll need to understand how the EVM works, which you can learn by taking our [Gas Optimization Course](https://hackmd.io/eQJUW4PLQN-6HRrgyxdhsQ?view), [Yul Course](https://hackmd.io/eQJUW4PLQN-6HRrgyxdhsQ?view), and practicing [Huff Puzzles](https://github.com/RareSkills/huff-puzzles).

However, if you simply want to know what areas of the code to target for possible gas optimizations, this article gives you a lot of areas to look.

## Gas optimization tricks do not always work

Some gas optimization tricks only work in a certain context. For example, intuitively, it would seem that

```
if (!cond) {
    // branch False
}
else {
    // branch True
}
```

is less efficient than

```
if (cond) {
    // branch True
}
else {
    // branch False
}
```

because extra opcodes are spent inverting the condition. Counterintuitively, there are many cases where this optimization actually increases the cost of the transaction. The solidity compiler can be unpredictable sometimes.

Therefore, you should actually measure the effect of the alternatives before settling on a certain algorithm. Think of some of these tricks are bringing awareness to areas where the compiler may be surprising.

Some tricks that are not universal are marked as such in this document. Gas optimization tricks sometimes depend on what the compiler is doing locally. You should generally test both the optimal version of the code and the non-optimal version to see you actually get an improvement. We will document some surprising cases where what should lead to an optimization actually leads to higher cost.

Second, some of these optimization behavior may change when using the --via-ir option on the Solidity compiler.

## Beware of complexity and readability

Gas optimizations usually make code less readable and more complex. A good engineer must make a subjective tradeoff about what optimizations are worth it, and which ones aren’t.

## Comprehensive treatment of each topic isn’t possible here

We cannot explain each optimization in detail, and it isn’t really necessary to as there are other online resources. For example, giving a complete, or even substantial treatment of layer 2s and state channels would be outside of scope, and there are other resources online to learn those subjects in detail.

The purpose of this article is to be the most comprehensive list of tricks out there. If a trick feels unfamiliar, it can be a prompt for further self-study. If the header looks like a trick you already know, just skim over that section.

## We do not discuss application-specific tricks

There are gas-efficient was to determine if a number is prime for example, but this is so rarely needed that dedicating space to it would lower the value of this article. Similarly, in our [Tornado Cash tutorial](https://www.rareskills.io/post/how-does-tornado-cash-work), we suggest ways the codebase could be made more efficient, but including that treatment here would not benefit readers as it is too application specific.

## 1. Most important: avoid zero to one storage writes where possible

Initializing a storage variable is one of the most expensive operations a contract can do.

When a storage variable goes from zero to non-zero, the user must pay 22,100 gas total (20,000 gas for a zero to non-zero write and 2,100 for a cold storage access).

This is why the Openzeppelin reentrancy guard registers functions as active or not with 1 and 2 rather than 0 and 1. It only costs 5,000 gas to alter a storage variable from non-zero to non-zero.

## 2. Cache storage variables: write and read storage variables exactly once

You will see the following pattern frequently in efficient solidity code. Reading from a storage variable costs at least 100 gas as Solidity does not cache the storage read. Writes are considerably more expensive. Therefore, you should manually cache the variable to do exactly one storage read and exactly one storage write.

```
// SPDX-License-Identifier: MIT
pragma solidity 0.8.20;

contract Counter1 {
    uint256 public number;

    function increment() public {
        require(number < 10);
        number = number + 1;
    }
}

contract Counter2 {
    uint256 public number;

    function increment() public {
        uint256 _number = number;
        require(_number < 10);
        number = _number + 1;
    }
}
```

The first function reads counter twice, the second code reads it once.

## 3. Pack related variables

Packing related variables into same slot reduces gas costs by minimizing costly storage related operations.

**Manual packing is the most efficient**

 We store and retrieve two uint80 values in one variable (uint160) by using bit shifting. This will use only one storage slot and is cheaper when storing or reading the individual values in a single transaction.

```
// SPDX-License-Identifier: MIT
pragma solidity 0.8.20;

contract GasSavingExample {
    uint160 public packedVariables;

    function packVariables(uint80 x, uint80 y) external {
        packedVariables = uint160(x) << 80 | uint160(y);
    }

    function unpackVariables() external view returns (uint80, uint80) {
        uint80 x = uint80(packedVariables >> 80);
        uint80 y = uint80(packedVariables);
        return (x, y);
    }
}
```

**EVM Packing is slightly less efficient** 

This also uses one slot like the above example, but may be slightly expensive when storing or reading values in a single transaction. This is because the EVM will do the bit-shifting itself.

```
contract GasSavingExample2 {
    uint80 public var1;
    uint80 public var2;

    function updateVars(uint80 x, uint80 y) external {
        var1 = x;
        var2 = y;
    }

    function loadVars() external view returns (uint80, uint80) {
        return (var1, var2);
    }
}
```

**No packing is least efficient** 

This does not use any optimization, and is more expensive when storing or reading values.

Unlike the other examples, this uses two storage slots to store the variables.

```
contract NonGasSavingExample {
    uint256 public var1;
    uint256 public var2;

    function updateVars(uint256 x, uint256 y) external {
        var1 = x;
        var2 = y;
    }

    function loadVars() external view returns (uint256, uint256) {
        return (var1, var2);
    }
}    
```

## 4. Pack structs

Packing struct items, like packing related state variables, can help save gas. (It is important to note that in Solidity, struct members are stored sequentially in the contract’s storage, starting from the slot position where they are initialized).

Consider the following examples:

**Unpacked Struct** 

The unpackedStruct has three items which will be stored in three seperate slot. However, if this items were packed, only two slots would be used and this will make reading and writing to the struct’s items cheaper.

```
// SPDX-License-Identifier: MIT
pragma solidity 0.8.20;

contract Unpacked_Struct {
    struct unpackedStruct {
        uint64 time; // Takes one slot - although it  only uses 64 bits (8 bytes) out of 256 bits (32 bytes).
        uint256 money; // This will take a new slot because it is a complete 256 bits (32 bytes) value and thus cannot be packed with the previous value.
        address person; // An address occupies only 160 bits (40 bytes).
    }

    // Starts at slot 0
    unpackedStruct details = unpackedStruct(53_000, 21_000, address(0xdeadbeef));

    function unpack() external view returns (unpackedStruct memory) {
        return details;
    }
}
```

**Packed Struct**

 We can make the example above use less gas by packing the struct items like this.

```
contract Packed_Struct {
    struct packedStruct {
        uint64 time; // In this case, both `time` (64 bits) and `person` (160 bits) are packed in the same slot since they can both fit into 256 bits (32 bytes)
        address person; // Same slot as `time`. Together they occupy 224 bits (28 bytes) out of 256 bits (32 bytes).
        uint256 money; // This will take a new slot because it is a complete 256 bits (32 bytes) value and thus cannot be packed with the previous value.
    }
    
    // Starts at slot 0
    packedStruct details = packedStruct(53_000, address(0xdeadbeef), 21_000);

    function unpack() external view returns (packedStruct memory) {
        return details;
    }
}
```

## 5. Keep strings smaller than 32 bytes

In Solidity, strings are variable length dynamic data types, meaning their length can change and grow as needed.

If the length is 32 bytes or longer, the slot in which they are defined stores the length of the string * 2 + 1, while their actual data is stored elsewhere (the keccak hash of that slot).

However, if a string is less than 32 bytes, the length * 2 is stored at the least significant byte of it’s storage slot and the actual data of the string is stored starting from the most significant byte in the slot in which it is defined.

**String example (less than 32 bytes)**

```
// SPDX-License-Identifier: MIT
pragma solidity 0.8.20;

contract StringStorage1 {
    // Uses only one slot
    // slot 0: 0x(len * 2)00...(hex"hello")
    // Has smaller gas cost due to size.
    string public exampleString = "hello";

    function getString() public view returns (string memory) {
        return exampleString;
    }
}
```

**String example (greater than 32 bytes)**

```
contract StringStorage2 {
    // Length is more than 32 bytes. 
    // Slot 0: 0x00...(length*2+1).
    // keccak256(0x00): stores hex representation of "hello"
    // Has increased gas cost due to size.
    string public exampleString = "This is a string that is slightly over 32 bytes!";

    function getStringLongerThan32bytes() public view returns (string memory) {
        return exampleString;
    }
}
```

**We can put this to test with following foundry test script:**

```
// SPDX-License-Identifier: MIT
pragma solidity 0.8.20;

import "forge-std/Test.sol";
import "../src/StringLessThan32Bytes.sol";

contract StringStorageTest is Test {
    StringStorage1 public store1;
    StringStorage2 public store2;

    function setUp() public {
        store1 = new StringStorage1();
        store2 = new StringStorage2();
    }

    function testStringStorage1() public {
        // test for string less than 32 bytes
        store1.getString();
        bytes32 data = vm.load(address(store1), 0); // slot 0
        emit log_named_bytes32("Full string plus length", data); // the full string and its length*2 is stored at slot 0, because it is less than 32 bytes
    }

    function testStringStorage2() public {
        // test for string longer than 32 bytes
        store2.getStringLongerThan32bytes();
        bytes32 length = vm.load(address(store2), 0); // slot 0 stores the length*2+1
        emit log_named_bytes32("Length of string", length);

        // uncomment to get original length as number
        // emit log_named_uint("Real length of string (no. of bytes)", uint256(length) / 2); 
        // divide by 2 to get the original length
        
        bytes32 data1 = vm.load(address(store2), keccak256(abi.encode(0))); // slot keccak256(0)
        emit log_named_bytes32("First string chunk", data1);

        bytes32 data2 = vm.load(address(store2), bytes32(uint256(keccak256(abi.encode(0))) + 1));
        emit log_named_bytes32("Second string chunk", data2);
    }
}
```

This is the result after running the test.

If we concatenate the hex value of the string (longer than 32 bytes) without the length, we convert it back to the original string (with Python).

If the length of a string is less than 32 bytes, it’s also efficient to store it in a bytes32 variable and use assembly to use it when needed.

Example:

```
contract EfficientString {
    bytes32 shortString;

    function getShortString() external view returns(string memory) {
        string memory value;

        assembly {
            // get slot 0
            let slot0Value := sload(shortString.slot)
            
            // get the byte that holds the length info and divide it by 2 to get the length
            let len := div(shr(248, slot0Value), 2)

            // get string, shift by 256 - (len * 8) to get it to the most significant byte
            let str := shl(sub(256, mul(len, 8)), slot0Value)
            
            // store length in memory
            mstore(0x80, len)
            
            // store string in memory
            mstore(0xa0, str)

            // make `value` reference 0x80 so that solidity does the returning for us
            value := 0x80
            
            // update the free memory pointer
            mstore(0x40, 0xc0)
        }

        return value;
    }

    function storeShortString(string calldata value) external {
        assembly {
            // require that the length is less than 32
            if gt(value.length, 31) {
                revert(0, 0)
            }

            // get the length, multiply it by 2 (following solidity pattern) and push the result to the most significant byte
            let shiftedLen := shl(248, mul(value.length, 2))

            // get the string itself
            let str := shr(sub(256, mul(value.length, 8)), calldataload(value.offset))

            // or the shiftedLen and str to get what we need to store in storage
            let toBeStored := or(shiftedLen, str)

            // store it in storage
            sstore(shortString.slot, toBeStored)
        }
    }
}
```

The code above can be further optimized but kept this way to make it easier to understand.

## 6. Variables that are never updated should be immutable or constant

In Solidity, variables which are not intended to be updated should be constant or immutable.

This is because constants and immutable values are embedded directly into the bytecode of the contract which they are defined and does not use storage because of this.

```
// SPDX-License-Identifier: MIT
pragma solidity 0.8.20;

contract Constants {
    uint256 constant MAX_UINT256 = 0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff;

    function get_max_value() external pure returns (uint256) {
        return MAX_UINT256;
    }
}

// This uses more gas than the above contract
contract NoConstants {
    uint256 MAX_UINT256 = 0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff;

    function get_max_value() external view returns (uint256) {
        return MAX_UINT256;
    }
}
```

This saves a lot of gas as we do not make any storage reads which are costly.

## 7. Using mappings instead of arrays to avoid length checks

When storing a list or group of items that you wish to organize in a specific order and fetch with a fixed key/index, it’s common practice to use an array data structure. This works well, but did you know that a trick can be implemented to save 2,000+ gas on each read using a mapping?

See the example below

```
/// get(0) gas cost: 4860 
contract Array {
    uint256[] a;

    constructor() {
        a.push() = 1;
        a.push() = 2;
        a.push() = 3;
    }

    function get(uint256 index) external view returns(uint256) {
        return a[index];
    }
}

/// get(0) gas cost: 2758
contract Mapping {
    mapping(uint256 => uint256) a;

    constructor() {
        a[0] = 1;
        a[1] = 2;
        a[2] = 3;
    }

    function get(uint256 index) external view returns(uint256) {
        return a[index];
    }
}
```

Just by using a mapping, we get a gas saving of 2102 gas. Why? Under the hood when you read the value of an index of an array, solidity adds bytecode that checks that you are reading from a valid index (i.e an index strictly less than the length of the array) else it reverts with a panic error (Panic(0x32) to be precise). This prevents from reading unallocated or worse, allocated storage/memory locations.

Due to the way mappings are (simply a key => value pair), no check like that exists and we are able to read from the a storage slot directly. It’s important to note that when using mappings in this manner, your code should ensure that you are not reading an out of bound index of your canonical array.

## 8. Using unsafeAccess on arrays to avoid redundant length checks

An alternative to using mappings to avoid the length checks that solidity does when reading from arrays (while still using arrays), is using the unsafeAccess function in Openzeppelin’s [Arrays.sol](https://hackmd.io/eQJUW4PLQN-6HRrgyxdhsQ?view) library. This allows developers to directly access values of any given index of an array while skipping the length overflow check. It’s still important to only use this if you are sure that indexes parsed into the function cannot exceed the length of the array parsed in.

## 9. Use bitmaps instead of bools when a significant amount of booleans are used

A common pattern, especially in airdrops, is to mark an address as “already used” when claiming the airdrop or NFT mint.

However, since it only takes one bit to store this information, and each slot is 256 bits, that means one can store a 256 flags/booleans with one storage slot.

You can learn more about this technique from these resources:

[Video tutorial by a student at RareSkills](https://www.youtube.com/watch?v=Iv0cPT-7AR8)

 [Bitmap presale tutorial](https://medium.com/donkeverse/hardcore-gas-savings-in-nft-minting-part-3-save-30-000-in-presale-gas-c945406e89f0)

## 10. Use SSTORE2 or SSTORE3 to store a lot of data

### SSTORE

SSTORE is an EVM opcode that allows us to store persistent data on a key value basis . As everything in EVM, a key and value are both 32 bytes values.

Costs of writing(SSTORE) and reading(SLOAD) are very expensive in terms of gas spent. Writing 32 bytes costs 22,100 gas, which translates to about 690 gas per bytes. On the other hand, writing a smart contract’s bytecode costs 200 gas per bytes.

### SSTORE2

SSTORE2 is a unique concept in a way that it uses a contract’s bytecode to write and store data. To achieve this we use bytecode’s inherent property of immutability.

Some properties of SSTORE2:

- We can write only once. Effectively using CREATE instead of SSTORE.
- To read, instead of using SLOAD, we now call EXTCODECOPY on the deployed address where the particular data is stored as bytecode.
- Writing data becomes significantly cheaper when more and more data needs to be stored.

Example:

**Writing data**

Our goal is to store a specific data (in bytes format) as the contract’s bytecode. To achieve this,We need to do 2 things:-

1. Copy our data to memory first, as EVM then takes this data from memory and store it as runtime code. For more, check this.
2. Return and store the newly deployed contract address for future use.

- We add the contract code size in place of the four zeroes(0000) between 61 and 80 in the below code 0x61000080600a3d393df300. Hence if code size is 65, it will become 0x61004180600a3d393df300(0x0041 = 65)
- This bytecode is responsible for step 1 we mentioned.
- Now we return the newly deployed address for step 2.

Final contract bytecode = 00 + data (00 = STOP is prepended to ensure the bytecode cannot be executed by calling the address mistakenly)

**Reading data**

- To get the relevant data , you need the address where you stored the data.
- We revert if code size is = 0 for obvious reasons.
- Now we simply return the contract’s bytecode from the relevant starting position which is after 1 bytes(remember first byte is STOP OPCODE(0x00)).

Additional Information for the curious:

- We can also use pre-deterministic address using CREATE2 to calculate the pointer address off chain or on chain without relying on storing the pointer.

Ref: [solady](https://github.com/Vectorized/solady/blob/main/src/utils/SSTORE2.sol)

####  

#### SSTORE3

To understand SSTORE3, first let’s recap an important property of SSTORE2.

- The newly deployed address is dependent on the data we intend to store.

##### Write data

SSTORE3 implements a design such that the newly deployed address is independent of our provided data. The provided data is first stored in storage using SSTORE. Then we pass a constant INIT_CODE as data in CREATE2 which internally reads the provided data stored in storage to deploy it as code.

This design choice enables us to efficiently calculate the pointer address of our data just by providing the salt(which can be less than 20 bytes). Thus enabling us to pack our pointer with other variables, thereby reducing storage costs.

##### Read data

Try to imagine how we could be reading the data.

- Answer is we can easily compute the deployed address just by providing salt.
- Then after we receive the pointer address, use the same EXTCODECOPY opcode to get the required data.

To **summarize**:

- *SSTORE2* is helpful in cases where write operations are rare, and large read operations are frequent (and pointer > 14 bytes)
- *SSTORE3* is better when you write very rarely, but read very often. (and pointer < 14 bytes)

Credit to [Philogy](https://twitter.com/real_philogy/status/1677811731962245120) for SSTORE3.

## 11. Use storage pointers instead of memory where appropriate

In Solidity, storage pointers are variables that reference a location in storage of a contract. They are not exactly the same as pointers in languages like C/C++.

It is helpful to know how to use storage pointers efficiently to avoid unnecessary storage reads and perform gas-efficient storage updates.

Here’s an example showing where storage pointers can be helpful.

```
// SPDX-License-Identifier: MIT
pragma solidity 0.8.20;

contract StoragePointerUnOptimized {
    struct User {
        uint256 id;
        string name;
        uint256 lastSeen;
    }

    constructor() {
        users[0] = User(0, "John Doe", block.timestamp);
    }

    mapping(uint256 => User) public users;

    function returnLastSeenSecondsAgo(uint256 _id) public view returns (uint256) {
        User memory _user = users[_id];
        uint256 lastSeen = block.timestamp - _user.lastSeen;
        return lastSeen;
    }
}
```

Above, we have a function that returns the last seen of a user at a given index. It gets the lastSeen value and subtracts that from the current block.timestamp. Then we copy the whole struct into memory and get the lastSeen which we use in calculating the last seen in seconds ago. This method works well but is not so efficient, this is because we are copying all of the struct from storage into memory including variables we don’t need. Only if there was a way to only read from the lastSeen storage slot (without assembly). That’s where storage pointers come in.

```
// This results in approximately 5,000 gas savings compared to the previous version.
contract StoragePointerOptimized {
    struct User {
        uint256 id;
        string name;
        uint256 lastSeen;
    }

    constructor() {
        users[0] = User(0, "John Doe", block.timestamp);
    }

    mapping(uint256 => User) public users;

    function returnLastSeenSecondsAgoOptimized(uint256 _id) public view returns (uint256) {
        User storage _user = users[_id]; 
        uint256 lastSeen = block.timestamp - _user.lastSeen;
        return lastSeen;
    }
}
```

“The above implementation results in approximately 5,000 gas savings compared to the first version”. Why so, the only change here was changing memory to storage and we were told that anything storage is expensive and should be avoided?

Here we store the storage pointer for users[_id] in a fixed sized variable on the stack (the pointer of a struct is basically the storage slot of the start of the struct, in this case, this will be the storage slot of user[_id].id). Since storage pointers are lazy (meaning they only act(read or write) when called or referenced). Next we only access the lastSeen key of the struct. This way we make a single storage load then store it on the stack, instead of 3 or possibly more storage loads and a memory store before taking a small chunk from memory unto the stack.

Note: When using storage pointers, it’s important to be careful not to reference [dangling pointers](https://docs.soliditylang.org/en/v0.8.21/types.html#dangling-references-to-storage-array-elements). (Here is a [video tutorial on dangling pointers](https://www.youtube.com/watch?v=Zi4BANKFNP8) by one of RareSkills' instructors).

## 12. Avoid having ERC20 token balances go to zero, always keep a small amount

This is related to the avoiding zero writes section above, but it’s worth calling out separately because the implementation is a bit subtle.

If an address is frequently emptying (and reloading) it’s account balance, this will lead to a lot of zero to one writes.

## 13. Count from n to zero instead of counting from zero to n

When setting a storage variable to zero, a refund is given, so the net gas spent on counting will be less if the final state of the storage variable is zero.











