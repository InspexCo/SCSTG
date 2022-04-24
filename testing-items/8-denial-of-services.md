# 8. Denial of Services

Improper contract logic can affect the availability of the contract. It should be made sure that the smart contract can function properly as designed.

## 8.1. State changing functions that loop over unbounded data structures should not be used

Execution of operations on smart contracts requires gas based on the computation needed. When a function loops through a data structure that keeps growing over time, it is possible that a denial of service can happen when the amount of gas required exceeds the block gas limit.

**Testing:**

Check that there is no action which requires looping over the entire unbounded data structure.

```solidity
function calculateInterests() public {
	for (uint i = 0; i < users.length; i++) {
		interests[users[i]] += calculateUserInterest(users[i]);
		depositTimes[users[i]] = block.timestamp;
	}
}

function calculateUserInterest(address user) public view returns (uint256) {
	if (balances[user] > 0) {
		return balances[user] * (block.timestamp - depositTimes[user]) / 3600;
	}
	return 0;
}
```

**Solution:**

Avoid looping through the whole data structure with an unbounded size; or, if looping over the entire structure is required, separate the looping into multiple transactions over multiple blocks.

## 8.2. Unexpected revert should not make the whole smart contract unusable

Some smart contracts require interaction with external accounts to perform the payment. If the target account is a smart contract that reverts on any payment, the smart contract may not be able to operate successfully, resulting in a denial of service.

**Testing:**

Check that no account can perform denial of service on the contract by reverting the transaction.

This can be done by looking at each external call to other wallets or contracts. If the target address can revert the transaction, it is vulnerable.

```solidity
function sendReward() external {
	require(!sent, "Reward sent");
	sent = true;
	for (uint i = 0; i < winnerList.length; i++) {
		(bool success, ) = winnerList[i].call{value: REWARD}("");
		require(success, "Transfer failed");
	}
}
```

From the example above, if one of the winners is a smart contract with a fallback function that reverts all transactions, the execution of this function will never be successful.

**Solution:**

Use the “Pull over Push” pattern by changing the payment design to allow users to withdraw funds instead of sending funds to other accounts

## 8.3. Strict equalities should not cause the function to be unusable

When determining the value of a state controllable by external actors, such as account balance, strict equality should not be used. This is because the state can be changed, such as by directly transfer to the contract, causing the function to be unusable.

**Testing:**

Check that there is no use of strict equality for states that can be modified by external actors that could lead to denial of service.

This can be done by inspecting the function that perform comparison on an externally controllable state and ensuring that the strict equality is not used, for example:

```solidity
function goalReached() public returns(bool){
    return this.balance == 5 ether;
}
```

The `goalReached()` function checks if the fund reaches its goal before returning true to perform the further steps. An attacker can transfer 1 wei after the fund reaches 5 ether, causing the balance of the contract to be unequal to 5. The `goalReached()` function will always return false and further steps can never be proceeded.

**Solution:**

Use "greater than or equal to" or "less than or equal to" instead of strict equality for a state controllable by external actors to prevent denial of service.

For example:

```solidity
function goalReached() public returns(bool){
    return this.balance >= 5 ether;
}
```