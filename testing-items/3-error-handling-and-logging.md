# 3. Error Handling and Logging

Error handling and logging are the keys in making errors in smart contracts traceable, directing the execution flow to the proper path depending on the execution result, allowing the users to know where and how the contract fails, and making it possible to trance the past actions done on the smart contract.

## 3.1. Function return values should be checked to handle different results

Failure in function callings can be implemented in various ways, such as reverting the transaction on failure, or returning false on failure. Improper handling of the return value allows failure execution to pass through and cause unexpected results.

**Testing:**

Check that the return values from all function callings are handled.

Take a look at the following example contract:

```solidity
interface IToken {
    function transferFrom(address sender, address receiver, uint256 amount) external returns (bool);
}

contract Bank {

    mapping(address => mapping(address => uint256)) public balances;

    constructor() {}

    function deposit(address _token, uint256 _amount) public payable {
        balances[_token][msg.sender] += _amount;
        IToken(_token).transferFrom(msg.sender, address(this), _amount);
    }
}
```

It is possible for the token to be implemented by returning false on transfer failure instead of reverting, causing the user’s balance to increase without actually transferring the funds in.

**Solution:**

Implement a conditional checking to handle the function that returns value.

```solidity
// SPDX-License-Identifier: GPL-3.0

pragma solidity ^0.8.0;

interface IToken {
    function transferFrom(address sender, address receiver, uint256 amount) external returns (bool);
}
contract Bank {

    mapping(address => mapping(address => uint256)) public balances;

    constructor() {}

    function deposit(address _token, uint256 _amount) public payable {
        balances[_token][msg.sender] += _amount;
        bool success = IToken(_token).transferFrom(msg.sender, address(this), _amount);
        require(success, "transferFrom failed");
    }
}
```

## 3.2. Privileged functions or modifications of critical states should be logged

Event logs can be used by the users to easily monitor the actions or changes happening in the smart contract. Execution of critical state modification functions should be logged to allow the community to monitor and take proper measures for each action being performed on the smart contract.

**Testing:**

Check that events are emitted for the execution of all privileged functions.

```solidity
function updateMultiplier(uint256 multiplierNumber) public onlyOwner {
    BONUS_MULTIPLIER = multiplierNumber;
}
```

**Solution:**

Emit events in the privileged functions.

```solidity
event UpdateMultiplier(uint256 newMultiplierNumber);
function updateMultiplier(uint256 multiplierNumber) public onlyOwner {
    BONUS_MULTIPLIER = multiplierNumber;
    emit UpdateMultiplier(multiplierNumber);
}
```

## 3.3. Modifier should not skip function execution without reverting

Logics and conditions can be implemented in function modifiers. It is possible for the modifier to skip the execution of the function, causing the default value to be returned. This can cause unexpected value to be returned and used.

**Testing:**

Check that the modifier cannot skip the execution of the function without reverting.

Take a look at the following example contract:

```solidity
contract Modifier {
    bool public maintenance = true;
    uint256 public price = 10;

    modifier notMaintenance() {
        if(!maintenance) {
            _;
        }
    }
    function getPrice() public view notMaintenance returns(uint256) {
        return price;
    }
}
```

If the `maintenance` state is true, the logic of the `getPrice()` function will not be executed. This causes the `getPrice()` to return 0, which is the default value.

**Solution:**

Revert the transaction in the modifier if the required condition is not fulfilled.

```solidity
modifier notMaintenance() {
    if(!maintenance) {
      _;
    } else {
      revert("Contract is under maintenance");
    }
}
```
