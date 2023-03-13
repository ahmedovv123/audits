# Introduction

A security review of the **TalentLayer** protocol was done by **ahmedov**.

# About **TalentLayer**

TalentLayer is infrastructure for open labor markets; backend tooling for building marketplaces that leverage an on-chain liquidity pool of users and services.
TalentLayer provides core tooling for building marketplaces including:

- On-chain jobs and proposals
- Soul-bound identity and reviews
- Configurable escrow supporting milestone, lump sum, hourly pay and more
- Decentralized and platform-managed dispute resolution options

# Severity classification

| Severity               | Impact: High | Impact: Medium | Impact: Low |
| ---------------------- | ------------ | -------------- | ----------- |
| **Likelihood: High**   | Critical     | High           | Medium      |
| **Likelihood: Medium** | High         | Medium         | Low         |
| **Likelihood: Low**    | Medium       | Low            | Low         |

**Impact** - the technical, economic and reputation damage of a successful attack

**Likelihood** - the chance that a particular vulnerability gets discovered and exploited

**Severity** - the overall criticality of the risk

# Security Assessment Summary

**_review commit hash_ - [fc2130e9d39ed0eb48ce07176fde896bf2eb10d6](https://github.com/TalentLayer/talentlayer-id-contracts/tree/fc2130e9d39ed0eb48ce07176fde896bf2eb10d6)**

### Scope

The following smart contracts were in scope of the audit:

- `Arbitrable.sol`
- `Arbitrator.sol`
- `TalentLayerArbitrator.sol`
- `TalentLayerEscrow.sol`
- `TalentLayerID.sol`
- `TalentLayerPlatformID.sol`
- `TalentLayerReview.sol`
- `TalentLayerService.sol`

The following number of issues were found, categorized by their severity:

- Critical & High: 2 issues
- Medium: 2 issues
- Low: 3 issues
- Informational: 2 issues

---

# Findings Summary

| ID     | Title                                                          | Severity      |
| ------ | -------------------------------------------------------------- | ------------- |
| [H-01] | Platform owner can control which party wins in escrow          | High          |
| [H-02] | Unsafe usage of ERC20 transfer and transferFrom                | High          |
| [M-01] | Return values of low-level calls are not checked               | Medium        |
| [M-02] | call() should be used instead of transfer()                    | Medium        |
| [L-01] | User can send more than necessary amount of fee                | Low           |
| [L-02] | Seller can lose ability to update his proposal                 | Low           |
| [L-03] | No storage gap for ERC2771RecipientUpgradeable                 | Low           |
| [I-01] | Create your own import names instead of using the regular ones | Informational |
| [I-02] | Code repetition                                                | Informational |

# Detailed Findings

# [H-01] Platform owner can control which party wins in escrow

## Severity

**Impact:** High

**Likelihood:** High

## Description

After the creation of a transaction for existing proposal by calling the function:

```solidity
contracts/TalentLayerEscrow.sol

419:  function createTransaction(...)
```

If one of the parties pays the arbitration fee, the transaction status will be changed to `WaitingSender` or `WaitingReceiver`, from now on functions `release` and `reimburse` won't work because they require status to be `NoDispute`. This means that only ways to proceed with the transaction are:

- Both parties pays the arbitration fee
- Arbitration fee timeout passes and the party who paid for arbitration wins.

Proceeding with the first one:

Let's say parties are Bob (sender) and Alice (receiver), Bob has paid the fee by calling `payArbitrationFeeBySender()` now Alice should pay also, otherwise Bob will win after the timeout period. Malicious Bob has bribed the owner of platform and now they decide to change the arbitration fee of the platform.

The platform owner calls the following function:

```solidity
60: function setArbitrationPrice(uint256 _platformId, uint256 _arbitrationPrice) public {
```

passing absurdly high price as `_arbitrationPrice`, lets say `type(uint256).max`. When Alice tries to pay the arbitration fee she now will have to send this much eth in order to proceed with the dispute, because of the way for retreiving the price:

```solidity
function payArbitrationFeeByReceiver(uint256 _transactionId) public payable {
        ...
        uint256 arbitrationCost = transaction.arbitrator.arbitrationCost(transaction.arbitratorExtraData);
        ...
}
```

Then at line 588:

```solidity
require(transaction.receiverFee == arbitrationCost, "The receiver fee must be equal to the arbitration cost");
```

Transaction will revert simply because Alice can't pay the fee, now she has nothing to do. Bob now should only wait for the fee timout period and then call the function `arbitrationFeeTimeout()` making the transaction status `Resolved` and setting the winner to be Bob (sender).

## Recommendations

One recommended way is to declare the arbitration fee when the transaction is created by adding `arbitrationfee` to `Transaction` struct and get the fee price from there. This will make sure that no one can change the fee and platform owner will not have the rights for this.

Also make sure that there is a fee limit that users can set, for example `0.5 ETH`.

# [H-02] Unsafe usage of ERC20 transfer and transferFrom

## Severity

**Impact:** High

**Likelihood:** High

## Description

Some ERC20 tokens functions don't return a boolean, for example USDT, BNB, OMG. So the `TalentLayerEscrow` contract simply won't work with tokens like that as the token.

```solidity
    function _safeTokenDeposit(address _token, uint256 _amount, address _sender) private {
        ...
        require(IERC20(_token).transferFrom(_sender, address(this), _amount), "Transfer must not fail");
       ...
    }
```

Function above requires every time when `tranfserFrom()` called to return true, but not all tokens return boolean.

The USDT's transfer and transferFrom functions doesn't return a bool, so the call to these functions will revert although the user has enough balance and the `TalentLayerEscrow` contract won't work, assuming that token is USDT.

The same goes for `transfer` function in L984:

```solidity
984: IERC20(_tokenAddress).transfer(_recipient, _amount);
```

Here it is the opposite. This line assumes that if the contract doesn't have enough balance the tx will revert, but there are some tokens that instead of reverting they return boolean. The [ZRX](https://etherscan.io/token/0xe41d2489571d322189246dafa5ebde1f4699f498#code) token is one of them.

## Recommendations

Use OpenZeppelin's [SafeERC20](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/utils/SafeERC20.sol) library when interaction with erc20 tokens.

# [M-01] Return values of low-level calls are not checked

## Severity

**Impact:** Medium

**Likelihood:** Medium

## Description

If the return value of a low-level message call is not checked then the execution will resume even if the called contract throws an exception. If the call fails accidentally or an attacker forces the call to fail, then this may cause unexpected behavior in the subsequent program logic.

Тhe lack of check occurs in various places on the contracts:

```solidity
contracts/TalentLayerEscrow.sol

620: payable(transaction.sender).call{value: senderFee}("");
627: payable(transaction.receiver).call{value: receiverFee}("");
753: payable(transaction.sender).call{value: extraFeeSender}("");
761: payable(transaction.receiver).call{value: extraFeeReceiver}("");
791: sender.call{value: senderFee}("");
794: receiver.call{value: receiverFee}("");
804: sender.call{value: splitFeeAmount}("");
805: receiver.call{value: splitFeeAmount}("");
982: _recipient.call{value: _amount}("");
```

## Recommendations

Always check the returned value and be sure that the calling address exists:

```solidity
require(address(contract) != address(0));
(bool success,) = contract.call{value: value}("");
require(success, "...");
```

# [M-02] call() should be used instead of transfer() on an address payable

## Severity

**Impact:** Medium

**Likelihood:** Medium

## Description

TalentLayerArbitrator.sol: L127.

The `transfer()` and `send()` functions forward a fixed amount of 2300 gas. Historically, it has often been recommended to use these functions for value transfers to guard against reentrancy attacks. However, the gas cost of EVM instructions may change significantly during hard forks which may break already deployed contract systems that make fixed assumptions about gas costs. For example. EIP 1884 broke several existing smart contracts due to a cost increase of the SLOAD instruction.

## Recommendations

Use `call()` instead of `transfer()`, but be sure to respect the CEI pattern and/or add re-entrancy guards, as several hacks already happened in the past due to this recommendation not being fully understood.

More info on <https://swcregistry.io/docs/SWC-134>

# [L-01] User can send more than necessary amount of fee

## Severity

**Impact:** Low

**Likelihood:** Low

## Description

Creating dispute requires user to send the needed fee. Currently the check if the fee is payed is implemented this way:

```solidity
contracts/Arbitrator.sol

22: require(msg.value >= arbitrationCost(_extraData), "Not enough ETH to cover arbitration costs.");
```

This checks if `msg.value` is greater or equal to the current set fee. If user mistakenly sends more `ETH` than necessary he will lose them.

## Recommendations

Change the `require` statement to check whether the amount sent is exactly as it should be.

```diff
-        require(msg.value >= arbitrationCost(_extraData), "Not enough ETH to cover arbitration costs.");
+        require(msg.value == arbitrationCost(_extraData), "Not enough ETH to cover arbitration costs.");
```

# [L-02] Seller can lose ability to update his proposal

**Impact:** Low

**Likelihood:** Low

## Description

After creating proposal seller can update it after some time. Updateable values are:

- rateToken
- rateAmount
- dataURI
- expirationDate

To update only one of the above, seller should pass only that value different the other values should be same.

Meanwhile if the owner of contract decides to update the `allowedTokenList` list and disables the current `rateToken` of seller, he won't be able to update propsal details `rateAmount`, `dataURI` and `expirationDate` because everytime it checks that if `rateToken` is whitelisted.

## Reocmmendations

Change the current implementation logic.

# [L-03] No storage gap for ERC2771RecipientUpgradeable

**Impact:** Medium

**Likelihood:** Low

## Description

When creating upgradable contracts that inherit from other contracts is important that there are storage gap in case storage variable are added to inherited contracts. If an inherited contract is a stateless contract (i.e. it doesn't have any storage) then it is acceptable to omit a storage gap, since these function similar to libraries and aren't intended to add any storage.

`ERC2771RecipientUpgradeable.sol` is inherited from the following contracts:

- `TalentLayerEscrow`
- `TalentLayerService`
- `TalentLayerReview`
- `TalentLayerID`

## Recommendations

Add storage gap `__gap[50]` variable to the `ERC2771RecipientUpgradeable.sol`

# [I-01] Create your own import names instead of using the regular ones

For better readability, you should name the imports instead of using the regular ones.

For example:

```solidity
import "./Arbitrable.sol";
```

should be refactored to:

```solidity
import {Arbitrable} from "./Arbitrable.sol";
```

# [I-02] Code repetition

At `TalentLayerEscrow::arbitrationFeeTimeout` decides which party wins the escrow and sends the fee paid by them on lines 620 or 627.

```solidity
    if (transaction.status == Status.WaitingSender) {
        if (transaction.senderFee != 0) {
            ...
            payable(transaction.sender).call{value: senderFee}("");
        }
        _executeRuling(_transactionId, RECEIVER_WINS);
    } else if (transaction.status == Status.WaitingReceiver) {
        if (transaction.receiverFee != 0) {
            ...
            payable(transaction.receiver).call{value: receiverFee}("");
        }
        _executeRuling(_transactionId, SENDER_WINS);
    }
```

Later on it calls `_executeRulling()` function where does the same thing:

```solidity
    ...
    transaction.senderFee = 0;
    transaction.receiverFee = 0;
    ...

    if (_ruling == SENDER_WINS) {
        sender.call{value: senderFee}("");
        ...
    } else if (_ruling == RECEIVER_WINS) {
        receiver.call{value: receiverFee}("");
        ...
    } ...
```

Consider doing this only in `_executeRuling()`
