# 4. Testing Privilege Function

Though smart contracts are deployed in a decentralized environment, some smart contracts can still be considered centralized smart contracts; they are two separate things. To achieve decentralization, a smart contract should exhibit some degrees of the following properties:

* Trustless: The participants should not need to rely on trust in a central authority or intermediary. The need for blind trust in third parties should not exist in a decentralized smart contract.
* Immutable: An ideal decentralized contract should be immutable, its code cannot be altered or tampered with after being deployed. This ensures that the contract's functionality and behavior remain consistent and predictable throughout its lifecycle.
* Transparent: The contract's code and state should be visible to all participants on the blockchain, for openness and accountability. Anyone can examine the contract's logic and verify its execution, enhancing trust and reducing the risk of malicious activity.
* Decentralized Governance: Decision-making and governance should be decentralized. It allows participants to collectively decide on the execution and evolution of the contract. This avoids concentration of power and ensures that the contract's governance aligns with the interests of its stakeholders.

## 4.1. Privilege functions <a href="#privilege-functions" id="privilege-functions"></a>

Privilege functions are functions that only some parties can access and have the ability to manipulate critical states of the contract unfairly. Only the states that require modifications by the privileged parties should be modifiable by the admin accounts. The admin accounts should have the fewest privileges possible to manage the operation of the smart contracts. With overly permissive rights, the privileged accounts can perform actions that are unfair to the smart contract users to gain benefits.

**Testing**

**4.1.1. State variables should not be unfairly controlled by privileged accounts**

Only the states that require modifications by the privileged parties should be modifiable by the admin accounts. The admin accounts should have the fewest privileges possible to manage the operation of the smart contracts. With overly permissive rights, the privileged accounts can perform actions that are unfair to the smart contract users to gain benefits.

The test can be done by checking that the critical states are not modifiable by the admin accounts, unless necessary, without providing an unfair advantage to the privileged parties.

```solidity
contract Platform {
    address public admin;
    boolean public isPaused;
  
    constructor() payable {}
  
    modifier onlyAdmin {
        require(msg.sender == admin);
        _;
    }

    function pause() external onlyAdmin {
        isPaused = true;
    }
    
    function unPause() external onlyAdmin {
        isPaused = false;
    }
}
```

**4.1.2. Privileged functions or modifications of critical states should be logged**

Event logs can be used by users to easily monitor the actions or changes happening in the smart contract. Execution of critical state modification functions should be logged to allow the community to monitor and take proper measures for each action being performed on the smart contract.

The test can be done by checking that events are emitted for the execution of all privileged functions.

```solidity
contract Platform {
    address public admin;
    boolean public isPaused;
    
    constructor() payable {}
  
    modifier onlyAdmin {
        require(msg.sender == admin);
        _;
    }
  
    event Pause();
    event UnPause();

    function pause() external onlyAdmin {
        isPaused = true;
        emit Pause();
    }
    
    function unPause() external onlyAdmin {
        isPaused = false;
        emit UnPause();
    }
}
```

## Checklist

* State variables should not be unfairly controlled by privileged accounts
* Privileged functions or modifications of critical states should be logged
