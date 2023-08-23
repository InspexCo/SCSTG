# 1. Testing Arithmetic Operation and Conversion

Mathematical operations on different programming languages and platforms may work differently. The arithmetic operations done in the smart contract should be able to safely handle the whole range of possible values.

This testing will cover the arithmetic operations and type conversions of smart contracts. The native data types of numbers in EVM are signed and unsigned integers, which do not natively support floating number data types. Therefore, these concerns should be checked thoroughly when handling the mathematical operations.

## 1.1. Integer Overflow and Underflow

Integer overflow occurs when the result of an arithmetic operation exceeds the maximum value of the data type. The resulted data type will be determined by the data type of the operands. When the size of the result value is larger than the size of the determined data type, the extra bits will be discarded, and the stored value may not be the same as the actual value.

For example, if an unsigned 8-bit integer variable holds the maximum value of `255` (`1111 1111` in binary) and is incremented by `1`, the resulting value of `256` (`1 0000 0000` in binary) cannot be stored in the same data type. As a result, only the last 8 bits of the value are stored, which is 0 (`0000 0000` in binary).

On the other hand, when a value is decreased below the minimum limit by being subtracted with a positive integer, the result will be wrapped around, which is also known as integer underflow.

In Solidity's complier version prior 0.8.0, there is no builtin overflow and underflow protection in both the contract compilation and runtime. Thus, integer overflow and integer underflow can occur whenever arithmetic operations have happened.

**Testing**

Check the Solidity complier version of the contract. If the version of the Solidity compiler version is below 0.8.0, you need to verify that all arithmetic operations are not affected by overflows and underflows.

This can be done by looking at each arithmetic operation and examining whether the range of the result exceeds the range of the declared data type or not, for example:

```solidity
contract TestingOverflowUnderflowInteger {
  
    mapping (address => uint8) balances;

    function transfer(address _to, uint8 _amount) external {
        require(balances[msg.sender] - _amount >= 0, "Insufficient balance");
        balances[msg.sender] -= _amount;
        balances[_to] += _amount;
    }
}
```

In the snippet code above, if the `_amount` is higher than the balance of the sender (`msg.sender`), the result will be wrapped around from the underflow, allowing the condition in the require statement to be fulfilled. This can cause the balance of the sender to be inflated. Also, if the balance of the receiver is added with `_amount` and exceeds the limit of `uint8` (2^8 - 1), the balance will overflow and wrap around to 0.

If the version of the Solidity compiler version is 0.8.0 or higher, the overflow and underflow will only occur under `unchecked{}` block. Thus, there are two points for verifying:

**1.1.1. Solidity compiler version 0.8.0 and higher**

Verify that the overflows and underflows inside the `unchecked{}` blocks are proper expected behaviors.

```solidity
contract OverflowExample {
    uint8 public count = 255;
    function overflow() external {
        unchecked {
            count = count + 1;
        }
    }
}
```

**1.1.2. Solidity compiler version 0.8.0 and below**

Verify that the overflows and underflows that could occur are prevented if they break the contract's design or functionalities.

```solidity
contract PreventingUnderflowExample {
    mapping (address => uint256) balances;
    function checkBalanceAfterWithdraw(uint256 amount) external view returns (uint256){
        require(balances[msg.sender] >= amount, "insufficient balance");
        return balances[msg.sender] - amount;
    }
}
```

## 1.2. Precision Loss

The numeric data types that EVM fully natively supports are signed and unsigned integers with a size up to 256 bits. EVM also has fixed point data types for floating, but they are currently not fully supported yet (as of writing, the Solidity complier version is 0.8.20). To represent a floating point number, we commonly separate the number of decimals and treat those digits as a decimal part of the value. Therefore, the defined number of decimals is a fixed number. It cannot distinguish a value that is less than that decimal point.

For example, we declare a `uint256` variable and define its decimal to be `9`. It means that the last `9` least significant digits represent the decimal of the variable, and the value from the variable will be treated as follows:

|    Stored value | Represented value |
| --------------: | ----------------: |
| 123 000 000 000 |   123.000 000 000 |
|   3 141 592 653 |     3.141 592 653 |
|   1 000 000 000 |     1.000 000 000 |
|     123 456 789 |     0.123 456 789 |
|     100 000 000 |     0.100 000 000 |
|      10 000 000 |     0.010 000 000 |
|       1 000 000 |     0.001 000 000 |
|           1 234 |     0.000 001 234 |
|               1 |     0.000 000 001 |

When an integer is divided, the remainder will not continue being divined to produce a decimal part of the result, since, it is an integer division. A precision loss will occur if the remainder of the division is not zero.

