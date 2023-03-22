# Original link
https://github.com/code-423n4/2022-12-escher-findings/issues/5
# [G-01] USE DIRECTLY ```block``` VALUES

main/src/uris/Generative.sol: [L22-L24](https://github.com/code-423n4/2022-12-escher/blob/main/src/uris/Generative.sol#L22-L24)

```diff
index db09737..f76998e 100644
--- a/src/uris/Generative.sol
+++ b/src/uris/Generative.sol
@@ -19,9 +19,7 @@ contract Generative is Unique {
     function setSeedBase() external onlyOwner {
         require(seedBase == bytes32(0), "SEED BASE SET");
 
-        uint256 time = block.timestamp;
-        uint256 numb = block.number;
-        seedBase = keccak256(abi.encodePacked(numb, blockhash(numb - 1), time, (time % 200) + 1));
+        seedBase = keccak256(abi.encodePacked(block.number, blockhash(block.number - 1), block.timestamp, (block.timestamp % 200) + 1));
     }
```

Saves

```
testSetSeedBase() (gas: -10 (-0.030%)) 
```

# [G-02] UNCHECKING ARITHMETICS OPERATIONS THAT CAN’T UNDERFLOW/OVERFLOW (3 INSTANCES)

main/src/minters/FixedPrice.sol: [65](https://github.com/code-423n4/2022-12-escher/blob/main/src/minters/FixedPrice.sol#L65)

```diff
-        for (uint48 x = sale_.currentId + 1; x <= newId; x++) {
+        for (uint48 x = sale_.currentId + 1; x <= newId;) {
             nft.mint(msg.sender, x);
+            unchecked { ++x; }
```

Saves

```
test_Buy() (gas: -82 (-0.027%)) 
```


main/src/minters/OpenEdition.sol: [66](https://github.com/code-423n4/2022-12-escher/blob/main/src/minters/OpenEdition.sol#L66)

```diff
-        for (uint24 x = temp.currentId + 1; x <= newId; x++) {
+        for (uint24 x = temp.currentId + 1; x <= newId;) {
             nft.mint(msg.sender, x);
+            unchecked { ++x; }
         }
```

Saves

```
test_Buy() (gas: -82 (-0.027%)) 
```

main/src/minters/LPDA.sol: [73](https://github.com/code-423n4/2022-12-escher/blob/main/src/minters/LPDA.sol#L73)

```diff
-        for (uint256 x = temp.currentId + 1; x <= newId; x++) {
+        for (uint256 x = temp.currentId + 1; x <= newId;) {
             nft.mint(msg.sender, x);
+            unchecked { ++x; }
         }
```

Saves

```
test_Buy() (gas: -63 (-0.017%))
```

# [G-03] X = X + Y IS MORE EFFICIENT, THAN X += Y (3 INSTANCES)

main/src/minters/LPDA.sol: [66](https://github.com/code-423n4/2022-12-escher/blob/main/src/minters/LPDA.sol#L66), [70](https://github.com/code-423n4/2022-12-escher/blob/main/src/minters/LPDA.sol#L66), [71](https://github.com/code-423n4/2022-12-escher/blob/main/src/minters/LPDA.sol#L71)

```diff
-        amountSold += amount;
+        amountSold = amountSold + amount;
         uint48 newId = amount + temp.currentId;
         require(newId <= temp.finalId, "TOO MANY");
 
-        receipts[msg.sender].amount += amount;
-        receipts[msg.sender].balance += uint80(msg.value);
+        receipts[msg.sender].amount = receipts[msg.sender].amount + amount;
+        receipts[msg.sender].balance = receipts[msg.sender].balance + uint80(msg.value);
```

Saves

```
test_Buy() (gas: -210 (-0.072%))
```

Results:
```
test_WhenEnded_Finalize() (gas: -82 (-0.023%)) 
test_RevertsWhenTooSoon_Buy() (gas: -82 (-0.026%)) 
test_RevertsWhenEnded_Buy() (gas: -82 (-0.026%)) 
test_RevertsWhenTooSoon_Buy() (gas: -82 (-0.026%)) 
test_RevertsTooMuchValue_Buy() (gas: -82 (-0.026%)) 
test_RevertsWhenTooMany_Buy() (gas: -82 (-0.026%)) 
test_RevertsTooMuchValue_Buy() (gas: -82 (-0.026%)) 
test_RevertsWhenNotOwner_TransferOwnership() (gas: -82 (-0.027%)) 
test_RevertsWhenNotOwner_TransferOwnership() (gas: -82 (-0.027%)) 
test_RevertsWhenNotEnded_Finalize() (gas: -82 (-0.027%)) 
test_RevertsWhenTooLittleValue_Buy() (gas: -82 (-0.027%)) 
test_RevertsWhenNotOwner_Cancel() (gas: -82 (-0.027%)) 
test_RevertsWhenTooLate_Cancel() (gas: -82 (-0.027%)) 
test_RevertsWhenTooLittleValue_Buy() (gas: -82 (-0.027%)) 
test_RevertsWhenTooLate_Cancel() (gas: -82 (-0.027%)) 
test_RevertsWhenNotOwner_Cancel() (gas: -82 (-0.027%)) 
test_Buy() (gas: -82 (-0.027%)) 
test_Buy() (gas: -82 (-0.027%)) 
test_SetSeedBase() (gas: -10 (-0.030%)) 
test_RevertsWhenAlreadyRefunded_Refund() (gas: -273 (-0.069%)) 
test_Refund() (gas: -273 (-0.069%)) 
test_WhenNotOver_Refund() (gas: -273 (-0.069%)) 
test_RevertsWhenTooSoon_Buy() (gas: -273 (-0.069%)) 
test_RevertsWhenNoRefund_Refund() (gas: -273 (-0.070%)) 
test_RevertsWhenTooLittleValue_Buy() (gas: -273 (-0.070%)) 
test_RevertsWhenTooLate_Cancel() (gas: -273 (-0.071%)) 
test_RevertsWhenNotOwner_Cancel() (gas: -273 (-0.071%)) 
test_Buy() (gas: -273 (-0.072%)) 
test_RevertsWhenMintedOut_Buy() (gas: -820 (-0.133%)) 
test_WhenMintsOut_Buy() (gas: -820 (-0.135%)) 
test_LPDA() (gas: -1050 (-0.150%)) 
test_RevertsWhenEnded_Buy() (gas: -1050 (-0.153%)) 
test_SellsOut_Buy() (gas: -1050 (-0.153%)) 
test_RevertsWhenSoldOut_Buy() (gas: -1092 (-0.157%)) 
```
```
Overall gas change: -9825 (-2.015%)
```