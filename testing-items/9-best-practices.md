# 9. Best Practices

Smart contract can be implemented in various ways, depending on each developer’s style. However, complying with the best practices can improve the code quality of the smart contract, making it cleaner, more readable, or more efficient.

## 9.1. State and function visibility should be explicitly labeled

State variables and functions have the default visibility assigned to them. The default visibility for state variables is internal, while the default visibility for functions is public. Labeling the visibility explicitly allows easy identification of the access scope.

**Testing:**

Check that all state variables and functions have explicit visibility.

This can be done by looking at the declaration of each state and function, making sure that they have visibility stated, for example:

```solidity
contract Visibility {
    uint256 state;

    function setState(uint256 newState) {
        state = newState;
    }

    function getState() view returns (uint256) {
        return state;
    }

}
```

**Solution:**

Explicitly label the visibility of the state variables and functions.

```solidity
contract Visibility {
    uint256 private state;

    function setState(uint256 newState) external {
        state = newState;
    }

    function getState() view external returns (uint256) {
        return state;
    }
}
```

## 9.2. Token implementation should comply with the standard specification

Implementation of tokens can be freely done; however, to make them compatible with other smart contracts, standard specifications such as ERC20 for fungible tokens, or ERC721 for non-fungible tokens should be followed.

**Testing:**

Check that the token implementation fully complies with the specifications of the selected standard.

The specifications for ERC20 can be found at: [https://eips.ethereum.org/EIPS/eip-20#specification](https://eips.ethereum.org/EIPS/eip-20#specification)

The specifications for ERC721 can be found at: [https://eips.ethereum.org/EIPS/eip-721#specification](https://eips.ethereum.org/EIPS/eip-721#specification)

For ERC20, the following slither script by tinchoabbate can be used: [https://github.com/tinchoabbate/slither-scripts/tree/master/erc20](https://github.com/tinchoabbate/slither-scripts/tree/master/erc20)

**Solution:**

Modify the smart contract to comply with the standard specifications.

As an example, the ERC20 or ERC721 contracts can be implemented by inheriting [OpenZeppelin's implementation](https://github.com/OpenZeppelin/openzeppelin-contracts/tree/master/contracts/token).
## 9.3. Floating pragma version should not be used

Smart contract compiler version can be specified using a floating pragma version; however, that may allow the contract to be compiled with other compiler versions than the one intended by the authors.

**Testing:**

Check that the compiler version is locked to one specific version.

This can be done by looking at the pragma solidity flag at the top of the contract source code file. Only one specific version should be used, not multiple, for example:

```solidity
// SPDX-License-Identifier: MIT

pragma solidity >=0.4.0 < 0.6.0;
pragma solidity >=0.4.0<0.6.0;
pragma solidity >=0.4.14 <0.6.0;
pragma solidity >0.4.13 <0.6.0;
pragma solidity 0.4.24 - 0.5.2;
pragma solidity >=0.4.24 <=0.5.3 ~0.4.20;
pragma solidity <0.4.26;
pragma solidity ~0.4.20;
pragma solidity ^0.4.14;
pragma solidity 0.4.*;
pragma solidity 0.*;
pragma solidity *;
pragma solidity 0.4;
pragma solidity 0;
```

**Solution:**

Change the compiler version flag to one fixed version.

For example:

```solidity
// SPDX-License-Identifier: MIT

pragma solidity 0.8.13;
// or
pragma solidity =0.8.13;
```

## 9.4. Builtin symbols should not be shadowed

In Solidity, multiple variables and functions are builtin, providing information about the blockchain general-use utility functions. However, those variables and functions can be overwritten by states, variables, functions, modifiers, or events with the same name, which is also known as shadowing. The shadowing of the builtin symbols can cause unintended consequences when they are used.

The list of the builtin variables and functions can be found here: [https://docs.soliditylang.org/en/latest/units-and-global-variables.html#special-variables-and-functions](https://docs.soliditylang.org/en/latest/units-and-global-variables.html#special-variables-and-functions)

**Testing:**

Check that there is no shadowing of the builtin symbols in the smart contract.

```solidity
contract FakeBuiltin {
    uint256 public now;

    function blockhash(uint blocknumber) public returns (bytes32) {
        return keccak256(abi.encodePacked(blocknumber));
    }
}

contract Shadow is FakeBuiltin {
    function fakeNextPeriod() public view returns (uint256) {
        return now + 86400;
    }

    function fakeLastBlockhash() public returns (bytes32) {
        return blockhash(block.number - 1);
    }
}
```

**Solution:**

Rename the symbol to avoid shadowing.

## 9.5. Functions that are never called internally should not have public visibility

Public functions with reference type parameters copy calldata to memory when being executed, while external functions can read directly from calldata. Memory allocation uses more resources (gas) than reading directly from calldata. Therefore, public functions that are never called internally by the contract itself should have external visibility. Furthermore, this improves the readability of the contract, allowing clear distinction between functions that are externally used and functions that are also called internally.

Please note that the increase of gas usage is no longer applicable to smart contracts compiled with Solidity version 0.6.9 and later, as the data location keywords are required to be chosen and specified explicitly for reference type parameters regardless of the function visibility.

**Testing:**

Check for public functions that are never called internally from the contract itself.

```solidity
contract PublicExternal {
    function multiply(uint256[2] inputs) public returns (uint256) {
         return inputs[0] * inputs[1];
    }
}
```

**Solution:**

Set the visibility of the public functions that are never called internally by the contract to external.

## 9.6. Assert statement should not be used for validating common conditions

The `assert()` statement is an overly assertive checking that drains all gas in the transaction when triggered. A properly functioning smart contract should **never** reach a failing assert statement. Instead, the `require()` statement should be used to validate that the conditions are met, or to validate return values from external contract callings.

**Testing:**

Check for the use of `assert()` statement, and make sure that it is a check for invariant, not a common condition validation.

Take a look at the following example function:

```solidity
function withdraw(uint256 amount) external {
    assert(balances[msg.sender] >= amount);
    balances[msg.sender] -= amount;
    (bool success, ) = msg.sender.call{value: amount}("");
    assert(success);
}
```

The function validates that the caller has enough balance and the transfer is successful, which are the conditions required for the user to withdraw. However, since `assert()` is improperly used here, anyone calling the function without sufficient balance will have their gas used to the limit of the transaction instead of having their leftover gas returned.

**Solution:**

Replace the `assert()` statement with `require()` statement if the condition checked is not an invariant or a condition that should be impossible to be reached.

For example:

```solidity
function withdraw(uint256 amount) external {
    require(balances[msg.sender] >= amount);
    balances[msg.sender] -= amount;
    (bool success, ) = msg.sender.call{value: amount}("");
    require(success);
}
```
