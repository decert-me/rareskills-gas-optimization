# Calldata 优化

- [1. 安全使用虚荣地址](#1-安全地使用虚荣地址)
- [2. 尽可能避免在 calldata 中使用有符号整数](#2-尽可能避免在-calldata-中使用有符号整数)
- [3. Calldata 通常比内存更便宜](#3-calldata-通常比内存更便宜)
- [4. 考虑对 calldata 进行打包，尤其是在 L2 上](#4-考虑对-calldata-进行打包尤其是在-l2-上)


以太坊对于 calldata 的零字节收取 4 gas，对于非零字节收取 16 gas。这在正常的函数调用和部署过程中都是成立的。因此，Solidity 优化器会尽可能使用零值。

## 1. （安全地）使用虚荣地址

使用具有前导零的虚荣地址更加便宜，这样可以节省 calldata 的 gas 成本。

一个很好的例子是 OpenSea 的 [Seaport 合约](https://etherscan.io/address/0x00000000000000adc04c56bf30ac9d3c0aaf14dc#code)，其地址为：0x00000000000000ADc04C56Bf30aC9d3c0aAF14dC。

直接调用该地址时不会节省 gas。然而，如果该合约地址作为函数的参数传递，由于 calldata 中有更多的零值，该函数调用将花费较少的 gas。

对于将大量零值的外部账户地址作为函数参数传递也是如此，由于相同的原因可以节省 gas。

只需注意，有人通过生成虚荣地址来攻击[钱包](https://www.halborn.com/blog/post/explained-the-profanity-address-generator-hack-september-2022)，这是由于生成时使用了不够随机的私钥。对于使用 create2 找到盐值来创建的智能合约虚荣地址，这个问题并不适用，因为智能合约没有私钥。

## 2. 尽可能避免在 calldata 中使用有符号整数

由于 [Solidity 使用二进制补码](https://www.rareskills.io/post/signed-int-solidity)来表示有符号整数，小的负数在 calldata 中将大部分为非零。例如，-1 在二进制补码形式下为 0xff..ff，因此更昂贵。

## 3. Calldata （通常）比内存更便宜

直接从 calldata 中加载函数输入或数据比从内存中加载更便宜。这是因为从 calldata 访问数据涉及的操作和 gas 成本较少。因此，建议仅在函数需要修改数据时使用内存（calldata 无法修改）。

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

## 4. 考虑对 calldata 进行打包，尤其是在 L2 上

Solidity 会自动对存储变量进行打包，但是在 calldata 中，对于那些本应该被打包的变量，其 abi 编码并不会进行打包。

这是一种相当极端的优化方式，会导致代码复杂度增加，但是如果一个函数接收大量的 calldata，这是值得考虑的。

ABI 编码并不适用于每种数据表示方式，某些数据表示方式可以通过应用特定的方式进行更高效的编码。

