# Diva Token Vesting - QA Report

| Letter | Name | Description |
|:--:|:-------:|:-------:|
| L  | Low risk | Potential risk |
| NC |  Non-critical | Non risky findings |
| R  | Refactor | Changing the code |
| O | Ordinary | Often found issues |

| Total Found Issues | 6 |
|:--:|:--:|

### Low Risk Issues Template
| Count | Explanation | Instances |
|:--:|:-------|:--:|
| [L-01] | Use most recent OpenZeppelin contracts | 1 |
| [L-02] | ```transferOwnership``` should be two step process | 1 |
| [L-03] | Potential locking of ETH if sent accidentally | 1 |

| Total Lor Risk Issues | 3 |
|:--:|:--:|

### Non-Critical Issues Template
| Count | Explanation | Instances |
|:--:|:-------|:--:|
| [N-01] | Create your own import names instead of using the regular ones | 6 |

| Total Non-Critical Issues | 1 |
|:--:|:--:|

### Refactor Issues Template
| Count | Explanation | Instances |
|:--:|:-------|:--:|
| [R-01] | Misleading modifier name | 1 |

| Total Refactor Issues | 1 |
|:--:|:--:|

### Ordinary Issues Template
| Count | Explanation | Instances |
|:--:|:-------|:--:|
| [O-01] | Use ```constant``` for constant variables | 1 |

| Total Ordinary Issues | 1 |
|:--:|:--:|

### [L-01] Use most recent OpenZeppelin contracts

Currently the project implements an old version of OpenZeppelin contracts package which is [^4.3.2](https://github.com/divaprotocol/diva-token-contract/blob/main/package.json#L34). 

There are [known](https://security.snyk.io/package/npm/@openzeppelin%2Fcontracts/4.3.2) public vulnerabilities and should be used most common stable version.

### [L-02] ```transferOwnership``` should be two step process

The contract is inherited from OpenZeppelin’s Ownable contract which enables the onlyOwner role to transfer ownership to another address. It’s possible that the onlyOwner role mistakenly transfers ownership to the wrong address, resulting in a loss of the onlyOwner role. The current ownership transfer process involves the current owner calling ```transferOwnership()```. This function checks the new owner is not the zero address and proceeds to write the new owner’s address into the owner’s state variable. If the nominated EOA account is not a valid account, it is entirely possible the owner may accidentally transfer ownership to an uncontrolled account, breaking all functions with the onlyOwner() modifier. Lack of two-step procedure for critical operations leaves them error-prone if the address is incorrect, the new address will take on the functionality of the new role immediately.

Also ```Ownable``` contract implements ```renounceOwnership``` function which removes owner and sets it to ```address(0)```. This will leave contract without owner and make all the ```onlyOwner``` function uncallable.

Recommended library to use: [Ownable2Step](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/access/Ownable2Step.sol) from OpenZeppelin.

### [L-03] Potential locking of ETH if sent accidentally

The [constructor](https://github.com/divaprotocol/diva-token-contract/blob/main/src/ClaimDIVALinearVesting.sol#L41) is payable, this means it is accepting any send eth value. If while deploying the contract eth are sent, there is no way of getting them back, because there is no ```withdraw``` function.

### [N-01] Create your own import names instead of using the regular ones

For better readability, you should name the imports instead of using the regular ones.

Refactor to:
```solidity
import {MerkleProof} from "@openzeppelin/contracts/utils/cryptography/MerkleProof.sol";
import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";
import {Pausable} from "@openzeppelin/contracts/security/Pausable.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {SafeMath} from"@openzeppelin/contracts/utils/math/SafeMath.sol";
import {ReentrancyGuard} from"@openzeppelin/contracts/security/ReentrancyGuard.sol";
```

### [R-01] Misleading modifier name

```soliditiy
    modifier isTriggered() {
        require(!trigger, "ClaimDIVA: you can not update data after triggered");
        _;
    }
```

Judging by modifier name it should check that ```trigger``` is truish, but it requires it to be set to ```false```

Consider renaming modifier name to: ```notTriggered```.

### [O-01] Use ```constant``` for constant variables

Using ```constant``` keyword stores the variable directly to the bytecode of smart contract instead of saving to storage slot.

```solidity
25: uint256 public constant YEAR = 31556926;
```

