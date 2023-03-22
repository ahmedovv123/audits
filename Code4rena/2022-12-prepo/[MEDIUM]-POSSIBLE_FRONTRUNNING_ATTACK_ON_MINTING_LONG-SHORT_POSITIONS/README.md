# Original link
https://github.com/code-423n4/2022-12-prepo-findings/issues/312
# Lines of code

https://github.com/prepo-io/prepo-monorepo/blob/feat/2022-12-prepo/apps/smart-contracts/core/contracts/PrePOMarket.sol#L65-L74


# Vulnerability details

## Impact
```createMarket``` function in ```PrePOMarketFactory.sol``` contract creates a new ```PrePOMarket``` contract. Salt is used for creating the contract which is computed from ```_createPairTokens``` function. Variables passed to this function are visible from anyone (they are input parameter for the ```createMarket``` function).
After the ```PrePOMarket``` contract is created, only the team should mint Long and Short Tokens ([Reference](https://github.com/prepo-io/prepo-monorepo/blob/feat/2022-12-prepo/apps/smart-contracts/core/contracts/interfaces/IPrePOMarket.sol#L68-L78)). This check is done with ```hook``` function from ```MintHook.sol``` contract. When the contract is created, ```_mintHook``` variable points to zero address, which means that the check [here](https://github.com/prepo-io/prepo-monorepo/blob/feat/2022-12-prepo/apps/smart-contracts/core/contracts/PrePOMarket.sol#L68) will not execute.
Since all transactions are visible in the mempool for a short while before being executed, there could be an observer which can see the ```PrePOMarket``` contract creation, compute the contract's address and call ```mint``` function before the team has set the ```mintHook```'s address. This can break the team's market economy or other things which depend on Long/Short Tokens.

## Proof of Concept


## Tools Used
Manual

## Recommended Mitigation Steps
One way is to set the ```mintHook``` variable in ```PrePOMarket.sol``` constructor when creating the contract in ```createMarket``` function. The other way is to use some type of enhanced commit and reveal schemes.