# Introduction

A security review of the **OpenPeer** protocol was done by **ahmedov**.

# About **OpenPeer**

OpenPeer is a completely decentralized P2P on/off ramp for emerging markets. The project allow users to create escrow contracts and in case of dispute a DAO will step in to arbitrate it.

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

**_review commit hash_ - [942befc83493735f72a6da3b016926607cc13f1e](https://github.com/Minke-Labs/openpeer_contracts/tree/942befc83493735f72a6da3b016926607cc13f1e)**

### Scope

The following smart contracts were in scope of the audit:

- `OpenPeerEscrow`
- `OpenPeerEscrowsDeployer`

The following number of issues were found, categorized by their severity:

- Critical & High: 0 issues
- Medium: 2 issues
- Low: 4 issues
- Informational: 3 issues

---

# Findings Summary

| ID     | Title                                                  | Severity      |
| ------ | ------------------------------------------------------ | ------------- |
| [M-01] | `deployNativeEscrow()` can be front-runned             | Medium        |
| [M-02] | `deployERC20Escrow()` can be front-runned              | Medium        |
| [L-01] | Sender can create escrow bypassing fees                | Low           |
| [L-02] | Possible DoS if escrow is created without feeRecipient | Low           |
| [L-03] | Owner can set very high fees                           | Low           |
| [L-04] | Possible lock of funds if arbitrator address is zero  | Low           |
| [I-01] | Default values doesn't need to be initialized          | Informational |
| [I-02] | Lack of zero address check                             | Informational |
| [I-03] | Missing events for important changes                   | Informational |

# Detailed Findings

# [M-01] `deployNativeEscrow()` can be front-runned

## Severity

**Impact:** High

**Likelihood:** Medium

## Description

`deployNativeEscrow` receives `bytes32 _orderId` as parameter, later in the `deploy` function it requires that this `_orderId` is never used.

```solidity
84: require(!escrows[_orderID].exists, "Order already exists");
```

A malicious user (front-runner) can front run the `seller` and call this function with the same `_orderId` sending a tiny amount of 1 wei MATIC which is very cheap.
This will end up reverting the original `seller`s transaction because the `_orderId` is used now.

## Recommendations

Recommended way is to not rely entirly on user provided paramaters for such checks.

# [M-02] `deployERC20Escrow()` can be front-runned

## Severity

**Impact:** High

**Likelihood:** Medium

## Description

`deployERC20Escrow()` calls the same function `deploy` so this leads to the same vulnerability discussed in `[M-01]`.

## Recommendations

Recommended way is to not rely entirly on user provided paramaters for such checks.

# [L-01] Sender can create escrow bypassing fees

## Severity

**Impact:** Low

**Likelihood:** Medium

## Description

The sellers pay 0,3% of the amount. Ex. If the buyer wants 10k USDT the seller will need to escrow 10k USDT + 0.3% as fee. Currently there is no minimum requirment for the `_amount` passed by the seller. If seller opens escrow with low `_amount` which `(_amount * fee)` is < `10_000`, this will end up as 0 because of roundings in solidity.

## Recommendations

Recommended to add a `require` to check this calculation:

```solidity
require((_amount * fee) >= 10_000);
uint256 amount = (_amount * fee / 10_000) + _amount;
```

# [L-02] Possible DoS if escrow is created without feeRecipient

## Severity

**Impact:** Low

**Likelihood:** Medium

## Description

There is a potential Denial-Of-Service problem if an escrow contract is deployed with `feeRecipient` = `address(0)`.

```solidity
        if (_fee > 0) {
            // transfers the fee to the fee recipient
            withdraw(feeRecipient, _fee);
        }

        ...

        function withdraw(address payable _to, uint256 _amount) private  {
            if (token == address(0)) {
                (bool sent,) = _to.call{value: _amount}("");
                require(sent, "Failed to send MATIC");
            } else {
                require(IERC20(token).transfer(_to, _amount), "Failed to send tokens");
            }
        }
```

At the code above if `feeRecipient` is `address(0)` and the MATIC is escrowed, will try to make a _low-level_ call to `address(0)` and fail.

## Recommendations

Make sure that `OpenPeerEscrow::initialize()` function validates that `feeRecipient` != `address(0)`

```diff
         require(_buyer != _seller, "Seller and buyer must be different");
         require(_seller != address(0), "Invalid seller");
         require(_buyer != address(0), "Invalid buyer");
+        require(_feeRecipient != address(0));
```

# [L-03] Owner can set very high fees

## Severity

**Impact:** Low

**Likelihood:** Low

## Description

Currently there is no upper limit for `fee` variable in `OpenPeerEscrowsDeployer.sol`. Тhis may cause loss of trust at the protocol because the owner can set very high fees like 90% or even 100%.

## Recommendations

There should be checks that only allow fees up to a specific value, e.g. 30%.

```diff
   function setFee(uint256 _fee) public onlyOwner {
-        fee = _fee;
-    }
+        require(_fee <= 30);
+        fee = _fee;
+    }
```

# [L-04] Possible lock of funds if arbitrator address is zero

## Severity

**Impact:** Low

**Likelihood:** Medium

## Description

Currently there is no check if `arbitrator` address is `address(0)` in initialize function of `OpenPeerEscrow`. This can lead to lock up the funds in that smart contract if there is no consensus between buyer and seller. When arbitrator is 0 there is no one to resolve the disput.

## Recommendations

Make sure that `OpenPeerEscrow::initialize()` function validates that `arbitrator` != `address(0)`

```diff
         require(_buyer != _seller, "Seller and buyer must be different");
         require(_seller != address(0), "Invalid seller");
         require(_buyer != address(0), "Invalid buyer");
+        require(_arbitrator != address(0));
```

# [I-01] Default values doesn't need to be initialized

Initializing default value for example: `uint256 zero = 0` consumes extra gas and should be avoided.

_contracts/OpenPeerEscrowsDeployer.sol:L24_

```diff
-    bool public stopped = false;
+    bool public stopped;
```

# [I-02] Lack of zero address check

Following setters in `OpenPeerEscrowsDeployer.sol` doesn't check if the passed address is `address(0)`:

- `setArbitrator()`
- `setFeeRecipient()`
- `setTrustedForwarder()`
- `setImplementation()`

Add `require` statement to verify these addresses are not 0, otherwise its posible that the deployed escrow contract to not behave correctly if one of these addresses is 0.

# [I-03] Missing events for important changes

It is recommended to emit `events` when critical variables are changed. Recommended to emit events in the following functions:

- `setArbitrator()`
- `setFeeRecipient()`
- `setFee()`
- `setSellerWaitingTime()`
- `setTrustedForwarder()`
- `setImplementation()`
- `toggleContractActive()`
