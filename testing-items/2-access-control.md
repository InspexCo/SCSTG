# 2. Access Control

Access control is the imposing of policy by preventing users from acting beyond the scope of their authorized permissions. Improper access control can lead to unauthorized information disclosure, data manipulation or loss, or performing of business functions outside the user's capability.

## 2.1. Contract self-destruct should not be done by unauthorized actors

`selfdestruct` instruction can be used to delete a smart contract from the blockchain and send the remaining ether to the destination address. Improper access control on the function with this ability allows malicious actors to terminate the smart contract.

**Testing:**

Check if there’s any function that invokes the `selfdestruct` instruction, and if there is, make sure that the function can only be called by the authorized parties only under necessary circumstances

This can be done by searching for all functions with `selfdestruct` or `suicide` option in the contract, and check the access control for the function, if unauthorized actors can execute the function, it is vulnerable, for example:

```solidity
contract Storage {
	address payable private owner;
	uint256 number;
	constructor() {
		owner = msg.sender;
	}
	function setNumber(uint256 _number) external {
		number = _number;
	}
	function getNumber() external returns (uint256) {
		return number;
 	}
	function kill() public {
		selfdestruct(owner);
	}
}
```

As seen in the example above, the `kill()` function is a public function without any access control, allowing anyone to destroy this smart contract.

**Solution:**

1. Remove the self-destruct functionality from the smart contract; or
2. If the functionality is required, implement a multisig scheme that requires multiple trusted parties to approve the self-destruct action

## 2.2. Contract ownership should not be modifiable by unauthorized actors

Functions with the ability to transfer the ownership of the contract should have a proper access control measure implemented to prevent unauthorized parties from taking over the contract ownership.

**Testing:**

If the contract ownership can be transferred, check that only the authorized parties can perform the ownership transfer action.

This can be done by searching for all functions that can modify the state that stores the ownership of the contract. If any unauthorized party can use those functions to modify the state, it is vulnerable, for example:

```solidity
contract Owner {
    address payable private owner;
    constructor() {
        owner = msg.sender;
    }
    function changeOwner(address _owner) external {
        require(msg.sender == _owner, "Only owner can transfer ownership");
        owner = _owner;
    }
    function withdraw() external {
        require(msg.sender == owner, "Only owner can withdraw");
        (bool sent, bytes memory data) = msg.sender.call{value: balance}("");
        require(sent, "Failed to send Ether");
    }
}
```

In the contract above, the `changeOwner()` function checks that the `msg.sender` is the `_owner`; however, `_owner` is a function parameter which can be controlled by the caller, not the contract state that saves the address of the owner. Therefore, anyone can call this function to change the contract owner.

**Solution:**

Implement an access control measure to allow only the authorized parties to manage the ownership of the contract.

From the example above, the flaw can be resolved by editing the checking condition for the `changeOwner()` function to check from the state instead of checking from the parameter.

```solidity
function changeOwner(address _owner) external {
    require(msg.sender == owner, "Only owner can transfer ownership");
    owner = _owner;
}
```

## 2.3. Access control should be defined and enforced for each actor roles

Each function should have a list of eligible actor roles defined. Any other users outside of the roles defined should not be able to use the functions.

**Testing:**

Check that each function can be executed only by the roles defined.

For example, the `claimAirdrop()` function should allow only the airdrop role to execute. In the following source code, there is no access control to ensure that function is executed by an eligible role/address which anyone can claim the airdrop via the `claimAirdrop()` function.

```solidity
address airdropAddr = 0x0C0fFEEC0FfeeC0FFeec0FfeEC0FFeE000000000;

function claimAirdrop() external {
    // Claim Airdrop Code.
}
```

**Solution:**

Implement an access control measure to prevent unauthorized actors from using the functions.

For example, validating that function is executed by eligible role/address. In this case, using the `require()` function to ensure that function is executed by the `airdropAddr` address.

```solidity
address airdropAddr = 0x0C0fFEEC0FfeeC0FFeec0FfeEC0FFeE000000000;

function claimAirdrop() external {
    require(airdropAddr == msg.sender, "Only Airdrop Address!");
    // Claim Airdrop Code.
}
```

## 2.4. Authentication measures must be able to correctly identify the user

The authentication measure used should be able to identify user correctly without allowing any malicious actor to bypass or act as another user.

**Testing:**

Check that the authentication cannot be bypassed, spoofed, or replayed by a malicious actor

**Solution:**

Implement an authentication scheme based on the business design that prevents a malicious actor from bypassing the authentication or acting as another user.

## 2.5. Smart contract initialization should be done only once by an authorized party

The initial states of a smart contract can be initialized through the constructor or a special function designed for the initialization. Initialization is required for the contracts that are using the proxy pattern, as the constructor cannot be used. It is important that the initialization should be done only once by the authorized account to prevent the contract states from being overwritten.

**Testing:**

Check that the initialization function can be used only once, and only by the authorized party.

```solidity
contract Initialize {
    address public owner;

    function initialize() external {
        owner = msg.sender;
    }

    function withdraw(address to, uint256 amount) external {
        require(msg.sender == owner, "Only the owner can withdraw");
        (bool success, ) = to.call{value: amount}("");
        require(success, "Transfer failed");
    }
}
```

**Solution:**

