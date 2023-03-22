# Original link
https://github.com/code-423n4/2022-12-prepo-findings/issues/38
# [G-01] X = ```X + Y``` IS MORE EFFICIENT, THAN ```X += Y``` (6 INSTANCES)

packages/prepo-shared-contracts/contracts/NFTScoreRequirement.sol: [60](https://github.com/prepo-io/prepo-monorepo/blob/feat/2022-12-prepo/packages/prepo-shared-contracts/contracts/NFTScoreRequirement.sol#L60)
apps/smart-contracts/core/contracts/DepositRecord.sol: [31](https://github.com/prepo-io/prepo-monorepo/blob/feat/2022-12-prepo/apps/smart-contracts/core/contracts/DepositRecord.sol#L31), [32](https://github.com/prepo-io/prepo-monorepo/blob/feat/2022-12-prepo/apps/smart-contracts/core/contracts/DepositRecord.sol#L32), [36](https://github.com/prepo-io/prepo-monorepo/blob/feat/2022-12-prepo/apps/smart-contracts/core/contracts/DepositRecord.sol#L36)
apps/smart-contracts/core/contracts/WithdrawHook.sol: [64](https://github.com/prepo-io/prepo-monorepo/blob/feat/2022-12-prepo/apps/smart-contracts/core/contracts/WithdrawHook.sol#L64), [71](https://github.com/prepo-io/prepo-monorepo/blob/feat/2022-12-prepo/apps/smart-contracts/core/contracts/WithdrawHook.sol#L71)

# [G-02] UNNECESSARY STORAGE READ ON EVENT EMITTING (1 INSTANCE)

apps/smart-contracts/core/contracts/DepositRecord.sol: [42](https://github.com/prepo-io/prepo-monorepo/blob/feat/2022-12-prepo/apps/smart-contracts/core/contracts/DepositRecord.sol#L42)

```diff
   function setGlobalNetDepositCap(uint256 _newGlobalNetDepositCap) external override onlyRole(SET_GLOBAL_NET_DEPOSIT_CAP_ROLE) {
     globalNetDepositCap = _newGlobalNetDepositCap;
-    emit GlobalNetDepositCapChange(globalNetDepositCap);
+    emit GlobalNetDepositCapChange(_newGlobalNetDepositCap);
   }
```