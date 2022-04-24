# 4. Business Logic

Business logic flow in general should be sequential, processed in order, and cannot be bypassed. Business logic vulnerabilities can happen when the smart contract's legitimate processing flow can be used in a way that has an adverse effect on the users or the smart contract's owner.

## 4.1. The business logic implementation should correspond to the business design

Smart contracts are implemented for specific purposes based on the business design. Even when a smart contract has no technical flaw, if the implementation does not comply with the design, unintended results can occur.

**Testing:**

Check that the smart contract functions as intended in the business design.

**Solution:**

Modify the smart contract to comply with the intended business design.

## 4.2. Measures should be implemented to prevent undesired effects from the ordering of transactions

Pending transactions can be viewed in the transaction pool, which allows others to submit a new transaction with a higher gas price to precede the transactions in the pool (front-running). Miners also have the ability to order the transaction as they want. If the smart contract is designed to be dependent on the correct ordering of the transaction, front-runners or miners may cause impact to the users of the smart contract.

**Testing:**

Check that the functions are not transaction ordering dependent, or measures are implemented to prevent the risk of transaction ordering.

The following example contract clearly demonstates the impact of transaction ordering.

```solidity
contract FindThisHash {
    bytes32 constant public hash =
      0x564ccaf7594d66b1eaaea24fe01f0585bf52ee70852af4eac0cc4b04711cd0e2;

    constructor() payable {}

    function solve(string memory solution) public {
        require(hash == keccak256(abi.encodePacked(solution)), "Incorrect answer");

        (bool sent, ) = msg.sender.call{value: 10 ether}("");
        require(sent, "Failed to send Ether");
    }
}
```

The attacker can monitor the transaction pool and submit a new transaction with the same solution, but with a higher gas price, to get the transaction to be mined earlier, allowing the attacker to gain the reward instead of the original solver.

**Solution:**

1. Remove the benefit of front-running from the smart contract; or
2. Use commitment schemes; or
3. Specifying a range of acceptable results from the transaction to prevent undesirable results

## 4.3. msg.value should not be used in loop iteration

When calling another function in the same contract that relies on `msg.value` due to the logic criteria, the `msg.value` can be accredited multiple times. Therefore, it is needed to verify the loop iteration on `msg.value` is correct according to the business design or not.

**Testing:**

Check that the `msg.value` calling in loop iteration is used correctly according to the business design.

```solidity
// SPDX-License-Identifier: GPL-3.0

pragma solidity ^0.8.0;

contract Bank {

    mapping(address => uint256) public balances;

    constructor() {
    }

    function deposit(address _receiver) public payable {
        balances[_receiver] += msg.value;
    }

    function depositFor(address[] memory _receivers) external payable {
        for (uint256 i=0; i < _receivers.length; i++) {
            deposit(_receivers[i]);
        }
    }
}
```

**Solution:**

If loop iteration is needed, tracking `msg.value` through a local variable and decreasing its amount on every iteration or usage can be done.

```solidity
// SPDX-License-Identifier: GPL-3.0

pragma solidity ^0.8.0;

contract Bank {

    mapping(address => uint256) public balances;

    constructor() {
    }

    function deposit(address _receiver) public payable {
        balances[_receiver] += msg.value;
    }

    function depositFor(address[] memory _receivers, uint256[] memory _amounts) external payable {
        require(_receivers.length == _amounts.length, "does not provide sufficient amount for all receivers");
        uint256 spend;
        for (uint256 i=0; i < _receivers.length; i++) {
            spend += _amounts[i];
            balances[_receivers[i]] += _amounts[i];
        }
        require(spend == msg.value, "incorrect provided _amounts");
    }
}
```
