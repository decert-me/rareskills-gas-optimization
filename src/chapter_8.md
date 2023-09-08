[**Dangerous techniques**](##Dangerous%20techniques)

- [1. Use gasprice() or msg.value to pass information](##1.%20Use%20gasprice()%20or%20msg.value%20to%20pass%20information)
- [2. Manipulate environment variables like coinbase() or block.number if the tests allow it](##2.%20Manipulate%20environment%20variables%20like%20coinbase()%20or%20block.number%20if%20the%20tests%20allow%20it)
- [3. Use gasleft() to branch decisions at key points](##3.%20Use%20gasleft()%20to%20branch%20decisions%20at%20key%20points)
- [4. Use send() to move ether, but don’t check for success](##4.%20Use%20send()%20to%20move%20ether,%20but%20don’t%20check%20for%20success)
- [5. Make all functions payable](##5.%20Make%20all%20functions%20payable)
- [6. External library jumping](##6.%20External%20library%20jumping)
- [7. Append bytecode to the end of the contract to create a highly optimized subroutine](##7.%20Append%20bytecode%20to%20the%20end%20of%20the%20contract%20to%20create%20a%20highly%20optimized%20subroutine)


## Dangerous techniques

If you are participating in a gas optimization contest, then these unusual design patterns can help, but using them in production is highly discouraged, or at the very least should be done with extreme caution.

## 1. Use gasprice() or msg.value to pass information

Passing parameters to a function will at bare minimum add 128 gas, because each zero byte of calldata costs 4 gas. However, you can set the gasprice or msg.value for free to pass numbers that way. This of course won’t work in production because msg.value costs real Ethereum and if your gas price is too low, the transaction won’t go through, or will waste cryptocurrency.

## 2. Manipulate environment variables like coinbase() or block.number if the tests allow it

This of course will not work in production, but it can serve as a side channel to modify a smart contract’s behavior.

## 3. Use gasleft() to branch decisions at key points

Gas is used up as the execution progresses, so if you want to do something like terminate a loop after a certain point or change behavior in a later part of the execution, you can use the gasprice() functionality to branch decision making. gasleft() decrements for “free” so this saves gas.

## 4. Use send() to move ether, but don’t check for success

The difference between send and transfer is that transfer reverts if the transfer fails, but send returns false. However, you can just ignore the return value of send, and that will result in fewer op codes. Ignoring return values is a very bad practice, and it’s a shame the compiler doesn’t stop you from doing that. In production systems, you should not use send() at all because of the gas limit.

## 5. Make all functions payable

This is a controversial optimization because it allows for an unexpected state change in a transaction, and doesn’t save that much gas. But in the context of a gas contest, make all functions payable to avoid the extra opcodes that check if msg.value is nonzero.

As noted earlier, setting the constructor or admin functions to be payable is a legitimate way to save gas, since presumably the deployer and admin know what they are doing and can do more destructive things than send ether.

## 6. External library jumping

Solidity traditionally uses 4 bytes and a jump table to determine which function to use. However, one can (very unsafely!) simply supply the jump destination as a calldata argument, reducing the “function selector” down to one byte and avoiding the jump table altogether. More information can be seen in this [tweet](https://twitter.com/AmadiMichaels/status/1697405235948310627).

## 7. Append bytecode to the end of the contract to create a highly optimized subroutine

Some computationally intensive algorithms, such as hash functions, are better written in raw bytecode rather than Solidity, or even Yul. For example, [Tornado Cash](https://www.rareskills.io/post/how-does-tornado-cash-work) writes the MiMC hash function as a separate smart contract, written directly in raw bytecode. Avoid the extra 2,600 or 100 gas cost (cold or warm access) of another smart contract by appending that bytecode to the actual contract and jumping back and forth from it. Here is a [proof of concept using Huff](https://twitter.com/AmadiMichaels/status/1696263027920634044).

