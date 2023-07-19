# 3. Testing External Interaction

Some smart contracts can operate without any interaction with external contracts. But for some, they cannot operate without interactions with external contracts. A smart contract in EVM can interact with each other by making external calls to another contract (a contract can technically make an external call to itself) to update their states or fetch some data.

Smart contracts that have functionality that depends on external contracts can have unforeseen risks that they cannot control. If the address of the external contract is not a trusted address, the owner of the external contract can manipulate the function that the other contracts rely on to gain benefit from the affected contracts.

## **3.1. Invoking external calls** <a href="#invoking-external-calls" id="invoking-external-calls"></a>

There are three types of call instructions in EVM: `STATICCALL`, `CALL`, and `DELEGATECALL`. For an internal function call, EVM uses `JUMP` instruction instead of the mentioned call instruction. The `STATICCALL` instruction is a call that does not change the states of blockchains. It can be used to call a function that doesn't change states, e.g., functions with `view` and `pure` visibility. Unlike the `CALL` and `DELEGATECALL` instructions, they can be used to call a contract and affect states on the blockchain. But they have a distinct use case.

The `CALL` instruction has the targeted address contract execute the function and return the value back to the caller.

Similarly, the `DELEGATECALL` instruction executes the function at the targeted address but with the states of the caller, which include the value of the `msg.sender` state. Since it executes with targeted contract implementation but the states of the caller are affected, using the `DELEGATECALL` instruction to call an untrusted contract could cause serious effects to the caller's states. The target of the `DELEGATECALL` instruction must always be controlled thoroughly.

**Testing**

**3.1.1. Unknown external components should not be invoked**

Check that only known and trusted contracts are invoked. Please have a look at the following example vulnerable contract.

```solidity
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";

interface IRouter {
    function swapExactTokensForTokens(
        uint amountIn,
        uint amountOutMin,
        address[] calldata path,
        address to,
        uint deadline
    ) external returns (uint[] memory amounts);
}

contract UnsafeVault {
    using SafeERC20 for IERC20;
    mapping(address => uint256) public balances;
    IERC20 public token;

    constructor(IERC20 _token) {
        token = _token;
    }

    function swapAndDeposit(IRouter router, IERC20 srcToken, uint256 amount, uint256 amountOutMin) external {
        srcToken.safeTransferFrom(msg.sender, address(this), amount);
        address[] memory path;
        path[0] = address(srcToken);
        path[1] = address(token);
        srcToken.safeIncreaseAllowance(address(router), amount);
        uint256[] memory amounts = router.swapExactTokensForTokens(amount, amountOutMin, path, address(this), block.timestamp);
        balances[msg.sender] += amounts[1];
    }

    function withdraw(uint256 amount) external {
        balances[msg.sender] -= amount;
        token.transfer(msg.sender, amount);
    }
}
```

In the contract above, the `router` can be set to any contract, allowing the attacker to implement a malicious contract that returns a high balance without actually swapping. Therefore, the smart contract should perform external calls to only known and trusted smart contracts, or define a whitelist of trustable contracts.

**3.1.2.** **Delegatecall should not be used on untrusted contracts**

Check that `delegatecall` is only used on trusted contracts.

```solidity
contract Worker {
    address public owner;
    function work(address worker) external {
        worker.delegatecall(bytes4(keccak256("work()")));
    }
}
```

In the example contract above, the `delegatecall` is used on a user supplied address, worker, allowing the attacker to create a contract with the `work()` function and perform arbitrary actions on the contract, such as assigning slot 0 with the new address as the `owner`.

**3.1.3. Invoke function with “this” keyword should be used with caution**

The `this` keyword in Solidity is used to retrieve the properties of the current smart contract address. When using `this` to invoke a function (this.'functionName') in the smart contract, it means the contract making an external call to the function to itself, which change the msg.sender from the former sender into the the contract itself.

The test can be done by checking for the use of this.'functionName' statement that affects the logic of the use of `msg.sender`, and make sure that it is correct according to the business design.

```solidity
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC721/IERC721.sol";

contract SimpleNFTMarketplace {
    uint256 public offerIdCounter;
    mapping(uint256 => Offer) public idToOffer;
    enum Status {
        NONE,
        CREATED,
        CANCELED,
        SWAPPED
    }
    struct Offer {
        address owner;
        IERC721 sellToken;
        uint256 sellId;
        IERC20 buyToken;
        uint256 buyAmount;
        Status status;
    }
    // deposit nft
    function offer(IERC721 sellToken, uint256 sellId, IERC20 buyToken, uint256 buyAmount) public {
        Offer memory offer = Offer(
            msg.sender,
            sellToken,
            sellId,
            buyToken,
            buyAmount,
            Status.CREATED
        );
        idToOffer[++offerIdCounter] = offer;
        IERC721(sellToken).transferFrom(msg.sender, address(this), sellId);
    }
   
    // sell multiple nft at once
    function bulkOffer(IERC721[] calldata sellTokens, uint256[] calldata sellIds, IERC20[] calldata buyTokens, uint256[] calldata buyAmounts) public {
        for (uint256 i = 0; i < sellTokens.length; ++i) {
            this.offer(sellTokens[i], sellIds[i], buyTokens[i], buyAmounts[i]);
        }
    }

    // accept offer
    function buy(uint256 offerId) public {
        Offer storage offer = idToOffer[offerId];
        require(offer.status == Status.CREATED, "invalid status");
        offer.status = Status.SWAPPED;
        IERC20(offer.buyToken).transferFrom(msg.sender, offer.owner, offer.buyAmount);
        IERC721(offer.sellToken).transferFrom(address(this), msg.sender, offer.sellId);
    }
}
```

The `offer()` function allows users to offer NFTs for sale through the `buy()` function, and the `bulkOffer()` function allows users to offer multiple NFTs for sale in a single transaction. The owner of the offered NFTs will be `msg.sender`. However, because the `bulkOffer()` function applies `this.offer()`, the caller will be the contract address rather than an EOA. This means that an attacker could steal NFTs from the contract by calling the `bulkOffer()` function with existing NFTs in the contract, along with a worthless token, and then executing the `buy()` function to acquire those NFTs.

## Checklist

* Unknown external components should not be invoked
* `delegatecall` should not be used on untrusted contracts
* Calling itself is counted as making an external call to itself, so the invariants should be adjusted to fulfill
