# 6. External Components

Smart contracts can be interconnected through the inheritance of the previously developed smart contracts or the calling of functions from other contracts. Usage of insecure external components can cause undesirable or harmful effects.

## 6.1. Unknown external components should not be invoked

Invoking of external components or contract allows that contract to perform state modifying actions during the execution of our functions. If unknown components can be arbitrarily invoked, the changing of states can cause unpredictable results to the smart contract.

**Testing:**

Check that only known and trusted contracts are invoked.

Please have a look at the following example vulnerable contract.

```solidity
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";

interface IRouter {
    function swapExactTokensForTokens(
        uint amountIn,
        uint amountOutMin,
        address[] calldata path,
        address to,
        uint deadline
    ) external returns (uint[] memory amounts);
}

contract UnsafeVault {
    using SafeERC20 for IERC20;
    mapping(address => uint256) public balances;
    IERC20 public token;

    constructor(IERC20 _token) {
        token = _token;
    }

    function swapAndDeposit(IRouter router, IERC20 srcToken, uint256 amount, uint256 amountOutMin) external {
        srcToken.safeTransferFrom(msg.sender, address(this), amount);
        address[] memory path;
        path[0] = address(srcToken);
        path[1] = address(token);
        srcToken.safeIncreaseAllowance(address(router), amount);
        uint256[] memory amounts = router.swapExactTokensForTokens(amount, amountOutMin, path, address(this), block.timestamp);
        balances[msg.sender] += amounts[1];
    }

    function withdraw(uint256 amount) external {
        balances[msg.sender] -= amount;
        token.transfer(msg.sender, amount);
    }
}
```

In the contract above, the router can be set to any contract, allowing the attacker to implement a malicious contract that returns high balance without actually swapping.

**Solution:**

Perform external callings to only known and trusted smart contracts, or define a whitelist of trustable contracts.

## 6.2. Funds should not be approved or transferred to unknown accounts

Funds approved or transferred to an unknown account can be pulled from the contract anytime by the target account. Funds should only be transferred or approved to accounts within the trusted scope.

**Testing:**

1. Check that funds are approved or transferred to the account that has rights for that fund
2. Check that only the necessary amount of funds is approved for other accounts

Please have a look at the following example vulnerable contract.

```solidity
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";

interface IRouter {
    function swapExactTokensForTokens(
        uint amountIn,
        uint amountOutMin,
        address[] calldata path,
        address to,
        uint deadline
    ) external returns (uint[] memory amounts);
}

contract UnsafeVault2 {
    using SafeERC20 for IERC20;
    mapping(address => uint256) public balances;
    IERC20 public token;

    constructor(IERC20 _token) {
        token = _token;
    }

    function swapAndDeposit(IRouter router, IERC20 srcToken, uint256 amount, uint256 amountOutMin) external {
        srcToken.safeTransferFrom(msg.sender, address(this), amount);
        address[] memory path;
        path[0] = address(srcToken);
        path[1] = address(token);
        srcToken.approve(address(router), type(uint256).max);
        uint256[] memory amounts = router.swapExactTokensForTokens(amount, amountOutMin, path, address(this), block.timestamp);
        srcToken.approve(address(router), 0);
        balances[msg.sender] += amounts[1];
    }

    function withdraw(uint256 amount) external {
        balances[msg.sender] -= amount;
        token.transfer(msg.sender, amount);
    }
}
```

In the contract above, the router can be set to any contract, allowing the attacker to implement a malicious contract and pass that contract as the `router` parameter. Since the maximum value of `uint256` is approved from the token transfer, which can also be controlled by the attacker, the attacker can drain all tokens in the contract.

**Solution:**

1. Avoid approving or transferring funds to unknown accounts
2. Set the approval amount to be only the necessary amount

## 6.3. Reentrant calling should not negatively affect the contract states

Reentrancy can happen when an external contract is invoked, allowing that contract to control the execution flow and perform reentrant calls. If the states are not completely updated or resolved, reentrancy can affect the integrity of the states.

**Testing:**

1. Check that external calling is not done to unknown contracts; or
2. Check that measures are implemented to prevent reentrancy

The following `withdrawAll()` function has a reentrancy issue on the external `call()` function which is used to transfer native tokens.

```solidity
function withdrawAll() external {
    require(balance[msg.sender] >= 0,'Insufficient funds');
    payable(msg.sender).call{value: balance[msg.sender]}("");
    balance[msg.sender] = 0;
}
```

When the `call()` function is executed, if the `fallback()` function is implemented on `msg.sender` address, the fallback function will be called before the `balance[msg.sender]` is set as 0. This can be used for reentrant calling to the `withdrawAll()` function for draining all tokens in the contract.

**Solution:**

1. Use the “Checks-Effects-Interactions” pattern to completely update the states before invoking external accounts or contracts; or
2. Implement a mutex lock to prevent a reentrant calling to the same contract

For example, the source code above can be modified to comply with the “Checks-Effects-Interactions” pattern as follows:

```solidity
function withdrawAll() external {
    require(balance[msg.sender] >= 0,'Insufficient funds'); // checks
    uint256 amount = balance[msg.sender];
    balance[msg.sender] = 0; // effects
    payable(msg.sender).call{value: amount}(""); // interactions
}
```

## 6.4. Vulnerable or outdated components should not be used in the smart contract

Smart contracts can use or inherit external components that have already implemented the desired functionalities. However, the use of outdated components can be harmful, as the latest patches, bug fixes, or features are missing from the outdated version.

**Testing:**

Check that all external components used are the latest stable versions.

**Solution:**

Make sure that the latest stable versions of external components are used.

## 6.5. Deprecated components that have no longer been supported should not be used in the smart contract

Smart contracts can use or inherit external components that have already implemented the desired functionalities. However, the use of unmaintained components can be harmful, as vulnerabilities for those components may be found and published without any bug fix being released, allowing attackers to use those flaws in attacking the smart contract.

**Testing:**

Check that external components are not deprecated or unmaintained.

**Solution:**

Avoid using the deprecated or unmaintained components.

## 6.6. Delegatecall should not be used on untrusted contracts

The delegatecall function can be used to execute the logic of other contracts with the states of the caller contract. If delegatecall can be used on untrusted contracts, a malicious user can implement a malicious function as the target of the delegatecall and perform any arbitrary actions on the caller contract.

**Testing:**

Check that delegatecall is only used on trusted contracts.

```solidity
contract Worker {
    function work(address worker) external {
        worker.delegatecall(bytes4(keccak256("work()")));
    }
}
```

In the example contract above, the delegatecall is used on a user supplied address, `worker`, allowing the attacker to create a contract with the `work()` function and perform arbitrary actions on the contract.

**Solution:**

Implement a whitelist of trusted contracts to be used for the delegatecall.
