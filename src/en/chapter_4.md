# Design Patterns

- [1. Use multidelegatecall to batch transactions](#1-use-multidelegatecall-to-batch-transactions)
- [2. Use ECDSA signatures instead of merkle trees for allowlists and airdrops](#2-use-ecdsa-signatures-instead-of-merkle-trees-for-allowlists-and-airdrops)
- [3. Use ERC20Permit to batch the approval and transfer step in on transaction](#3-use-erc20permit-to-batch-the-approval-and-transfer-step-in-on-transaction)
- [4. Use L2 message passing for games or other high-throughput, low transaction value applications](#4-use-l2-message-passing-for-games-or-other-high-throughput-low-transaction-value-applications)
- [5. Use state-channels if applicable](#5-use-state-channels-if-applicable)
- [6. Use voting delegation as a gas saving measure](#6-use-voting-delegation-as-a-gas-saving-measure)
- [7. ERC1155 is a cheaper non-fungible token than ERC721](#7-erc1155-is-a-cheaper-non-fungible-token-than-erc721)
- [8. Use one ERC1155 or ERC6909 token instead of several ERC20 tokens](#8-use-one-erc1155-or-erc6909-token-instead-of-several-erc20-tokens)
- [9. The UUPS upgrade pattern is more gas efficient for users than the Transparent Upgradeable Proxy](#9-the-uups-upgrade-pattern-is-more-gas-efficient-for-users-than-the-transparent-upgradeable-proxy)
- [10. Consider using alternatives to OpenZeppelin](#10-consider-using-alternatives-to-openzeppelin)


## 1. Use multidelegatecall to batch transactions

Multi-delegatecall helps the msg.sender to call multiple functions within a contract while preserving the env vars like msg.sender and msg.value .

*Note*: Be mindful that since msg.value is persistent, it can lead to issues that the developer need to address while inheriting multi delegatecall in their contract.

Example of Multi delegatecall is Uniswap’s implementation below:

```
function multicall(bytes[] calldata data) public payable override returns (bytes[] memory results) {
        results = new bytes[](data.length);
        for (uint256 i = 0; i < data.length; i++) {
            (bool success, bytes memory result) = address(this).delegatecall(data[i]);

            if (!success) {
                // Next 5 lines from https://ethereum.stackexchange.com/a/83577
                if (result.length < 68) revert();
                assembly {
                    result := add(result, 0x04)
                }
                revert(abi.decode(result, (string)));
            }

            results[i] = result;
        }
    }
```

## 2. Use ECDSA signatures instead of merkle trees for allowlists and airdrops

Merkle trees use a considerable amount of calldata and increase in cost with the size of the merkle proof. Generally, using digital signatures is cheaper gas-wise compared to merkle proofs.

## 3. Use ERC20Permit to batch the approval and transfer step in on transaction

ERC20 Permit has an additional function that accepts digital signatures from a toke holder to increase the approval for another address. This way, the recipient of the approval can submit the permit transaction and the transfer in one transaction. The user granting the permit does not have to pay any gas, and the recipient of the permit can batch the permit and transferFrom transaction into a single transaction.

## 4. Use L2 message passing for games or other high-throughput, low transaction value applications

[Etherorcs](https://github.com/EtherOrcsOfficial/etherOrcs-contracts) was one of the early pioneers of this pattern, so you can look to their Github (linked above) for inspiration. The idea is that assets on Ethereum can be “bridge” (via message passing) to another chain such as Polygon, Optimism, or Arbitrum and the game can be conducted there where transactions are cheap.

## 5. Use state-channels if applicable

State channels are probably the oldest, but still useable scalability solutions for Ethereum. Unlike L2s, they are application specific. Rather than users commiting their transactions to a chain, they commit assets to a smart contract then share binding signatures with each other as state transitions. When the operation is over, they then commit the final result to the chain.

If one of the participants is dishonest, then an honest participant can use the counterparty’s signature to force the smart contract to release their assets.

## 6. Use voting delegation as a gas saving measure

Our tutorial on [ERC20 Votes](https://www.rareskills.io/post/erc20-votes-erc5805-and-erc6372) describes this pattern in more detail. Instead of every token owner voting, only the delegates vote, which net reduces the number of voting transactions.

## 7. ERC1155 is a cheaper non-fungible token than ERC721

The ERC721 balanceOf function is rarely used in practice but adds a storage overhead whenever a mint and transfer happens. ERC1155 tracks balance per id, and also uses the same balance to track ownership of the id. If the maximum supply for each id is one, then the token becomes non-fungible per each id.

## 8. Use one ERC1155 or ERC6909 token instead of several ERC20 tokens

This was the original intent of the ERC1155 token. Each individual token behaves like and ERC20, but only one contract needs to be deployed.

The drawback of this approach is that the tokens will not be compatible with most DeFi swapping primitives.

ERC1155 uses callbacks on all of the transfer methods. If this is not desired, [ERC6909](https://eips.ethereum.org/EIPS/eip-6909) can be used instead.

## 9. The UUPS upgrade pattern is more gas efficient for users than the Transparent Upgradeable Proxy

The transparent upgradeable proxy pattern requires reading the admin address out of the storage slot (defined by ERC-1967) every time a transaction happens, to see if the caller is the admin or not. This introduces an extra storage read.

## 10. Consider using alternatives to OpenZeppelin

OpenZeppelin is a great and popular smart contract library, but there are other alternatives that are worth considering. These alternatives offer better gas efficiency and have been tested and recommended by developers.

Two examples of such alternatives are [Solmate](https://github.com/transmissions11/solmate) and [Solady](https://github.com/Vectorized/solady).

Solmate is a library that provides a number of gas-efficient implementations of common smart contract patterns. Solady is another gas-efficient library that places a strong emphasis on using assembly.



