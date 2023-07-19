# 8. Testing Loop Operation

Loop iteration is one of the important structures in programming languages. The loop can execute the same instructions multiple times, depending on the number of loops. If the number of loops depends on dynamically sized data, the number of executed instructions in the loop can linearly vary proportionally.

In Solidity, there are three types of loop that can be used:

* While Loop:
  * The condition is checked before executing the code inside the loop.
  * If the condition is false initially, the code inside the loop will never execute.
* Do...While Loop:
  * The code inside the loop is executed first, and then the condition is checked.
  * It guarantees that the loop will execute at least once, regardless of the condition.
* For Loop:
  * It is commonly used for iterating over dynamic-sized data.
  * It consists of an initialization step, a condition check, and an iteration step.

## 8.1. Block gas limit

A transaction requires a gas for every instruction in the execution. There is a limit to the cost of gas within one block. A transaction whose gas cost exceeds the block gas limit or the initial supply of gas will be reverted. Some functions have a deterministic gas cost, and some don't. The functions that contain loops can have more to-be-executed instructions depending on the data that the loop is interfacing with. If the data that the loop depends on keeps growing over time, it is possible that the gas cost will exceed the block gas limit, which will result in a transaction being reverted.

**Testing**

**8.1.1. Gas cost could exceed the block limit from loop operations**

In the given code snippet, the `calculateInterests()` function iterates over an array of `users`. As the size of the `users` array increases over time, there is a potential risk of encountering a denial of service (DoS) situation due to the block gas limit.

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

As a result, avoid looping through the whole data structure with an unbounded size; or, if looping over the entire structure is required, separate the looping into multiple transactions over multiple blocks.

## **8.2. Reusing msg.value**

In Solidity, there is a special read-only state called `msg.value`. It is the value of the native token that has been attached to the called payable function and will be the same in the same calling depth context. Since the `msg.value` will be the same, directly using it repeatedly in a loop would rarely be a valid case.

**Testing**

**8.2.1. Improper using `msg.value` in a loop**

In the contract below, the `bulkBuy()` function allows users to buy a specified amount of NFTs in a single transaction. This function is designed to enable any user to purchase an NFT for 1 `ether`.

```solidity
// SPDX-License-Identifier: GPL-3.0

pragma solidity ^0.8.0;

contract NFTshop {
    function buy(address _to) external payable {
        buy1NFT(_to);  
    }
    
    function bulkBuy(address[] memory addr) external payable {
        for (uint i = 0; i < addr.length; i++) {
            buy1NFT(addr[i]);
        }
    }

    function buy1NFT(address _to) internal {
        require(msg.value == 1 ether, "Need 1 ether");
        /* transfer 1 NFT operation */
    }
}
```

Unfortunately, the `msg.value` value is fixed during the whole transaction. If we call the `bulkBuy()` function with multiple `addr`, it would only cost us 1 Ether to purchase multiple NFTs. Therefore, it should be sure that `msg.value` in a loop is used correctly according to business design.

## **8.3. Unexpected revert inside loop**

When a loop involves making multiple external calls, a single failed call can cause the entire transaction to be reverted.

Check that no account can perform a denial of service on the contract by reversing the transaction. This can be done by looking at each external call to other wallets or contracts. If the target address can revert the transaction, it is vulnerable.

**Testing**

**8.3.1. Using multiple external calls in a loop**

Looking at each external call to other addresses If the single address causes the execution to fail, the entire transaction can be reverted.

```solidity
address[] public winnerList;
function sendReward() external {
      require(!sent, "Reward sent");
      sent = true;
      for (uint i = 0; i < winnerList.length; i++) {
          (bool success, ) = winnerList[i].call{value: REWARD}("");
          require(success, "Transfer failed");
      }
  }
```

