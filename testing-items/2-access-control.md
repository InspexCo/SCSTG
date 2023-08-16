# 2. Testing Contract Compiling

The final form of smart contract is the bytecode form. It can directly be deployed into the blockchain. To get the bytecode of a smart contract, it can be done by using a compiler or directly write a contract from a scratch in a bytecode form. When compiling the source code using a compiler, the user must adhere to the specification of the respective compiler version. Since Solidity is not the only compiler for EVM.

## 2.1. Contract dependency

In a blockchain community, there are various standards proposed and accepted. The standard acts as a conformation of interfaces to let developers develop smart contracts that can communicate with other smart contracts seamlessly. For example, the ERC20 standard is a standard on the Ethereum blockchain about smart contracts that act as fungible tokens. To have the smart contract comply with the ERC20 standard, the developer must implement functions in the contract according to the standard.

To comply with any standards, the developer must implement/override the required functions accordingly. On the other hand, there are special symbols that implicitly exist, known as the built-in symbols, but they should not be overridden with functions with the same name. The built-in symbols serve as the foundation of a contract. If the symbol's behavior is changed, the whole contract may not work normally.

**Testing**

**2.1.1. Contract implementation should comply with the standards specification**

Implementation of tokens can be freely done; however, to make them compatible with other smart contracts, standard specifications such as ERC20 for fungible tokens, or ERC721 for non-fungible tokens should be followed:

The specifications for ERC20 can be found at: [https://eips.ethereum.org/EIPS/eip-20#specification ](https://eips.ethereum.org/EIPS/eip-20#specification)The specifications for ERC721 can be found at: [https://eips.ethereum.org/EIPS/eip-721#specification](https://eips.ethereum.org/EIPS/eip-721#specification)

```solidity
// contracts/GLDToken.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract Token is ERC20 {
    constructor(uint256 initialSupply) ERC20("Token", "TKN") {
        _mint(msg.sender, initialSupply);
    }
}
```

**2.1.2. Built-in symbols should not be shadowed**

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

## 2.2. Solidity

For a smart contract that is written in Solidity, there are some points that could be enforced to improve the contract’s clarity. The Solidity complier has many versions. Each major version has changes that could break the old source code that was compiled by the older version of the compiler.

**Testing**

**2.2.1. Solidity compiler version should be specific**

Check that the compiler version is locked to one specific version. This can be done by looking at the pragma solidity flag at the top of the contract source code file. Only one specific version should be used, not multiple, for example:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.20;
```

**2.2.2. State and function visibility should be explicitly labeled**

State variables and functions have the default visibility assigned to them. The default visibility for state variables is `internal`, while the default visibility for functions is `public`. Labeling the visibility explicitly allows easy identification of the access scope.

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

**2.2.3. Functions that are never called internally should not have public visibility**

Public functions with reference type parameters copy calldata to memory when being executed, while external functions can read directly from calldata. Memory allocation uses more resources (gas) than reading directly from calldata. Therefore, public functions that are never called internally by the contract itself should have external visibility. Furthermore, this improves the readability of the contract, allowing clear distinction between functions that are externally used and functions that are also called internally.

Please note that the increase of gas usage is no longer applicable to smart contracts compiled with Solidity version `0.6.9` and later, as the data location keywords are required to be chosen and specified explicitly for reference type parameters regardless of the function visibility.

The testing can be done by checking for public functions that are never called internally from the contract itself.

```solidity
contract Visibility {
    uint256 private state;

    function setState(uint256 newState) external {
        state = newState;
    }

    function getState() view public returns (uint256) {
        return state;
    }
}
```

## Checklist

* Contract implementation should comply with the standards specification
* Built-in symbols should not be shadowed
* Floating pragma version should not be used
* State and function visibility should be explicitly labeled
* Functions that are never called internally should not have public visibility
