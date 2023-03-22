# Original link
https://github.com/code-423n4/2022-12-caviar-findings/issues/8
# [G-01] USE MOST EFFICIENT WAY TO CONVERT BYTES TO STRING

https://github.com/code-423n4/2022-12-caviar/blob/main/src/lib/SafeERC20Namer.sol

```diff
--- a/src/lib/SafeERC20Namer.sol
+++ b/src/lib/SafeERC20Namer.sol
@@ -7,40 +7,10 @@ import "openzeppelin/utils/Strings.sol";
 // produces token descriptors from inconsistent or absent ERC20 symbol implementations that can return string or bytes32
 // this library will always produce a string symbol to represent the token
 library SafeERC20Namer {
-    function bytes32ToString(bytes32 x) private pure returns (string memory) {
-        bytes memory bytesString = new bytes(32);
-        uint256 charCount = 0;
-        for (uint256 j = 0; j < 32; j++) {
-            bytes1 char = x[j];
-            if (char != 0) {
-                bytesString[charCount] = char;
-                charCount++;
-            }
-        }
-
-        bytes memory bytesStringTrimmed = new bytes(charCount);
-        for (uint256 j = 0; j < charCount; j++) {
-            bytesStringTrimmed[j] = bytesString[j];
-        }
-
-        return string(bytesStringTrimmed);
-    }
 
     // assumes the data is in position 2
     function parseStringData(bytes memory b) private pure returns (string memory) {
-        uint256 charCount = 0;
-        // first parse the charCount out of the data
-        for (uint256 i = 32; i < 64; i++) {
-            charCount <<= 8;
-            charCount += uint8(b[i]);
-        }
-
-        bytes memory bytesStringTrimmed = new bytes(charCount);
-        for (uint256 i = 0; i < charCount; i++) {
-            bytesStringTrimmed[i] = b[i + 64];
-        }
-
-        return string(bytesStringTrimmed);
+        return string(abi.encodePacked(b));
     }

// uses a heuristic to produce a token name from the address
@@ -59,17 +29,11 @@ library SafeERC20Namer {
     function callAndParseStringReturn(address token, bytes4 selector) private view returns (string memory) {
         (bool success, bytes memory data) = token.staticcall(abi.encodeWithSelector(selector));
         // if not implemented, or returns empty data, return empty string
-        if (!success || data.length == 0) {
+        if (!success) {
             return "";
         }
-        // bytes32 data always has length 32
-        if (data.length == 32) {
-            bytes32 decoded = abi.decode(data, (bytes32));
-            return bytes32ToString(decoded);
-        } else if (data.length > 64) {
-            return abi.decode(data, (string));
-        }
-        return "";
+        
+        return string(abi.encodePacked(data));
     }
``` 

Gas report diff results on ```Create.t.sol```

```bash
# forge snapshot --diff .gas-snapshot --match-contract Create

Running 6 tests for test/Caviar/Create.t.sol:CreateTest
[PASS] testItEmitsCreateEvent() (gas: 3424124)
[PASS] testItReturnsPair() (gas: 3419438)
[PASS] testItRevertsIfDeployingSamePairTwice() (gas: 3423667)
[PASS] testItSavesPair() (gas: 3420842)
[PASS] testItSavesPair(address,address,bytes32) (runs: 256, μ: 3457811, ~: 3475546)
[PASS] testItSetsSymbolsAndNames() (gas: 3486364)
Test result: ok. 6 passed; 0 failed; finished in 234.20ms
testItSetsSymbolsAndNames() (gas: -423593 (-10.834%)) 
testItRevertsIfDeployingSamePairTwice() (gas: -423593 (-11.010%)) 
testItSavesPair() (gas: -423582 (-11.018%)) 
testItReturnsPair() (gas: -423687 (-11.025%)) 
Overall gas change: -1694455 (-43.887%)
```

# [G-02] UNCHECKING ARITHMETICS OPERATIONS THAT CAN’T UNDERFLOW/OVERFLOW (3 INSTANCES)

