# Calldata Optimizations

- [1. Use vanity addresses (safely!)](#1-use-vanity-addresses-safely)
- [2. Avoid signed integers in calldata if possible](#2-avoid-signed-integers-in-calldata-if-possible)
- [3. Calldata is (usually) cheaper than memory](#3-calldata-is-usually-cheaper-than-memory)
- [4. Consider packing calldata, especially on an L2](#4-consider-packing-calldata-especially-on-an-l2)


Ethereum charges 4 gas for a zero byte of calldata and 16 gas for a non-zero byte. This is true during a normal function call and during deployment. Because of this, Solidity optimizers try to use zeros where possible.

## 1. Use vanity addresses (safely!)

It is cheaper to use vanity addresses with leading zeros, this saves calldata gas cost.

A good example is OpenSea [Seaport contract](https://etherscan.io/address/0x00000000000000adc04c56bf30ac9d3c0aaf14dc#code) with this address: 0x00000000000000ADc04C56Bf30aC9d3c0aAF14dC.

This will not save gas when calling the address directly. However, if that contract’s address is used as an argument to a function, that function call will cost less gas due to having more zeros in the calldata.

This is also true of passing EOAs with a lot of zeros as a function argument – it saves gas for the same reason.

Just be aware that there have been [hacks](https://www.halborn.com/blog/post/explained-the-profanity-address-generator-hack-september-2022) from generating vanity addresses for wallets with insufficiently random private keys. This is not a concert for smart contracts vanity addresses created with finding a salt for create2, because smart contracts do not have private keys.

## 2. Avoid signed integers in calldata if possible

Because [solidity uses two’s complement](https://www.rareskills.io/post/signed-int-solidity) to represent signed integers, calldata for small negative numbers will be largely non-zero. For example, -1 is 0xff..ff in two’s complement form and therefore more expensive.

## 3. Calldata is (usually) cheaper than memory

Loading function inputs or data directly from calldata is cheaper compared to loading from memory. This is because accessing data from calldata involves fewer operations and gas costs. Hence, it is advised to use memory only when the data needs to be modified in the function (calldata cannot be modified).

```
// SPDX-License-Identifier: MIT
pragma solidity 0.8.20;
contract CalldataContract {
    function getDataFromCalldata(bytes calldata data) public pure returns (bytes memory) {
        return data;
    }
}

contract MemoryContract {
    function getDataFromMemory(bytes memory data) public pure returns (bytes memory) {
        return data;
    }
}
```

## 4. Consider packing calldata, especially on an L2

Solidity automatically packs storage variables, but the abi encoding for variables that would be packed in storage are not packed in calldata.

This is a rather extreme optimization that leads to higher code complexity, but it is something to consider if a function takes a lot of calldata.

ABI encoding is not efficient for every data representation, some data representations can be encoded more efficiently in an application specific way.

##  