A precision loss is inevitable by the nature of EVM, but the impacts depend on the difference between the magnitude of the loss value and the dividend. So, a precision loss problem will be insignificant if the dividend is sufficiently larger than the divisor.&#x20;

**Testing**

A precision loss can occur when there is division in the operation. There are two cases for testing the division:

**1.2.1. The rounding down of the division**

The result of division will always be rounded down, and it will be zero if the divisor is larger than the dividend. You must verify the impact of the result of the rounding down.

The test can be done by checking the division operations in the contract. If the divisor and the dividend are independent values, the divisor can be greater than the dividend and the result will be zero.

For example, the function below, it take the `amount` of the `depositToken` and mint a token (`shareToMint`) according to the proportion of the `amount` and the balance of the deposited token in the contract.

```solidity
import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract Valut is ERC20 {
    ERC20 public depositToken;
    
    function deposit(uint256 amount, address receiver) external {
        if(totalSupply == 0) {
            shareToMint = amount;
        } else {
            shareToMint = (amount * totalSupply) / depositToken.balanceOf(address(this));
        }
    
        depositToken.safeTransferFrom(msg.sender, address(this), amount);
    
        _mint(receiver, shareToMint);
    }
}
```

It is possible to use the rounding down to perform the attack as follows:

1. Attacker deposits 1 `wei` of token to the contract as the first depositor
2. Attacker attacker transfers 100 `ether` of tokens to the contract.
3. Victim deposits 200 `ether` of tokens into the contract. Due to the rounding down, the victim gets only 1 share (200 `ether` / 101 `ether`).
4. Attacker attacker withdraws 1 share to get 150 `ether`, resulting in a profit of 50 `ether`

**1.2.2. The order of division and multiplication**

To have the least impact from precision loss, it is better to have a larger dividend before it is divided. Since division and multiplication are commutative, calculating the multiplication part first and then doing the division will result in less impact from a precision loss.

The test can be done by checking that the division operation that results in truncation of precision is not done before multiplication.

For example, using the source code below:

```solidity
import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract Vault is ERC20 {
    uint256 public constant FEE_RATE = 50;
    uint256 public constant TOTAL_FEE = 10000;
    ERC20 public token;
  
    function deposit(uint256 amount) public {
        uint256 depositFee = amount / TOTAL_FEE * FEE_RATE;
        uint256 deposit = amount - depositFee;
        balances[address(treasury)] += depositFee;
        balances[msg.sender] += deposit;
        token.transferFrom(msg.sender, address(this), amount);
    }
}
```

If the amount deposited is 87654321, the `depositFee` should be equal to:

```
87654321 / 10000 * 50 = 438271.605
```

However, as the result is rounded down during the division, the result from the Solidity source code above will be equal to:

```
87654321 / 10000 = 8765
8765 * 50 = 438250
```

## 1.3. Type conversion

The data in EVM is stored in binary format. The data type of the states on the contract level is just how the contract perceives the stored data. The same data in storage or memory can represent different values depending on the data type that the contract will interpret.

At the contract level, the data type of the variables must conform with the operators' accepted data type (as regards the Solidity complier version `0.8.20`). The data type of a variable can be cast to another type explicitly; however, if the destination type cannot hold the values of the original variable, unexpected values can be yielded from the truncation or padding.

In this testing, we will focus only on the conversion of these types of variables: `intXX`, `uintXX`, `address`, and `bytesXX`. Since they occupy one slot of storage and can be freely converted around. There is a rule about the type conversion in solidity: you cannot change size, sign (negativity), and type in one conversion. So, there are three cases to consider when checking the conversions:

* the change of size
* the change of type
* the change of sign

**Testing**

**1.3.1. The change of size (Same type with different size conversion)**

It is a conversion that can occur on a data type that has many sizes, i.e., `intXX`, `uintXX`, and `bytesXX`. Converting a smaller data type into a larger one has no issues. But, in the case of converting from a larger data type into a smaller one, the value can be altered; you should check thoroughly that the result is intended. The types `uintXX` and `bytesXX` have different ways of padding and truncating the data. The user must beware when the conversion takes many steps, a different intermediate conversion could lead to a different conversion outcome.

The test can be done by checking the explicit conversions in the contracts. When there are conversions that reduce the size, a validation for the converted value must be enforced.

For example, a contract below locks a token and packs the token address and the amount into a struct. But the amount, with size 256 bits, has been converted into 96 bits. The stored amount in the contract will be less than the transferred amount if the transferred amount is greater than the maximum value of `uint96`.

