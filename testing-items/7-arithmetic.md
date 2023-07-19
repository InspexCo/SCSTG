# 7. Testing Randomness

EVM does not have a function for a contract to get a random number natively. In blockchain, the transactions will be sequentially executed in a batch; we called it a `block`. Every transaction in the same block will be executed within the same block parameters, e.g., block number and block timestamp. Relying on the block parameters as an entropy source is also not possible because other transactions within the same block can also access the same value and be able to generate the same value from the random.&#x20;

Implementing a random function in EVM is quite challenging, but not impossible. There are many ways to achieve it, but each way has a different trade-off, and some are considered insecure. We will show you how to test the random functions.

## 7.1. External Source

Since the contract cannot rely on the source of randomness itself, it would be more feasible to get the source of randomness from an external source, the oracle. The off-chain entity can generate a pseudo-random number and feed the number into the blockchain to act as a source of randomness for the blockchain.

**Testing**

**7.1.1. VRF**

VRF stands for Verifiable Random Function. It is a function that can emit a pseudo-random value, and the value can be verified to be really unbiasedly random. The VRF needs to stay off-chain to be able to produce an unpredictable number. For example, Chainlink's VRF ([https://docs.chain.link/vrf/v2/introduction](https://docs.chain.link/vrf/v2/introduction))

**7.1.2. Provenance hash**

Provenance hash is a kind of commit-and-reveal scheme. Usually used in situations where there is a party responsible for using some random values, The provenance hash is the hash of the random value that the party has privately generated off-chain. The party can publish the proof hash before revealing the result. When the random number has been revealed, the provenance hash can be used as proof of the number's integrity.&#x20;

## 7.2. Internal Source

Since the contract cannot rely on the built-in functions or the block parameter as a secure source of randomness, There are schemes that could circumvent this problem with some trade-offs. By using these schemes, the contract can use the not-secure random number from the built-in functions or the block parameter without being taken advantage of. There is a scheme called the commit-and-reveal scheme, or commitment scheme. This scheme has the participants commit their values first, then reveal the results of the committed values. By doing this, the participants cannot alter their result after knowing the revelation. There are many variations of the commit-and-reveal scheme, depending on the committed data.

**Testing**

**7.2.1. Future block hash**

The block hash is the hash value of the transaction in that block. It is a deterministic value once the block has been mined; otherwise, it is zero, including the block hash of the block that is older than 256 blocks. The commit-and-reveal scheme can use the future block number to decide the reveling block and use the blockhash of that block as the random number.

```solidity
contract FutureBlockhash {
    /// Stores the block of each address.
    mapping(address => uint256) public revealBlock;

    /// User select a future block as a source of randomness
    function commitBlock(uint256 _futureBlock) external {
        require(revealBlock[msg.sender] == 0, "Can only commit 1 time per address");
        require(_futureBlock > block.number + 10, "Must commit to a future block at least 10 blocks");
        revealBlock[msg.sender] = _futureBlock;
    }

    function getRandomness() external view returns (uint256) {
        uint256 selectedBlock = revealBlock[msg.sender];
        require(block.number > selectedBlock, "Not reach the target block yet");
        require(block.number <= selectedBlock + 256, "Commit expired");
        return blockhash(selectedBlock);
    }
}
```

## Checklist

* The design is properly implemented as the Commit & Reveal scheme
* Result from random value generation should not be predictable