src/Pair.sol: [238](https://github.com/code-423n4/2022-12-caviar/blob/main/src/Pair.sol#L238), [258](https://github.com/code-423n4/2022-12-caviar/blob/main/src/Pair.sol#L258), [468](https://github.com/code-423n4/2022-12-caviar/blob/main/src/Pair.sol#L468)

```diff
--- a/src/Pair.sol
+++ b/src/Pair.sol
@@ -235,8 +235,10 @@ contract Pair is ERC20, ERC721TokenReceiver {
         // *** Interactions *** //
 
         // transfer nfts from sender
-        for (uint256 i = 0; i < tokenIds.length; i++) {
+        for (uint256 i = 0; i < tokenIds.length;) {
             ERC721(nft).safeTransferFrom(msg.sender, address(this), tokenIds[i]);
+
+            unchecked { ++i; }
         }
 
         emit Wrap(tokenIds);
@@ -255,8 +257,10 @@ contract Pair is ERC20, ERC721TokenReceiver {
         // *** Interactions *** //
 
         // transfer nfts to sender
-        for (uint256 i = 0; i < tokenIds.length; i++) {
+        for (uint256 i = 0; i < tokenIds.length;) {
             ERC721(nft).safeTransferFrom(address(this), msg.sender, tokenIds[i]);
+
+            unchecked { ++i; }
         }
 
         emit Unwrap(tokenIds);
@@ -465,9 +469,11 @@ contract Pair is ERC20, ERC721TokenReceiver {
         if (merkleRoot == bytes23(0)) return;
 
         // validate merkle proofs against merkle root
-        for (uint256 i = 0; i < tokenIds.length; i++) {
+        for (uint256 i = 0; i < tokenIds.length;) {
             bool isValid = MerkleProofLib.verify(proofs[i], merkleRoot, keccak256(abi.encodePacked(tokenIds[i])));
             require(isValid, "Invalid merkle proof");
+
+            unchecked { ++i; }
         }
     }
```

## OVERAL GAS REPORT DIFF

```bash
# forge snapshot --diff .gas-snapshot

testItReturnsBaseTokenAmountAndFractionalTokenAmount() (gas: 11 (0.011%)) 
testItReturnsInputAmount() (gas: -19 (-0.034%)) 
testItTransfersBaseTokens() (gas: -27 (-0.045%)) 
testItRefundsSurplusEther() (gas: -30 (-0.051%)) 
testItTransfersBaseTokens() (gas: 61 (0.060%)) 
testItRevertsSlippageOnSell() (gas: -15 (-0.062%)) 
testItTransfersBaseTokens() (gas: 53 (0.086%)) 
testItRevertsFractionalTokenSlippage() (gas: -30 (-0.105%)) 
testItInitMintsLpTokensToSender() (gas: 138 (0.111%)) 
testItTransfersBaseTokens() (gas: 144 (0.114%)) 
testItBurnsLpTokens() (gas: 129 (0.127%)) 
testItTransfersEther() (gas: 37 (0.136%)) 
testItTransfersEther() (gas: 168 (0.172%)) 
testItTransfersEther() (gas: 128 (0.183%)) 
testItTransfersEther() (gas: 90 (0.184%)) 
testItReturnsOutputAmount() (gas: -99 (-0.187%)) 
testItTransfersFractionalTokens() (gas: 109 (0.190%)) 
testItTransfersFractionalTokens() (gas: 197 (0.197%)) 
testItBuysSellsEqualAmounts(uint256) (gas: 436 (0.203%)) 
testItReturnsOutputAmount() (gas: 365 (0.212%)) 
testItTransfersFractionalTokens() (gas: 268 (0.216%)) 
testItRevertsSlippageAfterInitMint() (gas: 301 (0.225%)) 
testItInitMintsLpTokensToSender() (gas: 606 (0.232%)) 
testItTransfersBaseTokens() (gas: 612 (0.233%)) 
testItTransfersBaseTokens() (gas: 437 (0.244%)) 
testItRevertsBaseTokenSlippage() (gas: 72 (0.253%)) 
testItRevertsSlippageOnSell() (gas: 449 (0.275%)) 
testItMintsFractionalTokens() (gas: 505 (0.286%)) 
testItTransfersFractionalTokens() (gas: 189 (0.316%)) 
testItMintsLpTokensAfterInit() (gas: 1793 (0.328%)) 
testItMintsLpTokensAfterInit() (gas: 2866 (0.337%)) 
testItAddsBuysSellsRemovesCorrectAmount(uint256,uint256,uint256) (gas: 2729 (0.348%)) 
testItRevertsSlippageOnInitMint() (gas: 645 (0.365%)) 
testItTransfersNfts() (gas: 993 (0.373%)) 
testCannotExitIfNotAdmin() (gas: 66 (0.389%)) 
testItRevertsSlippageAfterInitMint() (gas: 1819 (0.392%)) 
testItMintsLpTokensAfterInitWithEther() (gas: 1272 (0.400%)) 
testItTransfersNfts() (gas: 800 (0.441%)) 
testItRevertsSlippageOnBuy() (gas: 116 (0.485%)) 
testItTransfersNftsAfterWithdraw() (gas: 444 (0.604%)) 
testItBurnsFractionalTokens() (gas: 717 (0.618%)) 
testItAddsWithMerkleProof() (gas: 30267 (0.633%)) 
testItSellsWithMerkleProof() (gas: 32090 (0.643%)) 
testExitSetsCloseTimestamp() (gas: 272 (0.707%)) 
testCannotWithdrawIfNotClosed() (gas: 119 (0.753%)) 
testCannotWithdrawIfNotEnoughTimeElapsed() (gas: 323 (0.767%)) 
testItTransfersTokens() (gas: 949 (0.788%)) 
testItRevertsSlippageOnInitMint() (gas: 162 (0.803%)) 
testCannotWithdrawIfNotAdmin() (gas: 343 (0.813%)) 
testItTransfersNfts() (gas: -2082 (-1.221%)) 
testItTransfersBaseTokens() (gas: -2199 (-1.287%)) 
testItBurnsLpTokens() (gas: -2200 (-1.290%)) 
testItReturnsBaseTokenAmountAndFractionalTokenAmount() (gas: -2277 (-1.382%)) 
testItTransfersNfts() (gas: -2102 (-1.644%)) 
testItTransfersBaseTokens() (gas: -2286 (-1.780%)) 
testItBurnsFractionalTokens() (gas: -2261 (-1.827%)) 
testItReturnsInputAmount() (gas: -2342 (-1.923%)) 
testItMintsFractionalTokens() (gas: 3161 (1.933%)) 
testItTransfersTokens() (gas: 3542 (2.075%)) 
testItRevertsNftSlippage() (gas: -2608 (-3.957%)) 
testItRevertsBaseTokenSlippage() (gas: -2558 (-6.305%)) 
testItRevertsSlippageOnBuy() (gas: -2468 (-7.342%)) 
testItSetsSymbolsAndNames() (gas: -423593 (-10.834%)) 
testItRevertsIfDeployingSamePairTwice() (gas: -423593 (-11.010%)) 
testItSavesPair() (gas: -423582 (-11.018%)) 
testItReturnsPair() (gas: -423687 (-11.025%)) 
testItRevertsIfMaxInputAmountIsNotEqualToValue() (gas: -2777 (-12.364%)) 
testItRevertsIfValueDoesNotMatchBaseTokenAmount() (gas: -6130 (-18.348%)) 
testItRevertsIfValueIsNot0AndBaseTokenIsNot0() (gas: -6117 (-18.440%)) 
testItRevertsIfValueIsGreaterThanZeroAndBaseTokenIsNot0() (gas: -8082 (-29.316%))
```

```
Overall gas change: -1652171 (-133.538%)
```