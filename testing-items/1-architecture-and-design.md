# 1. Architecture and Design

Implementing smart contracts to be secure requires proper architecture and design. This testing category involves the use of compilers, the design of the smart contract calling architecture, and the design of roles and permissions.

## 1.1. Proper measures should be used to control the modifications of smart contract logic

Smart contracts on multiple blockchains can be upgraded, or designed to be upgradable. Modifications of contract logic can help in resolving newly found issues. Nevertheless, upgradability without any restrictions opens up room for the holders of privileged accounts to insert malicious logic into the smart contract. Furthermore, the upgrade pattern increases the complexity of the code and may introduce new bugs to the smart contract.

**Testing:**

1. Check that the smart contract is immutable and cannot be upgraded; or
2. If the upgradability is needed, check that measures are implemented to control the upgrade process of the smart contract

**Solution:**

1. Make the smart contract immutable; or
2. Transfer the upgrade permission to community-run smart contract governance or DAO; or
3. Mitigate the risk by using a timelock to delay the upgrade for a reasonable amount of time, e.g. at least 24 hours

## 1.2. The latest stable compiler version should be used

Smart contracts are generally written using high-level languages, and compiled into bytecodes that can be interpreted and executed by virtual machines that stores data on-chain. As the compilers are regularly updated with bug fixes and new features, the latest stable compiler version should be used to compile the smart contracts.

**Testing:**

Check that the pragma directive is explicitly set to the single latest stable version of the compiler.

The Solidity compiler version used by the contract can be checked through the “pragma solidity” flag

```solidity
// SPDX-License-Identifier: MIT

pragma solidity 0.8.4;
```

