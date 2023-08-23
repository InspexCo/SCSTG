# 9. Testing Contract Upgradability

Smart contracts on multiple blockchains can be upgraded or designed to be upgraded. Modifications to contract logic can help resolve newly discovered issues. Nevertheless, upgradability without any restrictions opens up room for the holders of privileged accounts to insert malicious logic into the smart contract. Furthermore, the upgrade pattern increases the complexity of the code and may introduce new bugs to the smart contract.

## **9.1. Identify an upgradability in contract**

There are several forms of upgradeability. But in general, there are two instructions that provide some degree of upgradability in EVM: `delegatecall` and `selfdestruct`. A proxy contract is a common form of upgradable contract. It consists of the proxy part and the implementation part. They are usually separate contracts, but they are not necessary. A proxy contract simply relies on the `delegatecall` instruction to call the implementation contract. We could use this as criteria for an upgradeable contract. But, for the implementation contract, there is no concrete criteria to judge whether the contract is the implementation contract or not. A lesser known usecase of the `selfdestruct` instruction is for upgrading a contract. It is possible to deploy a different contract to the same address by using the `selfdestruct` instruction.

**Testing**

Smart contracts on multiple blockchains can be upgraded, or designed to be upgradable. Modifications of contract logic can help in resolving newly found issues. Nevertheless, upgradability without any restrictions opens up room for the holders of privileged accounts to insert malicious logic into the smart contract. Furthermore, the upgrade pattern increases the complexity of the code and may introduce new bugs to the smart contract.

**9.1.1. Identify a `delegatecall` instruction that could lead to the contract upgradability**

The `delegatecall` instruction can change the contract states with an external contract's logic.

The test can be done by checking the `delegatecall` instructions in the contract. The target address of the `delegatecall` instruction must not be an arbitrarily untrusted contract.

For example, the function below relies on the implementation of the `transfer()` function on the `masterToken` address. Though the target function must be named `transfer`, but the user can implement any logic on it. Instead of transferring token logic, it can increase the balance of the caller.

```solidity
function transfer(address masterToken, address to, uint256 amount) external {
    (bool success, ) = masterToken.delegatecall(
        abi.encodeWithSignature("transfer(address,uint256)", to, amount)
    );
    
    require(success, "Delegatecall failed");
}
```

**9.1.2. Identify a `selfdestruct` instruction that could lead to the contract upgradability**

The `selfdestruct` instruction can remove the code from the contract. It could be used to steal all the funds in the contract. Moreover, in special cases, the `selfdestruct` insturction can be used to upgrade the contract logic.

The test can be done by checking the `selfdestruct` instruction in the contract. The contract should not have the `selfdestruct` instruction.

For example, the function below has the `selfdestruct` instruction. After the contract is self-destructed, it is possible to redeploy the new logic to the same address with some technique, resulting in the contract upgrade.

```solidity
function emergencyWithdraw() external {
    require(owner == msg.sender);
    selfdestruct(payable(msg.sender));
}
```

## **9.2. The initialize function implementation**

Most of the upgradeable contracts have a special function commonly called an initialize function. It is a function for initializing important state variables. Since most of the upgradable contracts cannot rely on the constructor to do the same tasks due to the scheme of the upgradable design that the developer chose, The initialize function can compensate for the lack of constructor execution. The initialize function is an important function that can alter critical states of the contract.

**Testing**

**9.2.1. The initialize function could only be executed once by the authorized party**

The initial states of a smart contract can be initialized through the constructor or a special function designed for initialization. Initialization is required for the contracts that are using the proxy pattern, as the constructor cannot be used. It is important that the initialization be done only once by the authorized account to prevent the contract states from being overwritten. The test can be done by checking that the initialization function can be used only once, and only by the authorized party. The example Solidity code has the initialize() function that can set the owner state without any access control.

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

## **9.3 Upgradable proxy contract pitfalls**

An upgradable proxy pattern mainly has a proxy contract to store the state data and do execution on the implementation contract. The data in EVM is stored in a slot pattern that can store 32 bytes of data in each slot. The same data from the same storage slot on a proxy contract can represent different information depending on the implementation contract that the proxy contract refers to. A problem may arise when a proxy contract uses the 'delegatecall' to access a different implementation contract that accesses the storage slot in a different context.

**Testing**

**9.3.1. Storage slot allocation should not conflict**

A proxy contract relies on the `delegatecall` instruction to use logic from another contract to modify its own states. If the proxy contract uses the `delegatecall` instruction to call another contract that is not designed to support the call of a proxy contract, the states on the proxy contract could collide and the execution can yield an unexpected result.

For example, in the pair of the implementation contract and the proxy contract below, the proxy contract uses the storage on slot 0 and 1. The implementation contract also uses storage slot number 0. When the proxy contract calls the `initialize()` function to set the `owner` state, they will notice that the execution is reverted because storage slot 0 of the proxy contract has already stored the address of the implementation contract. Thus, the check `owner == address(0)` will not pass.

```solidity
contract ImplementationA {
    address owner; // slot 0
    
    function initialize() external {
        require(owner == address(0)); // read from the storage slot 0
        owner = msg.sender; // edit the storage slot 0
    }
    function emergencyWithdraw() external {
        require(owner == msg.sender);
        payable(msg.sender).call{value: address(this).balance}();
    }
    ...
}

contract SimpleProxy {
    address implementation; // slot 0
    address owner;          // slot 1
    
    constructor(address _implementation){
        implementation = _implementation;
        owner = msg.sender;
    }
    
    function changeImplementation(address newImplementation) external {
        require(owner == msg.sender);
        implementation = newImplementation; // edit the storage slot 0
    }
    
    fallback() external payable() {
       assembly {
            calldatacopy(0, 0, calldatasize())
            let result := delegatecall(gas(), implementation, 0, calldatasize(), 0, 0)
            returndatacopy(0, 0, returndatasize())

            switch result
            case 0 {
                revert(0, returndatasize())
            }
            default {
                return(0, returndatasize())
            }
        }
    }
}
```
