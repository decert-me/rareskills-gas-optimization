[**Cross contract calls**](##Cross-contract-calls)

- [1. Use transfer hooks for tokens instead of initiating a transfer from the destination smart contract](##1.-Use-transfer-hooks-for-tokens-instead-of-initiating-a-transfer-from-the-destination-smart-contract)
- [2. Use fallback or receive instead of deposit() when transferring Ether](##2.-Use-fallback-or-receive-instead-of-deposit()-when-transferring-Ether)
- [3. Use ERC2930 access list transactions when making cross-contract calls to pre-warm storage slots](##3.-Use-ERC2930-access-list-transactions-when-making-cross-contract-calls-to-pre-warm-storage-slots-and-contract-addresses)
- [4. Cache calls to external contracts where it makes sense (like caching return data from chainlink oracle)](##4.-Cache-calls-to-external-contracts-where-it-makes-sense-(like-caching-return-data-from-chainlink-oracle))
- [5. Implement multicall in router-like contracts](##5.-Implement-multicall-in-router-like-contracts)
- [6. Avoid contract calls by making the architecture monolithic](##6.-Avoid-contract-calls-by-making-the-architecture-monolithic)


## Cross contract calls

## 1. Use transfer hooks for tokens instead of initiating a transfer from the destination smart contract

Let’s say you have contract A which accepts token B (an NFT or an ERC1363 token). The naive workflow is as follows:

1. msg.sender approves contract A to accept token B
2. msg.sender calls contract A to transfer tokens from msg.sender to A
3. Contract A then calls token B to do the transfer
4. Token B does the transfer, and calls onTokenReceived() in contract A
5. Contract A returns a value from onTokenReceived() to token B
6. Token B returns execution to contract A

This is very inefficient. It’s better for msg.sender to call contract B to do a transfer which calls the tokenReceived hook in contract A.

Note that:

- All ERC1155 tokens include a transfer hook
- safeTransfer and safeMint in ERC721 have a transfer hook
- ERC1363 has transferAndCall
- ERC777 has a transfer hook but has been deprecated. Use ERC1363 or ERC1155 instead if you need fungible tokens

If you need to pass arguments to contract A, simply use the data field and parse that in contract A.

## 2. Use fallback or receive instead of deposit() when transferring Ether

Similar to above, you can “just transfer” ether to a contract and have it respond to the transfer instead of using a payable function. This of course, depends on the rest of the contract’s architecture.

**Example Deposit in AAVE**

```
contract AddLiquidity{

    receive() external payable {
      IWETH(weth).deposit{msg.value}();
      AAVE.deposit(weth, msg.value, msg.sender, REFERRAL_CODE)
    }
}
```

The fallback function is capable of receiving bytes data which can be parsed with abi.decode. This servers as an alternative to supplying arguments to a deposit function.

## 3. Use ERC2930 access list transactions when making cross-contract calls to pre-warm storage slots and contract addresses

Access list transactions allow you to prepay the gas costs for some storage and call operations, with a 200 gas discount. This can save gas on further state or storage access, which is paid as a warm access.

If your transaction will make a cross-contract call, you should almost certainly be using access list transactions.

When calling clones or proxies which always involve a cross-contract call via delegatecall, you should make the transaction an access list transaction.

We have a dedicated blog post on this, visit https://www.rareskills.io/post/eip-2930-optional-access-list-ethereum to learn more.

## 4. Cache calls to external contracts where it makes sense (like caching return data from chainlink oracle)

Caching data is generally recommended to avoid duplication in memory when you want to use the same data > 1 times during a single execution process.

Obvious example is if you need to make multiple operations, say, using ETH price gotten from chainlink. You store the price in memory, instead of making the expensive external call again.

## 5. Implement multicall in router-like contracts

This is a common feature, such as the Uniswap Router and the Compound Bulker.

If you expect your users to make a sequence of calls, have a contract batch them together using multicall.

## 6. Avoid contract calls by making the architecture monolithic

Contract calls are expensive, and the best way to save gas on them is by not using them at all. There is a natural tradeoff with this, but having several contracts that talk to each other can sometimes increase gas and complexity rather than manage it.

