# Saving Gas On Deployment

- [1. Use the account nonce to predict the addresses of interdependent smart contracts thereby avoiding storage variables and address setter functions](#1-use-the-account-nonce-to-predict-the-addresses-of-interdependent-smart-contracts-thereby-avoiding-storage-variables-and-address-setter-functions)
- [2. Make constructors payable](#2-make-constructors-payable)
- [3. Deployment size can be reduced by optimizing the IPFS hash to have more zeros (or using the --no-cbor-metadata compiler option)](#3-deployment-size-can-be-reduced-by-optimizing-the-ipfs-hash-to-have-more-zeros-or-using-the---no-cbor-metadata-compiler-option)
- [4. Use selfdestruct in the constructor if the contract is one-time use](#4-use-selfdestruct-in-the-constructor-if-the-contract-is-one-time-use)
- [5. Understand the trade-offs when choosing between internal functions and modifiers](#5-understand-the-trade-offs-when-choosing-between-internal-functions-and-modifiers)
- [6. Use clones or metaproxies when deploying very similar smart contracts that are not called frequently](#6-use-clones-or-metaproxies-when-deploying-very-similar-smart-contracts-that-are-not-called-frequently)
- [7. Admin functions can be payable](#7-admin-functions-can-be-payable)
- [8. Custom errors are (usually) smaller than require statements](#8-custom-errors-are-usually-smaller-than-require-statements)
- [9. Use existing create2 factories instead of deploying your own](#9-use-existing-create2-factories-instead-of-deploying-your-own)


## 1. Use the account nonce to predict the addresses of interdependent smart contracts thereby avoiding storage variables and address setter functions

When using traditional contract deployment ([create](https://hackmd.io/eQJUW4PLQN-6HRrgyxdhsQ?view)), the address of a smart contract can be detrministically computed based on the deployer’s address and their nonce.

The [LibRLP library from Solady](https://github.com/Vectorized/solady/blob/6c54795ef69838e233020e9ab29f3f6288efdf06/src/utils/LibRLP.sol#L27) can help us do just that.

Take the following example scenario;

StorageContract only allows Writer to set the storage variable x, which means it needs to know the address of Writer. But for Writer to write to StorageContract, it also needs to know the address of StorageContract.

The below implementation is a naive approach to this problem. It handles it by having a setter function which sets a storage variable after deployment. But storage variables are expensive and we’d rather avoid them.

```
contract StorageContract {
    address immutable public writer;
    uint256 public x;
    
    constructor(address _writer) {
        writer = _writer;
    }

    function setX(uint256 x_) external {
        require(msg.sender == address(writer), "only writer can set");
        x = x_;
    }
}

contract Writer {
    StorageContract public storageContract;

    // cost: 49291
    function set(uint256 x_) external {
        storageContract.setX(x_);
    }

    function setStorageContract(address _storageContract) external {
        storageContract = StorageContract(_storageContract);
    }
}
```

This costs more both at deployment and at runtime. It involves deploying the Writer, then deploying the StorageContract with the deployed Writer address set as the writer. Then setting Writer’s StorageContract variable with the newly created StorageContract. This involves a lot of steps and can be expensive since we store StorageContract in storage. Calling Writer.setX() costs 49k gas.

A more efficient way to do this would be to calculate the address the StorageContract and Writer will be deployed to beforehand and set them in both their constructors.

Here’s an example of what this would look;

```
import {LibRLP} from "https://github.com/vectorized/solady/blob/main/src/utils/LibRLP.sol";

contract StorageContract {
    address immutable public writer;
    uint256 public x;
    
    constructor(address _writer) {
        writer = _writer;
    }

    // cost: 47158
    function setX(uint256 x_) external {
        require(msg.sender == address(writer), "only writer can set");
        x = x_;
    }
}

contract Writer {
    StorageContract immutable public storageContract;
    
    constructor(StorageContract _storageContract) {
        storageContract = _storageContract;
    }

    function set(uint256 x_) external {
        storageContract.setX(x_);
    }
}

// one time deployer.
contract BurnerDeployer {
    using LibRLP for address;

    function deploy() public returns(StorageContract storageContract, address writer) {
        StorageContract storageContractComputed = StorageContract(address(this).computeAddress(2)); // contracts nonce start at 1 and only increment when it creates a contract
        writer = address(new Writer(storageContractComputed)); // first creation happens here using nonce = 1
        storageContract = new StorageContract(writer); // second create happens here using nonce = 2
        require(storageContract == storageContractComputed, "false compute of create1 address"); // sanity check
    }
}
```

Here, calling Writer.setX() costs 47k gas. We saved 2k+ gas by precomputing the address that StorageContract would be deployed to before deploying it so we could use it when deploying Writer, hence no need for a setter function.

It is not required to use a separate contract to employ this technique, you can do it inside the deployment script instead.

We provide a [video tutorial of address prediction](https://www.youtube.com/watch?v=eb3qtUc4UE4) done by [Philogy](https://twitter.com/real_philogy) if you wish to explore this further.

## 2. Make constructors payable

```
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.20;

contract A {}

contract B {
    constructor() payable {}
}
```

Making the constructor payable saved 200 gas on deployment. This is because non-payable functions have an implicit require(msg.value == 0) inserted in them. Additionally, fewer bytecode at deploy time mean less gas cost due to smaller calldata.

There are good reasons to make a regular functions non-payable, but generally a contract is deployed by a privileged address who you can reasonably assume won’t send ether. This might not apply if inexperienced users are deploying the contract.

## 3. Deployment size can be reduced by optimizing the IPFS hash to have more zeros (or using the --no-cbor-metadata compiler option)

We’ve already explained this in our tutorial about [smart contract metadata](https://www.rareskills.io/post/solidity-metadata), but to recap, the Solidity compiler appends 51 bytes of metadata to the actual smart contract code. Since each deployment byte costs 200 gas, removing them can take over 10,000 gas cost off of deployment.

This is not always ideal though as it can affect smart contract verification. Instead, developers can mine for code comments that make the IPFS hash that gets appended have more zeros in it.

## 4. Use selfdestruct in the constructor if the contract is one-time use

Sometimes, contracts are used to deploy several contracts in one transaction, which necessitates doing it in the constructor.

If the contract’s only use is the code in the constructor, then selfdestructing at the end of the operation will save gas.

Although selfdestruct is set for removal in an upcoming hardfork, it will still be supported in the constructor per [EIP 6780](https://eips.ethereum.org/EIPS/eip-6780)

## 5. Understand the trade-offs when choosing between internal functions and modifiers

Modifiers inject its implementation bytecode where it is used while internal functions jump to the location in the runtime code where the its implementation is. This brings certain trade-offs to both options.

- Using modifiers more than once means repetitiveness and increase in size of the runtime code but reduces gas cost because of the absence of jumping to the internal function execution offset and jumping back to continue. This means that if runtime gas cost matter most to you, then modifiers should be your choice but if deployment gas cost and/or reducing the size of the creation code is most important to you then using internal functions will be best.
- However, modifiers have the tradeoff that they can only be executed at the start or end of a functon. This means executing it at the middle of a function wouldn’t be directly possible, at least not without internal functions which kill the original purpose. This affects it’s flexibility. Internal functions however can be called at any point in a function.

Example showing difference in gas cost using modifiers and an internal function

```
// SPDX-License-Identifier: MIT
pragma solidity 0.8.19;

/** deployment gas cost: 195435
    gas per call:
              restrictedAction1: 28367
              restrictedAction2: 28377
              restrictedAction3: 28411
 */
 contract Modifier {
    address owner;
    uint256 val;

    constructor() {
        owner = msg.sender;
    }

    modifier onlyOwner() {
        require(msg.sender == owner);
        _;
    }

    function restrictedAction1() external onlyOwner {
        val = 1;
    }

    function restrictedAction2() external onlyOwner {
        val = 2;
    }

    function restrictedAction3() external onlyOwner {
        val = 3;
    }
}



/** deployment gas cost: 159309
    gas per call:
              restrictedAction1: 28391
              restrictedAction2: 28401
              restrictedAction3: 28435
 */
 contract InternalFunction {
    address owner;
    uint256 val;

    constructor() {
        owner = msg.sender;
    }

    function onlyOwner() internal view {
        require(msg.sender == owner);
    }

    function restrictedAction1() external {
        onlyOwner();
        val = 1;
    }

    function restrictedAction2() external {
        onlyOwner();
        val = 2;
    }

    function restrictedAction3() external {
        onlyOwner();
        val = 3;
    }
}
```

| Operation          | Deployment | restrictedAction1 | restrictedAction2 | **restrictedAction3** |
| ------------------ | ---------- | ----------------- | ----------------- | --------------------- |
| Modifiers          | 195435     | 28367             | 28377             | 28411                 |
| Internal Functions | 159309     | 28391             | 28401             | 28435                 |

From the table above, we can see that the contract that uses modifiers cost more than 35k gas more than the contract using internal functions when deploying it due to repetition of the onlyOwner functionality in 3 functions.

During runtime, we can see that each function using modifiers cost a fixed 24 gas less than the functions using internal functions.

## 6. Use clones or metaproxies when deploying very similar smart contracts that are not called frequently

When deploying multiple similar smart contracts, the gas costs can be high. To reduce these costs, you can use minimal clones or metaproxies which store the address of the implementation contract in their bytecode and interact with it as a proxy.

However, there is a trade-off between the runtime cost and deployment cost of clones. Clones are more expensive to interact with than normal contracts due to the delegatecall they use, so they should only be used when you don’t need to interact with them frequently. For example, the Gnosis Safe contract uses clones to reduce deployment costs.

Learn more about how to use clones and metaproxies to reduce the gas costs of deploying smart contracts from our blog posts:

- EIP-1167: Minimal Proxy Standard
- EIP-3448 Metaproxy Clone

## 7. Admin functions can be payable

We can make admin specific functions payable to save gas, because the compiler won’t be checking the callvalue of the function.

This will also make the contract smaller and cheaper to deploy as there will be fewer opcodes in the creation and runtime code.

## 8. Custom errors are (usually) smaller than require statements

Custom errors are cheaper than require statements with strings because of how custom errors are handled. Solidity stores only the first 4 bytes of the hash of the error signature and returns only that. This means during reverting, only 4 bytes needs to be stored in memory. In the case of string messages in require statements, Solidity has to store(in memory) and revert with at least 64 bytes.

Here’s an example below.

```
// SPDX-License-Identifier: MIT
pragma solidity 0.8.20;

contract CustomError {
    error InvalidAmount();

    function withdraw(uint256 _amount) external pure {
        if (_amount > 10 ether) revert InvalidAmount();
    }
}

// This uses more gas than the above contract
contract NoCustomError {
    function withdraw(uint256 _amount) external pure {
        require(_amount <= 10 ether, "Error: Pass in a valid amount");
    }
}
```

## 9. Use existing create2 factories instead of deploying your own

The title is self-explanatory. If you need a deterministic address, you can usually re-use a pre-deployed one.

