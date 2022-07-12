# 5. Blockchain Data

The usage of data on the blockchain, including the storage, retrieval, and modification, should be done properly to keep the integrity, and sometimes confidentiality, of the data.

## 5.1. Result from random value generation should not be predictable

Randomness on most blockchains is pseudo-random using a deterministic algorithm. Without a secure source of entropy, the result of an improperly implemented on-chain randomness can be predicted.

**Testing:**

Check that the result of the random function cannot be manipulated or predicted. If the block properties (`block.number`, `block.timestamp`, etc.) are used as the source of randomness or seed, the random result will be predictable.

For example, the following code snippet shows a commonly found method of randomization using the block hash of the last block. Anyone can fetches the value of the block hash and know the result of the randomization, giving advantages to those who know this value.

```solidity
function random(uint256 range) internal view returns (uint256) {
    return uint256(keccak256(abi.encodePacked(blockhash(block.number-1)))) % range;
}
```

**Solution:**

1. Implement a provably-fair and verifiable source of randomness, e.g., verifiable random function (VRF); or
2. Use an existing oracle service provider for source of randomness, such as [Chainlink VRF](https://docs.chain.link/docs/chainlink-vrf/); or
3. Use a commit-reveal scheme for randomness seed to prevent any party from knowing the result prior to the reveal, for example: [https://fravoll.github.io/solidity-patterns/randomness.html](https://fravoll.github.io/solidity-patterns/randomness.html)

## 5.2. Spot price should not be used as a data source for price oracles

Price oracle can be implemented by fetching the real-time on-chain data from a decentralized exchange contract. However, the spot price can easily be manipulated; therefore, the price fetched from the oracle can be manipulated by a malicious actor. This is especially impactful when combined with flash loans.

**Testing:**

Check that the price data used is not a spot price which can be easily manipulated.

The use of spot price can be found by searching for the use of reserves to calculate the price. This includes the use of the `getAmountOut()` function.

```solidity
function getPrice() external returns (uint256) {
    (uint256 reserve0, uint256 reserve1) = pair.getReserves();
    return reserve0 * PRECISION / reserve1;
}
```

**Solution:**

1. Use a time-weighted average price feed (Please note that this method does not work for low liquidity assets); or
2. Use external trusted decentralized price oracles

## 5.3. Timestamp should not be used to execute critical functions

In Geth and Parity Ethereum protocol implementation, the miner can manipulate the timestamp to be anywhere within 15 seconds of the block being validated. This allows the miner to precompute and use the timestamp that is the most favorable for the miner.

**Testing:**

Check that timestamp is not used to perform decisions in executing critical functions, unless the scale of the time-dependent event can vary by 15 seconds and maintain integrity.

Take a look at the following example contract:

```solidity
contract Roulette {
    uint public pastBlockTime;

    constructor() payable {}

    function spin() external payable {
        require(msg.value == 10 ether); // must send 10 ether to play
        require(block.timestamp != pastBlockTime); // only 1 transaction per block

        pastBlockTime = block.timestamp;

        if (block.timestamp % 15 == 0) {
            (bool sent, ) = msg.sender.call{value: address(this).balance}("");
            require(sent, "Failed to send Ether");
        }
    }
}
```

From the Roulette contract above, the miner can manipulate the timestamp to be divisible by 15 and mine that block to win the reward.

**Solution:**

Avoid using `block.timestamp`, or redesign the contract to remove the sensitive time-dependent event.

## 5.4. Plain sensitive data should not be stored on-chain

Data stored on the blockchain can be publicly viewed, so confidential data should not be submitted or stored on-chain. This is also true even for the states that have `private` visibility, since anyone can get the value by directly accessing the storage slot of the contract.

**Testing:**

Check that no sensitive data is designed to be stored on-chain, even with private state visibility.

```solidity
contract PrivateVault {
    bytes32 private secret;

    constructor(bytes32 _secret) {
        secret = _secret;
    }

    function withdraw(uint256 amount, bytes32 _secret) external {
        require(_secret == secret, "Incorrect secret");
        (bool success, ) = msg.sender.call{value: amount}("");
        require(success, "Withdraw failed");
    }
}
```

**Solution:**

1. Avoid storing private data on-chain; or
2. If privacy is required only for a specific period of time, use a commitment scheme

## 5.5. Modification of array state should not be done by value

Array state can be passed into a function by reference or by value, using the `storage` or `memory` keywords respectively. If the state is aimed to be modified, it should not be passed by value.

**Testing:**

Check for state modification of array that is passed by value.

Take a look at the following example contract:

```solidity
contract Memory {
    uint[1] public array;

    function set() public {
        setStorage(array); // update array
        setMemory(array); // do not update array
    }

    function setStorage(uint[1] storage arr) internal { // by reference
        arr[0] = 100;
    }

    function setMemory(uint[1] memory arr) internal { // by value
        arr[0] = 200;
    }
}
```

From the contract above, if the `set()` function is called, the value of `array[0]` state will be set to 100, instead of 200, since the `setMemory()` function accepts value as a parameter, not the reference to the storage slot.

**Solution:**

Modify the array state by passing the parameter by reference using the `storage` keyword.

## 5.6. State variable should not be used without being initialized

The default value of an uninitialized state is 0, or 0x0. If the state is not intended to be zero, the use of that state without initialization can cause unintended effects.

**Testing:**

Check for the non-zero states that are used in the contract without being initialized.

Take a look at the following example contract:

```solidity
contract Uninitialized{
    address payable treasury;
    address owner;

    constructor() {
        owner = msg.sender;
    }

    function setTreasury(address payable _treasury) external {
        require(msg.sender == owner, "Only owner can set treasury");
        treasury = _treasury;
    }

    function transferToTreasury() payable public {
        (bool success, ) = treasury.call{value: address(this).balance}("");
        require(success, "Transfer to treasury failed");
    }
}
```

In the contract above, it is possible for the `transferToTreasury()` function to be executed prior to the `setTreasury()` function. With the `treasury` state uninitialized, the fund will be transferred to the zero address and lost forever.

**Solution:**

Initialize all state variables with their intended value on their declarations or contract construction.
