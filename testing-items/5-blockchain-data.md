# 5. Testing Control Flow

A normal EVM blockchain transaction can only call one function at the receiver address, and the executions will be determined by the code stored in the address. The outcome of each function depends on the input (the user’s input parameters and the world states); if the contract blindly trusts the user’s parameters, the user could manipulate the execution flow.

## 5.1. Reentrancy

If the contract has an external call (we usually call this as callback) to an address that the user can control, the user could call this function again or other functions in the execution flow (re-entrant). This could cause a problem if the contract does not expect their states to be changed while calling an external address. A state check can be done after the call to guarantee the invariance of the contract or by applying check-effect interactions.

**Testing**

**5.1.1 Reentrant calling should not negatively affect the contract states**

Reentrancy can happen when an external contract is invoked, allowing that contract to control the execution flow and perform reentrant calls. If the states are not completely updated or resolved, reentrancy can affect the invariant of the states.

The test can be done by checking the following condition:

* Check that external calling is not done to unknown contracts
* Check that measures are implemented to prevent reentrancy
* Check that there are invariant validation after the reentrant

For example, the `withdrawAll()` function has a reentrancy issue because it gives the external call to the user (`msg.sender`), and the state is updated (`balance[msg.sender] = 0`) after the external call.

```solidity
function withdrawAll() external {
    require(balance[msg.sender] >= 0,'Insufficient funds');
    payable(msg.sender).call{value: balance[msg.sender]}("");
    balance[msg.sender] = 0;
}i
```

When the `call()` function is executed, if the `fallback()` function is implemented on the `msg.sender` address, the fallback function will be called before the `balance[msg.sender]` is set to `0`. This can be used for reentrant calls to the `withdrawAll()` function for draining all tokens in the contract.

### 5.2. Input validation

The parameters of a function are the parts over which the user has control. The result of the function would normally depend on the input parameters. If the user can craft input parameters to the function that do not properly handle the parameter, e.g., a zero value, a negative value, or a blank array, the user could break an invariant of the contract.

**Testing**

**5.2.1. Lack of input validation**

If the function can only operate within some range of input, the contract should always validate the user input, which could invalidate the platform's invariants.

For example, the function below can deposit multiple tokens in one execution, and the caller will get a free gift from the contract. But the user can send a blank array as the input to bypass the depositing and get the gift freely.

```solidity
contract Gift {
    struct TokenData {
        address token,
        uint256 amount
    } 
    mapping(address => uint256) public totalValue;
  
    function participate(TokenData[] memory tokens) external {
        require(totalValue[msg.sender] == 0, "Each user can only participate once");
    
        for(uint256 i; i<tokens[i]; i++) {
            require(_isValidToken(tokens[i].token));  // only the whitelist tokens are allowed
            uint256 value = _calculateValue(tokens[i].token, tokens[i].amount); // get the token in $USD
            require(value > 100 ether, "Each deposit must worth more than $100.")
            IERC20(tokens[i].token).transferFrom(msg.sender, tokens[i].amount);
            totalValue[msg.sender] += value;
        }
        // A gift for every participant
        sendGift(msg.sender);
    }
}
```

## Checklist

* Reentrant calling should not invalidate the platform's invariants
* The function inputs should always be validated