```solidity
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";

contract Vault {
    struct UserData{
        address lockToken;
        uint96 amount;
    }

    mapping(address=>UserData) Users;

    function lock(address lockToken, uint256 amount) external{
        UserData user = Users[msg.sender];
        require(user.lockToken == address(0));
    
        IERC20(lockToken).safeTransferFrom(msg.sender, address(this), amount);
    
        user.lockToken = lockToken;
        user.amount = uint96(amount);
    }
}
```

**1.3.2.** **The change of type (Different types with the same size conversion)**

It is a conversion that changes the type but still keeps the size. It normally happens between `uintXX`, `bytesXX`, and `address`. The data type of `address` is special. The `address` type can be directly converted to `uint160` and `bytes20`. The underlying value (the binary value) will always be the same in the same size conversion case.

For example, the conversions below are coversions with the same size across the address, uint, and bytes datatypes. The results of the conversion are the same for the same datatype, including the conversion that increases the size of the datatype.

```solidity
contract ConvertType {
    address A = 0x123491f0489ED2EaAd9d7e61810b8c9cc7803ce9;

    uint160 public u160A        = uint160(A);                    // 103934185670909946599851984498482858906070564073
    bytes20 public b20A         = bytes20(A);                    // 0x123491f0489ed2eaad9d7e61810b8c9cc7803ce9

    bytes20 public b20u160A     = bytes20(u160A);                // 0x123491f0489ed2eaad9d7e61810b8c9cc7803ce9
    uint160 public u180b20A     = uint160(b20A);                 // 103934185670909946599851984498482858906070564073

    address public Ab20u160A    = address(b20u160A);             // 0x123491f0489ED2EaAd9d7e61810b8c9cc7803ce9
    address public Au180b20A    = address(u180b20A);             // 0x123491f0489ED2EaAd9d7e61810b8c9cc7803ce9

    bytes32 public b32b20A      = bytes32(b20A);                 // 0x123491f0489ed2eaad9d7e61810b8c9cc7803ce9000000000000000000000000
    bytes32 public b32b20u160A  = bytes32(b20u160A);             // 0x123491f0489ed2eaad9d7e61810b8c9cc7803ce9000000000000000000000000

    // Increase the size of the final data type

    uint256 public u256u160A    = uint256(u160A);                // 103934185670909946599851984498482858906070564073
    uint256 public u256u180b20A = uint256(u180b20A);             // 103934185670909946599851984498482858906070564073

    bytes32 public b32u160A     = bytes32(bytes20(b20u160A));    // 0x123491f0489ed2eaad9d7e61810b8c9cc7803ce9000000000000000000000000
    bytes32 public b32b20u180b20A  = bytes32(bytes20(u180b20A)); // 0x123491f0489ed2eaad9d7e61810b8c9cc7803ce9000000000000000000000000

    uint256 public u256b20A     = uint256(uint160(b20A));        // 103934185670909946599851984498482858906070564073
    uint256 public u256b20u160A = uint256(uint160(b20u160A));    // 103934185670909946599851984498482858906070564073

    // Decrease the size of the final data type

    uint160 public u160u256 = uint160(u256u160A);            	 // 103934185670909946599851984498482858906070564073
    uint160 public u160u256b32 = uint160(uint256(b32u160A)); 	 // 736717366702721042145392082337433340935348944896 *

    bytes20 public b20u256 = bytes20(b32u160A);              	 // 0x123491f0489ed2eaad9d7e61810b8c9cc7803ce9
    bytes20 public b20u256b32 = bytes20(bytes32(u256u160A)); 	 // 0x000000000000000000000000123491f0489ed2ea *
}
```

Some values will differ when we covert the value down with a changing of type.

**1.3.3. The change of sign (Different sign conversion)**

The `intXX` type is the only signed data type. Other data types in Solidity are considered unsigned data types. Therefore, the `intXX` type can only be converted into the `uintXX` type.&#x20;

The test can be done by checking the explicit conversions in the contracts that converting from and into `int` datatype.

```solidity
uint256 a = type(uint256).max-1000; // 115792089237316195423570985008687907853269984665640564039457584007913129638935
int256 b = -1000;                   // -1000

int256 intA = int256(a);            // -1001
uint256 uintb = uint256(b);         // 115792089237316195423570985008687907853269984665640564039457584007913129638936
```

## Checklist

* Values should be checked before performing arithmetic operations to prevent overflows and underflows.
* Integer division should not be done before multiplication to prevent loss of precision.
* Explicit conversion of types should be checked to prevent unexpected results.
