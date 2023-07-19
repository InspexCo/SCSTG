# 6. Testing Access Control

Access control is the imposing of policies by preventing users from acting beyond the scope of their authorized permissions. Improper access control can lead to unauthorized information disclosure, data manipulation or loss, or the performance of business functions outside the user's capability. Smart contracts can have access controls to enforce the proper behavior of the intended work flow. Access control comprises two main components: authentication and authorization. Authentication and authorization are important for the smart contract's security. Authentication is a process for verifying the identity of a user. In EVM, a user uses a key pair as an identity (a public key) and proves the identity with another key (a private key). Authorization is a control over the accessibility of each user to the smart contract's functions.

## 6.1. Contract's authentication

In EVM, a user authenticates their transactions by using their private key. In a smart contract context, there are two special states that could be used as authentication: `tx.origin` and `msg.sender`. The `tx.origin` state is the address of the user who initiated the transaction, which will be the same throughout the whole transaction. The `msg.sender` state is the address of the immediate caller of the current external call, which can be changed every time an external call is made (a contract can also make an external call to itself).

There are some cases where the contract supports a scheme that the contract executes the functions on behalf of the users. The contract has to make sure that the referred `msg.sender` address is the address of the users, not the contract itself.

**Testing**

**6.1.1. tx.origin should not be used for authentication**

When `tx.origin` is used for authorization, it is possible for other contracts to call and perform actions using the permission of the transaction signer, which may not be intended.

The test can be done by checking that `tx.origin` is not used for authorization.

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

By using `tx.origin` for the authorization, an attacker can create a malicious contract that calls the `Treasury.withdrawTo()` function. If the real owner executes a function in that malicious contract, the funds can be unknowingly transferred from the contract.

**6.1.2. Authentication measures must be able to correctly identify the user**

The authentication measure used should be able to identify the user correctly without allowing any malicious actor to bypass or act as another user and correctly obtain the user's identity.

The test can be done by checking that the authentication cannot be bypassed, spoofed, or replayed by a malicious actor and can correctly obtain the user identity.

For example, a function below is for obtaining the identity of the user on a contract that supports ERC2771, where the signed transaction will be relayed by the relayer, resulting in the change of `msg.sender`. Therefore, if the contract directly uses `msg.sender` instead of the `_msgSender()` function, the contract will work incorrectly in the case where the relayer executes the transaction on behalf of the user.

```solidity
function _msgSender() internal override virtual view returns (address ret) {
    if (msg.data.length >= 20 && isTrustedForwarder(msg.sender)) {
        // At this point we know that the sender is a trusted forwarder,
        // so we trust that the last bytes of msg.data are the verified sender address.
        // extract sender address from the end of msg.data
        assembly {
            ret := shr(96,calldataload(sub(calldatasize(),20)))
        }
    } else {
        ret = msg.sender;
    }
}
```

## 6.2. Contract's authorization

Using role-based access control is common in a smart contract. It offers a smart contract for simple yet effective access control. It can be used to protect critical functions that can change the contract's critical states from being accessed by an unauthorized party. Role-based access control is an authorization that defines roles and authorizes privilege users, allowing them to perform critical actions.

**Testing**

**6.2.1. The roles are well defined and enforced**

Each function should have a list of eligible actors defined. Any other users outside of the roles defined should not be able to use the functions.

The test can be done by checking that each function can be executed only by the roles defined.

For example, the `claimAirdrop()` function should allow only the airdrop role to execute. In the following source code, there is no access control to ensure that function is executed by an eligible role or address from which anyone can claim the airdrop via the `claimAirdrop()` function.

```solidity
address airdropAddr = 0x0C0fFEEC0FfeeC0FFeec0FfeEC0FFeE000000000;

function claimAirdrop() external {
    // Claim Airdrop Code.
}
```

**6.2.2. The roles can be safely transferred**

When a contract has a function to transfer membership in a role to another address, the contract should have a mechanism to make sure that the new member is correctly transferred. If the membership is transferred to an invalid address, the lost membership cannot be re-gained, and no one will have access to functions that are associated with the role.

The example below is a function that requires the admin's permission to execute. But when executed, the old admin will lose the admin's rights, and the new admin can be any address.

```solidity
function transferAdmin(address newAdmin) external {
    require(msg.sender == admin);
    
    admin = newAdmin;
}
```

**6.2.3. Least privilege principle should be used for the rights of each role**

The least privilege principle should be applied to each role. The privilege of each role should be set only to do the task in their role. For example, the minter role should be allowed to mint the token only, and should not be allowed to do other actions.

The test can be done by checking that the privilege of each role is applied to the principle of least privilege. The privileges of each role should allow them to perform only their required tasks.

For example, the ERC-20 token was used as a reward token for the yield-farming contract. The `OWNER_ROLE` role was set to MasterChef's contract for minting rewards and the owner wallet to pause or unpause the token.

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

The contract should ensure that each role is allowed to perform its required tasks only by separating the tasks for each role. For example, the minter role should only be allowed to mint the token and should not have the right to do other tasks, e.g., pause token transfer.

## 6.3. Signature verification

The keypair on the EVM is generated using elliptic curve cryptography. It consists of a public key and a private key. The address of each user is derived from their public key, and only the corresponding private key can sign the transaction for the address. In EVM, there is a special function that can recover the public key from a message that is signed by a corresponded private key. With this function, there is a functionality that allows the user to sign a message (not a transaction), and the signed messages can be used by another party to do things on the user's behalf.

**Testing**

**6.3.1. The critical function should enforce an access control**

The functions that can change critical states should not be accessible by anyone. It must have access control that allows only the suitably qualified party to access it, e.g., DAO governance.

For example, the function below is a function for configuring the oracle of the contract, but the function can be accessed by anyone. An attacker could use this function to set a malicious oracle of the contract, manipulating the asset price.

```solidity
function setOracle(address newOracle) external {
    IOracle(newOracle).getPrice(); // sanity check
    
    oracle = newOracle;
}
```

## Checklist

* `tx.origin` should not be used for authentication
* Authentication measures must be able to correctly identify the user
* The roles' membership can be safely transferred
* Least privilege principle should be used for the rights of each role
* Access control should be defined and enforced for each actor roles
* Access control must be able to transfer to other entities safely
* The critical function should enforce an access control
