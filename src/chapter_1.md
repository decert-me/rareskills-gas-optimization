# The RareSkills Book of Gas Optimization

## [Gas optimization tricks do not always work](./ Gas optimization tricks do not always work)

## Beware of complexity and readability

## Comprehensive treatment of each topic isn’t possible here

## We do not discuss application-specific tricks

1. Most important: avoid zero to one storage writes where possible

2. Cache storage variables: write and read storage variables exactly once

3. Pack related variables

4. Pack structs

5. Keep strings smaller than 32 bytes

6. Variables that are never updated should be immutable or constant

7. Using mappings instead of arrays to avoid length checks

8. Using unsafeAccess on arrays to avoid redundant length checks

9. Use bitmaps instead of bools when a significant amount of booleans are used

10. Use SSTORE2 or SSTORE3 to store a lot of data

11. Use storage pointers instead of memory where appropriate

12. Avoid having ERC20 token balances go to zero, always keep a small amount

13. Count from n to zero instead of counting from zero to n


### Gas optimization tricks do not always work
Some gas optimization tricks only work in a certain context. For example, intuitively, it would seem that

if (!cond) {
    // branch False
}
else {
    // branch True
}
is less efficient than

if (cond) {
    // branch True
}
else {
    // branch False
}
because extra opcodes are spent inverting the condition. Counterintuitively, there are many cases where this optimization actually increases the cost of the transaction. The solidity compiler can be unpredictable sometimes.

Therefore, you should actually measure the effect of the alternatives before settling on a certain algorithm. Think of some of these tricks are bringing awareness to areas where the compiler may be surprising.


Some tricks that are not universal are marked as such in this document. Gas optimization tricks sometimes depend on what the compiler is doing locally. You should generally test both the optimal version of the code and the non-optimal version to see you actually get an improvement. We will document some surprising cases where what should lead to an optimization actually leads to higher cost.

Second, some of these optimization behavior may change when using the --via-ir option on the Solidity compiler.

