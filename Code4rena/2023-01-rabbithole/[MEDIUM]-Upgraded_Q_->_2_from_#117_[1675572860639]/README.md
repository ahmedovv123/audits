# Original link
https://github.com/code-423n4/2023-01-rabbithole-findings/issues/685
Judge has assessed an item in Issue #117 as 2 risk. The relevant finding follows:

 Description
If a single address has certain amount of RabbitHoleReceipt tokens (receipts) - according to tests ~1050, when he tries to call claim function from Quest.sol it will always revert with 'Transaction ran out of gas' error. The reason for the error is that claim function loops over user's all tokens.

uint[] memory tokens = rabbitHoleReceiptContract.getOwnedTokenIdsOfQuest(questId, msg.sender);
...
for (uint i = 0; i < tokens.length; i++) {
  if (!isClaimed(tokens[i])) {
    redeemableTokenCount++;
  }
}
PoC
Add a new test case in test/Erc20Quest.spec.ts file.

it('user can DoS himself (claim function)', async () => {
    const signers = await ethers.getSigners();
    const seller = signers[9]
    const buyer = signers[10];

    await deployedQuestContract.start()
    await ethers.provider.send('evm_increaseTime', [86400])

    // Ignore below function. It represents 1050 tokens from different users.
    for (let i = 0; i < 1050; ++i) {
      await deployedRabbitholeReceiptContract.connect(seller).mint(seller.address, questId)
    }

    const totalTokens = await deployedRabbitholeReceiptContract.getOwnedTokenIdsOfQuest(questId, seller.address);

    // Assume buyer gets 1050 tokens from different users, not one
    for (let i = 0; i < totalTokens.length; ++i) {
      await deployedRabbitholeReceiptContract.connect(seller).transferFrom(seller.address, buyer.address, totalTokens[i]);
    }

    // Now buyer has 1050 tokens
    // He calls claim to get the rewards, but function reverts with `Transaction ran out of gas` error everytime
    await deployedQuestContract.connect(buyer).claim()
})
Mitigation Steps
The user can always generate a new address and transfer some of the funds there. But if he has many tokens (> 10000), he should manage dozen accounts which can be become difficult and time consuming.
Better approach is to add maximum token claim amount as function parameter or to check tokens array length to not exceed certain amount.