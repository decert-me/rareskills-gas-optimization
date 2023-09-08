[**Outdated tricks**](##Outdated tricks)

- [1. external is cheaper than public](##1. external is cheaper than public)
- [2. != 0 is cheaper than > 0](##2. != 0 is cheaper than > 0)

## Outdated tricks

## 1. external is cheaper than public

You should still prefer the external modifier for clarity sake if the function cannot be called inside the contract, but it has no effect on gas savings.

## 2. != 0 is cheaper than > 0

Somewhere around solidity 0.8.12 or so, this stopped being true. If you are forced to use an old version, you can still benchmark it.

## 