In the example above, the `sendReward()` function iterates over an array of `winnerList`. If one of the winners is a smart contract with a fallback function that reverts all transactions, the execution of this function will never be successful. It is recommended to use the “Pull over Push” pattern ([https://fravoll.github.io/solidity-patterns/pull\_over\_push.html](https://fravoll.github.io/solidity-patterns/pull\_over\_push.html)) to handle this kind of situation. For example, by changing the payment design to allow users to withdraw funds instead of sending funds to other accounts.

## **8.4. Using flow control expressions over loop execution**

Flow control expressions such as `continue` and `break` provide control over loop execution. `continue` skips the current iteration, while `break` terminates the loop. They can be useful for skipping some unneccessary operation to save the gas cost. Moreover, the `return` instruction can also be used to terminate the loop prematurely.

However, using `continue`, `break`, and `return` incorrectly can potentially lead to business logic errors or unintended behavior in the code.

**Testing**

**8.4.1. Control flow operator skips a crucial part of code**

The code snippet below shows how the `return` instruction can break the logic flow of the business design.

```solidity
function registerToken(address[] memory tokens) external {
    for(uint256 i; i<tokens.length ; i++) {
        if(tokens[i] == address(0)) {
            return; // the return terminate the whole function, not just the loop 
        }
        registeredToken.push(tokens[i]);
    }
}

```

## **8.5. Inconsistent loop iterator**

The condition expression of a loop determines how many times the loop will be executed. For a for loop, there is a special expression block called `iteration step`, which intends to change the iterator with every loop execution. Modifying the iterator during the loop execution could result in unexpected behaviors in the contract.

**Testing**

**8.5.1. Having multiple expression that alter the same iterator of the loop**

For the for loop expression, it has a special place for updating the iterator. Having multiple places that change the iterator could lead to unintended behavior in the contract. The test can be done by identifying the iterators of loops and inspecting the loop to see if there are any expressions that could change the iterator value.

**8.5.2. Variable loop boundary**

The number of executions of a loop should be a constant value. Depending on the number of executions on dynamic data, it can cause an unexpected outcome if the data has been altered inside the loop.

```solidity
contract Buggy {
    uint[] public myNumber;

    function popAllElement() public {
        for (uint i; i < myNumber.length - 1; i++) {
            myNumber.pop();
        }
    }
}
```

The purpose of the `popAllElement()` function is to remove all elements from the `myNumber` array using a for loop.

For example:

```plaintext
Before: myNumber = [10, 20, 30, 40]

After:  myNumber = []
```

With the implementation of a for loop, each element within the `myNumber` array is intended to be iterated over and subsequently removed using the `pop()` function.

However, the loop condition is based on the length of the `myNumber` array minus `1`. This means that in each iteration, the maximum number of iterations decreases by `1`. As a result, the loop will not iterate over all the elements in the array, but instead stop before reaching the last element.

Let's consider an example array before executing the function: myNumber = \[10, 20, 30, 40, 50].

1. The loop initializes the loop counter `i` to `0`.
2. The loop condition checks if `i` is less than the length of the `myNumber` array minus `1`. Since the initial length of `myNumber` is `5`, the condition `i < myNumber.length - 1` is equivalent to `i < 4`. Therefore, the loop will execute as long as `i` is less than `4`.
3. Inside the loop, the `myNumber.pop()` function is called, removing the last element of the `myNumber` array in each iteration:

* In the first iteration, the `myNumber.pop()` function removes the element at index 4, which is `50`. The updated `myNumber` array becomes `[10, 20, 30, 40]`.
* In the second iteration, the `myNumber.pop()` function removes the element at index 3, which is `40`. The updated `myNumber` array becomes `[10, 20, 30]`.
* The loop did not iterate over the element at index 2 because the loop condition `i < myNumber.length - 1` evaluates to false when `i` is equal to `2`. Thus, the loop is complete.

4. After the loop completes, the `myNumber` array will contain the remaining elements that were not removed.&#x20;

Therefore, after executing the `popAllElement()` function, the resulting `myNumber` array will be \[10, 20, 30].

## Checklist

* Check that the gas cost could exceed the block limit from loop operations
* Check that there is no action which requires looping over the entire unbounded data structure.
* Check that no account can perform denial of service on the contract by reverting the transaction.
* Check that the `msg.value` calling in loop iteration is used correctly according to the business design.
* Check that the `break`, `continue`, and `return` in loop iteration is used correctly according to the business design.