The latest Solidity compiler version can be referenced from here: [https://github.com/ethereum/solidity/releases](https://github.com/ethereum/solidity/releases)

**Solution:**

Set the pragma directive to the single latest stable version of the compiler

```solidity
// SPDX-License-Identifier: MIT

pragma solidity 0.8.13;
```

## 1.3. The circuit breaker mechanism should not prevent users from withdrawing their funds

Kill-switch mechanism or circuit breaker can be used to stop or pause the operation of the smart contract when new bugs are found, such as disabling the deposit of funds to prevent the damages from being escalated. However, if users are normally allowed to withdraw their funds at any time, the mechanism should not stop the users from withdrawing their funds.

**Testing:**

Check that the circuit breaker mechanism does not prevent the users from withdrawing their funds.

The following example is the `withdraw()` function has the circuit breaker mechanism by using the `whenNotPaused` modifier from "@openzeppelin/contracts/utils/Pausable.sol”

```solidity
function withdraw(uint256 _amount) public whenNotPaused {
    UserInfo storage user = userInfo[msg.sender];
    require(user.amount >= _amount, "insufficient funds");
    payReward();
    if(_amount > 0) {
        user.amount = user.amount.sub(_amount);
        baseToken.safeTransfer(address(msg.sender), _amount);
    }
    emit Withdraw(msg.sender, _amount);
}
```

**Solution:**

Provide a method for the users to safely withdraw their funds when the contract is stopped or paused.

```solidity
function emergencyWithdraw() public {
    UserInfo storage user = userInfo[msg.sender];
    uint256 userAmount = user.amount;
    user.amount = 0;
    baseToken.safeTransfer(address(msg.sender), userAmount);
    emit EmergencyWithdraw(msg.sender, userAmount);
}
```

## 1.4. The smart contract source code should be publicly available

The source code of the smart contract explicitly describes how each smart contract works. Having it publicly available for the users to view allows the users to understand and confirm that the contract is working exactly as they understood it. Furthermore, this allows the users to compare the deployed smart contract with the source code audited by professional smart contract auditors (if any). Without the source code published, it is possible for hidden functionalities or risks to be included in the smart contract.

**Testing:**

1. Check that the smart contracts have verified source code on the block explorer; or
2. Check that the smart contracts source code repository is public

**Solution:**

1. Verify the smart contract source code on the block explorer; or
2. Publish the smart contract source code repository

## 1.5. State variables should not be unfairly controlled by privileged accounts

Only the states that require modifications by the privileged parties should be modifiable by the admin accounts. The admin accounts should have the least privilege possible to manage the operation of the smart contracts. With overly permissive rights, the privileged accounts can perform actions that are unfair to the smart contract users to gain benefits.

**Testing:**

Check that the critical states are not modifiable by the admin accounts, unless necessary without providing an unfair advantage to the privileged parties.

**Solution:**

1. Remove the functions with unnecessarily high privilege; or
2. Transfer the privilege to community-run smart contract governance or DAO; or
3. Mitigate the risk by using a timelock to delay the effect of the privileged functions by a sufficient amount of time, e.g. at least 24 hours

## 1.6. Least privilege principle should be used for the rights of each role

The least privilege principle should be applied to each role. The privilege of each role should be set only to do the task in their role. For example, the minter role should be allowed to mint the token only, and should not be allowed to do other actions.

**Testing:**

Check that the privilege of each role is applied to the principle of least privilege. The privilege of each role should allow them to perform their required tasks only.

For example, the ERC-20 token was used as a reward token for the yield-farming contract. The `OWNER_ROLE` role was set to MasterChef contract for minting reward and the owner wallet to pause/unpause the token.

```solidity
contract ERC20PresetMinterPauser is Context, AccessControlEnumerable, ERC20Burnable, ERC20Pausable {
    bytes32 public constant OWNER_ROLE = keccak256("OWNER_ROLE");

    constructor(string memory name, string memory symbol) ERC20(name, symbol) {
        _setupRole(DEFAULT_ADMIN_ROLE, _msgSender());
        _setupRole(OWNER_ROLE, _msgSender());
    }

    function mint(address to, uint256 amount) public virtual {
        require(hasRole(OWNER_ROLE, _msgSender()), "You must have owner role to mint");
        _mint(to, amount);
    }

    function pause() public virtual {
        require(hasRole(OWNER_ROLE, _msgSender()), "You must have owner role to pause");
        _pause();
    }

    function unpause() public virtual {
        require(hasRole(OWNER_ROLE, _msgSender()), "You must have owner role to unpause");
        _unpause();
    }

    function _beforeTokenTransfer(
        address from,
        address to,
        uint256 amount
    ) internal virtual override(ERC20, ERC20Pausable) {
        super._beforeTokenTransfer(from, to, amount);
    }
}
```

**Solution:**

1. Ensure that each role is allowed to perform their required tasks only.
2. Separating the tasks for each role. For example, the minter role should only be allowed to mint the token, and should not have the right to do other tasks, e.g., pause token transfer.

The solution for an example is separate the privilege of `OWNER_ROLE` to `MINTER_ROLE` and `PAUSER_ROLE` which allows them to do only their task. The fixed code is shown below: (The following code is from: [OpenZeppelin's ERC20PresetMinterPauser Contract](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/presets/ERC20PresetMinterPauser.sol))

```solidity
contract ERC20PresetMinterPauser is Context, AccessControlEnumerable, ERC20Burnable, ERC20Pausable {
    bytes32 public constant MINTER_ROLE = keccak256("MINTER_ROLE");
    bytes32 public constant PAUSER_ROLE = keccak256("PAUSER_ROLE");

      constructor(string memory name, string memory symbol) ERC20(name, symbol) {
        _setupRole(DEFAULT_ADMIN_ROLE, _msgSender());

        _setupRole(MINTER_ROLE, _msgSender());
        _setupRole(PAUSER_ROLE, _msgSender());
    }

    function mint(address to, uint256 amount) public virtual {
        require(hasRole(MINTER_ROLE, _msgSender()), "ERC20PresetMinterPauser: must have minter role to mint");
        _mint(to, amount);
    }

    function pause() public virtual {
        require(hasRole(PAUSER_ROLE, _msgSender()), "ERC20PresetMinterPauser: must have pauser role to pause");
        _pause();
    }

    function unpause() public virtual {
        require(hasRole(PAUSER_ROLE, _msgSender()), "ERC20PresetMinterPauser: must have pauser role to unpause");
        _unpause();
    }

    function _beforeTokenTransfer(
        address from,
        address to,
        uint256 amount
    ) internal virtual override(ERC20, ERC20Pausable) {
        super._beforeTokenTransfer(from, to, amount);
    }
}
```
