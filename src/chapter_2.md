# Saving Gas On Deployment
1. Use the account nonce to predict the addresses of interdependent smart contracts thereby avoiding storage variables and address setter functions

2. Make constructors payable

3. Deployment size can be reduced by optimizing the IPFS hash to have more zeros (or using the --no-cbor-metadata compiler option)

4. Use selfdestruct in the constructor if the contract is one-time use

5. Understand the trade-offs when choosing between internal functions and modifiers

6. Use clones or metaproxies when deploying very similar smart contracts that are not called frequently

7. Admin functions can be payable

8. Custom errors are (usually) smaller than require statements

9. Use existing create2 factories instead of deploying your own