Implement a condition check to prevent the initialization from being executed multiple times or by unauthorized users.

```solidity
contract Initialize {
    address public owner;
    bool isInitialized;

    function initialize() external {
        require(!isInitialized, "The contract is already initialized");
        owner = msg.sender;
        isInitialized = true;
    }

    function withdraw(address to, uint256 amount) external {
        require(msg.sender == owner, "Only the owner can withdraw");
        (bool success, ) = to.call{value: amount}("");
        require(success, "Transfer failed");
    }
}
```

## 2.6. tx.origin should not be used for authorization

When `tx.origin` is used for authorization, it is possible for other contracts to call and perform actions using the permission of the transaction signer, which may not be intended.

**Testing:**

Check that `tx.origin` is not used for authorization.

Take a look at the following example contract:

```solidity
contract Treasury {
    address public owner;

    constructor() {
        owner = tx.origin;
    }

    function withdrawTo(address to, uint256 amount) payable external {
        require(tx.origin == owner, "Only the owner can use this function");
        (bool success, ) = to.call{value: amount}("");
        require(success, "Transfer failed");
    }

    receive() external payable {}
}
```

By using the `tx.origin` for the authorization, an attacker can create a malicious contract that calls the `Treasury.withdrawTo()` function. If the real owner execute a function in that malicious contract, the funds can be unknowingly transferred from the contract.

**Solution:**

Change the authorization checking to `msg.sender`.

```solidity
// SPDX-License-Identifier: MIT
contract Treasury {
    address public owner;

    constructor() {
        owner = msg.sender;
    }

    function withdrawTo(address to, uint256 amount) external {
        require(msg.sender == owner, "Only the owner can use this function");
        (bool success, ) = to.call{value: amount}("");
        require(success, "Transfer failed");
    }

    receive() external payable {}
}
```

## 2.7. Signature verification must contain all necessary data and have protection against signature replay attack

The Ethereum signed message can be used to authorize a user without creating a transaction on the blockchain. However, when using signature verification as an authorization method, the signed message must include all necessary data according to the authorization context. Furthermore, if a signed message is designed to be used only once, the signature replay attack protection mechanism must be implemented.

**Testing:**

The following `NFTMarketplace` contract has a `_verifySignature()` function to verify the signature when the user executes `buy()` function. The seller must sign the message first, and then the buyer will supply the signature and the signer as inputs when calling `buy()` function. However, this implementation is vulnerable to a signature replay attack and also lacks necessary data that should be in the signed message, such as order type and expiration time.

```solidity
contract NFTMarketplace {
    // NFTMarketplace contract logic

    function _verifySignature(address _nftAddress, uint256 _tokenId, uint256 _price, bytes memory _signature, address _signer) internal pure returns(bool) {
        bytes32 messageHash = keccak256(abi.encodePacked(_nftAddress, _tokenId, _price));
        bytes32 signedMessageHash = getEthSignedMessageHash(messageHash);
        return recoverSigner(signedMessageHash, _signature) == _signer;
    }

    function buy(address _nftAddress, uint256 _tokenId, uint256 _price, bytes memory _signature, address _signer) external {
        require(_verifySignature(_nftAddress, _tokenId, _price, _signature, _signer));
        
        // buy function logic
    }

    // NFTMarketplace contract logic
}
```

From the example above, since it lacks an order type in the signed message, the attacker can use the signed message of the seller in both buy and sell. The signed message is also valid forever because there is no expiration check. The valid signature can also be used multiple times.

**Solution:**

Adding `_isSell` and `_expiry` to the message when signing it and validating it in the `buy()` function should prevent the attacker from abusing the order type and permanent validity of the signature. To prevent the signature replay attack, add the `_nonce` to the message and mark the signature as used with the `usedSignature` mapping state.

Additionally, adding `address(this)` to the message can prevent the cross-contract signature replay attack, and adding `block.chainid` to the message can prevent the cross-chain signature replay attack. Furthermore, the EIP712 can be used to improve the readability of the signed message. The `EIP712Domain` also includes the platform name, version, verifying contract address, and chain ID, which is enough to protect against cross-contract and cross-chain signature replay attacks.

```solidity
contract NFTMarketplace {
    // NFTMarketplace contract logic

    function _verifySignature(address _nftAddress, uint256 _tokenId, uint256 _price, bool _isSell, uint256 _expiry, uint256 _nonce, bytes memory _signature, address _signer) internal view returns(bool) {
        bytes32 messageHash = keccak256(abi.encodePacked(_nftAddress, _tokenId, _price, _isSell, _expiry, _nonce, address(this), block.chainid));
        bytes32 signedMessageHash = getEthSignedMessageHash(messageHash);
        return recoverSigner(signedMessageHash, _signature) == _signer;
    }

    function buy(address _nftAddress, uint256 _tokenId, uint256 _price, bool _isSell, uint256 _expiry, uint256 _nonce, bytes memory _signature, address _signer) external payable {
        require(_isSell);
        require(_expiry >  block.number);
        require(!usedSignature[_signature]);
        require(_verifySignature(_nftAddress, _tokenId, _price, _isSell, _expiry, _nonce, _signature, _signer));
        usedSignature[_signature] = true;
        
        // buy function logic
    }

    // NFTMarketplace contract logic
}
```