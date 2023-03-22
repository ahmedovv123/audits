# Original link
https://github.com/code-423n4/2022-12-prepo-findings/issues/37
# [N-01] LARGE MULTIPLES OF TEN SHOULD USE SCIENTIFIC NOTATION

Use (e.g. 1e6) rather than decimal literals (e.g. 1000000), for better code readability

apps/smart-contracts/core/contracts/Collateral.sol: [19](https://github.com/prepo-io/prepo-monorepo/blob/feat/2022-12-prepo/apps/smart-contracts/core/contracts/Collateral.sol#L19)
apps/smart-contracts/core/contracts/ManagerWithdrawHook.sol: [12](https://github.com/prepo-io/prepo-monorepo/blob/feat/2022-12-prepo/apps/smart-contracts/core/contracts/ManagerWithdrawHook.sol#L12)

# [N-02] USE SCIENTIFIC NOTATION (E.G. 1E18) RATHER THAN EXPONENTIATION (E.G. 10**18)

Scientific notation should be used for better code readability

apps/smart-contracts/core/contracts/Collateral.sol: [31](https://github.com/prepo-io/prepo-monorepo/blob/feat/2022-12-prepo/apps/smart-contracts/core/contracts/Collateral.sol#L31)
apps/smart-contracts/core/contracts/TokenSender.sol: [33](https://github.com/prepo-io/prepo-monorepo/blob/feat/2022-12-prepo/apps/smart-contracts/core/contracts/TokenSender.sol#L33)

# [N-03] MISLEADING COMMENT BLOCK

On lines [L8-L9](https://github.com/prepo-io/prepo-monorepo/blob/feat/2022-12-prepo/apps/smart-contracts/core/contracts/interfaces/IPrePOMarket.sol#L8-L9) it says that ```Users can mint/redeem long/short positions on a specific asset in exchange for Collateral tokens.```.
On lines [L73-L74](https://github.com/prepo-io/prepo-monorepo/blob/feat/2022-12-prepo/apps/smart-contracts/core/contracts/interfaces/IPrePOMarket.sol#L73-L74) there is a comment about the ```mint``` which says ```Minting will only be done by the team, and thus relies on the `_mintHook` to enforce access controls```.