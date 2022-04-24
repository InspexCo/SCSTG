# 7. Arithmetic

Mathematical operations on different programming languages and platforms may work differently. The arithmetic operations done in the smart contract should be able to safely handle the whole range of possible values.

## 7.1. Values should be checked before performing arithmetic operations to prevent overflows and underflows

Each state or variable can store values in a specific range depending on their data type. If the values are not checked before performing arithmetic operations, overflows or underflows can happen which perform wrap-around on the value, resulting in unintended outputs.

**Testing:**

Check that Solidity compiler version is before 0.8.0 and verify all arithmetic operations are not affected by overflows and underflows.

This can be done by looking at each arithmetic operation and examining whether the range of the result exceeds the range of the declared data type or not, for example:

```solidity
pragma solidity 0.7.6;

mapping (address => uint8) balances;

function transfer(address _to, uint8 _amount) external {
	require(balances[msg.sender] - _amount >= 0, "Insufficient balance");
	balances[msg.sender] -= _amount;
	balances[_to] += _amount;
}
```

In the source code above, if the `_amount` is higher than the balance of the sender, the result will be wrapped around from the underflow, allowing the condition in the require statement to be fulfilled. This can cause the balance of the sender to be inflated. Also, if the balance of the receiver is added with `_amount` and exceeds the limit of `uint8` (2^8 - 1), the balance will overflow and wrap around to 0.

**Solution:**

1. Check the arithmetic operations for overflows and underflows, e.g., using OpenZeppelin’s SafeMath library; or
2. Use Solidity compiler version 0.8.0 or above

## 7.2. Explicit conversion of types should be checked to prevent unexpected results

Data type of a variable can be cast to another type explicitly; however, if the destination type cannot hold the values of the original variable, unexpected values can be yielded from the truncation or padding.

Casting of types on Solidity works as follows:

**Integer Conversion:**

Converting a larger integer type to a smaller type will trim the higher bits.

```solidity
uint32 a = 0x12345678;
uint16 b = uint16(a); // b will be 0x5678
```

Converting a smaller integer type to a larger type will pad the higher bits, and the resulting value will be equal to the original.

```solidity
uint16 a = 0x1234;
uint32 b = uint32(a); // b will be 0x00001234
```

**Fixed-Size Byte Conversion:**

Converting a larger byte type to a smaller type will trim the lower bits.

```solidity
bytes4 a = 0x12345678;
bytes2 b = bytes2(a); // b will be 0x1234
```

Converting a smaller byte type to a larger type will pad the lower bits.

```solidity
bytes2 a = 0x1234;
bytes4 b = bytes4(a); // b will be 0x12340000
```

For more details, please refer to Solidity documentation: [https://docs.soliditylang.org/en/latest/types.html#explicit-conversions](https://docs.soliditylang.org/en/latest/types.html#explicit-conversions)

**Testing:**

Check that the type conversion supports the whole range of the intended values, or the trimming and padding are working as intended.

```solidity
contract Cast {
    uint128 public pricePerItem;
    mapping(address => uint256) public balances;

    constructor(uint128 _pricePerItem) {
        pricePerItem = _pricePerItem;
    }

    function buy(uint amount) external payable {
        uint128 price = uint128(pricePerItem * amount);
        require(msg.value >= price);
        balances[msg.sender] += amount;
    }
}
```

**Solution:**

Perform conditional checking to make sure that the whole range of possible values is supported.

## 7.3. Integer division should not be done before multiplication to prevent loss of precision

As integers do not support floating points, the result of division will be rounded down to the nearest whole integer. Performing division before multiplication can cause a loss of precision in the resulting calculation.

**Testing:**

Check that the division operation that results in truncation of precision is not done before multiplication.

```solidity
uint256 public constant FEE_RATE = 50;
uint256 public constant TOTAL_FEE = 10000;

function deposit(uint256 amount) public {
	uint256 depositFee = amount / TOTAL_FEE * FEE_RATE;
	uint256 deposit = amount - depositFee;
	balances[address(treasury)] += depositFee;
	balances[msg.sender] += deposit;
	token.transferFrom(msg.sender, address(this), amount);
}
```

For example, using the source code above, if the `amount` deposited is 87654321, the `depositFee` should be equal to:

```solidity
87654321 / 10000 * 50 = 438271.605
```

However, as the result is rounded down during the division, the result from the Solidity source code above will be equal to:

```solidity
87654321 / 10000 = 8765
8765 * 50 = 438250
```

**Solution:**

Order the operations to perform multiplication before division.
