# Original link
https://github.com/code-423n4/2022-12-escher-findings/issues/119
# Lines of code

https://github.com/code-423n4/2022-12-escher/blob/main/src/minters/LPDA.sol#L58-L89


# Vulnerability details

## Impact

The ```buy()``` function will call ```_end()``` when the ```finalId``` is reached, but before that it depends on ```transfer``` function which can fail if ```saleReceiver``` is smart contract without ```receive()``` or ```callback()``` function. This could lead to ```DoS``` of the LPDA edition and fees will never be transfered to ```feeReceiver```.

## Proof of Concept

Consider following test:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;

import "forge-std/Test.sol";
import {EscherTest} from "./utils/EscherTest.sol";
import {LPDAFactory, LPDA} from "src/minters/LPDAFactory.sol";

contract Receiver {
    // without receive() and fallback()
}

contract LPDABase is EscherTest {
    LPDAFactory public lpdaSales;
    LPDA.Sale public lpdaSale;
    Receiver receiver;

    function setUp() public virtual override {
        super.setUp();
        receiver = new Receiver();
        lpdaSales = new LPDAFactory();
        // set up a LPDA Sale
        lpdaSale = LPDA.Sale({
            currentId: uint48(0),
            finalId: uint48(10),
            edition: address(edition),
            startPrice: uint80(uint256(1 ether)),
            finalPrice: uint80(uint256(0.1 ether)),
            dropPerSecond: uint80(uint256(0.1 ether) / 1 days),
            startTime: uint96(block.timestamp),
            saleReceiver: payable(address(receiver)), // Here we set the contract address
            endTime: uint96(block.timestamp + 1 days)
        });
    }
}

contract LPDADOSTest is LPDABase {
    LPDA public sale;
    event End(LPDA.Sale _saleInfo);

    function test_Buy() public {
        sale = LPDA(lpdaSales.createLPDASale(lpdaSale));
        // authorize the lpda sale to mint tokens
        edition.grantRole(edition.MINTER_ROLE(), address(sale));
        //lets buy an NFT
        sale.buy{value: 1 ether}(1);
        assertEq(address(sale).balance, 1 ether);
    }

    function test_LPDA() public {
        // make the lpda sales contract
        sale = LPDA(lpdaSales.createLPDASale(lpdaSale));
        // authorize the lpda sale to mint tokens
        edition.grantRole(edition.MINTER_ROLE(), address(sale));
        //lets buy an NFT
        sale.buy{value: 1 ether}(1);
        assertEq(address(sale).balance, 1 ether);

        vm.warp(block.timestamp + 1 days);
        assertApproxEqRel(sale.getPrice(), 0.9 ether, lpdaSale.dropPerSecond);

        // buy the rest
        // this will auto end the sale
        sale.buy{value: uint256((0.9 ether + lpdaSale.dropPerSecond) * 9)}(9); // When we try to buy rest it will revert and the fees wil not be sent 

        vm.warp(block.timestamp + 2 days);

        // now lets get a refund
        uint256 bal = address(this).balance;
        sale.refund();
        assertApproxEqRel(address(this).balance, bal + 0.1 ether, lpdaSale.dropPerSecond);
    }
}
```

Test results:

```
    │   │   ├─ [24] Receiver::fallback{value: 8550000000000334400}() 
    │   │   │   └─ ← "EvmError: Revert"
    │   │   └─ ← "EvmError: Revert"
    │   └─ ← "EvmError: Revert"
    └─ ← "EvmError: Revert"

Test result: FAILED. 1 passed; 1 failed; finished in 3.82ms

Failing tests:
Encountered 1 failing test in test/LPDADOS.t.sol:LPDADOSTest
[FAIL. Reason: EvmError: Revert] test_LPDA() (gas: 660235)
```

## Tools Used

Forge

## Recommended Mitigation Steps

One approach is to use [pull over push](https://fravoll.github.io/solidity-patterns/pull_over_push.html) pattern. This way everyone can withdraw their balance, in this scenario - their ```totalSale``` amount.

Other solution is to make sure that ```saleReceiver``` is not a contract address using ```isContract()``` function from ```Address``` [library](https://docs.openzeppelin.com/contracts/3.x/api/utils#Address) of OpenZeppelin.