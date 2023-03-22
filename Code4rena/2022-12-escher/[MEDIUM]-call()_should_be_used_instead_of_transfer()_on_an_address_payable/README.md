# Original link
https://github.com/code-423n4/2022-12-escher-findings/issues/96
# Lines of code

https://github.com/code-423n4/2022-12-escher/blob/main/src/minters/LPDA.sol#L85
https://github.com/code-423n4/2022-12-escher/blob/main/src/minters/LPDA.sol#L86
https://github.com/code-423n4/2022-12-escher/blob/main/src/minters/LPDA.sol#L105
https://github.com/code-423n4/2022-12-escher/blob/main/src/minters/FixedPrice.sol#L109
https://github.com/code-423n4/2022-12-escher/blob/main/src/minters/OpenEdition.sol#L92


# Vulnerability details

## Impact
The use of the deprecated ```transfer()``` function for an address will inevitably make the transaction fail when:

* The claimer smart contract does not implement a payable function.
* The claimer smart contract does implement a payable fallback which uses more than 2300 gas unit.
* The claimer smart contract implements a payable fallback function that needs less than 2300 gas units but is called through proxy, raising the call's gas usage above 2300.

Additionally, using higher than 2300 gas might be mandatory for some multisig wallets.

## Proof of concept 

```solidity
src/minters/FixedPrice.sol:
  109:         ISaleFactory(factory).feeReceiver().transfer(address(this).balance / 20);

src/minters/LPDA.sol:
   85:         ISaleFactory(factory).feeReceiver().transfer(fee);
   86:         temp.saleReceiver.transfer(totalSale - fee);
  105:         payable(msg.sender).transfer(owed);

src/minters/OpenEdition.sol:
   92:         ISaleFactory(factory).feeReceiver().transfer(address(this).balance / 20);
```


## Tools Used

Manual

## Recommended Mitigation Steps

Recommended using ```call()``` instead of ```transfer()```.