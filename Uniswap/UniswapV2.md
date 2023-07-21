### The codebase of Uniswap V2 contracts is split into two repositories:

    1. core
    2. periphery

1. The core repository stores these contracts:

    * UniswapV2ERC20 – an extended ERC20 implementation that’s used for LP-tokens. It additionally implements EIP-2612 to support off-chain approval of transfers.

    * UniswapV2Factory – similarly to V1, this is a factory contract that creates pair contracts and serves as a registry for them. The registry uses create2 to generate pair addresses–we’ll see how it works in details.

    * UniswapV2Pair – the main contract that’s responsible for the core logic. It’s worth noting that the factory allows to create only unique pairs to not dilute liquidity.

2. The periphery repository contains multiple contracts that make it easier to use Uniswap. Among them is **UniswapV2Router**, which is the main entrypoint for the Uniswap UI and other web and decentralized applications working on top of Uniswap. This contracts has an interface that’s very close to that of the exchange contract in Uniswap V1.

#### [Refer this link to learn more about how to add and remove liquidity]("https://jeiwan.net/posts/programming-defi-uniswapv2-1/")

***

### Tokens swapping

There a two ways of transferring tokens to someone:

1. By calling transfer method of the token contract and passing recipient’s address and the amount to be sent.

2. By calling approve method to allow the other user or contract to transfer some amount of your tokens to their address. The other party would have to call transferFrom to get your tokens. You pay only for approving a certain amount; the other party pays for the actual transfer.

***

### Price oracle

Uniswap, while being an on-chain application, can also serve as an oracle. The kind of prices provided by the price oracle in Uniswap V2 is called time-weighted average price, or TWAP. It basically allows to get an average price between two moments in time. To make this possible, the contract stores accumulated prices: before every swap, it calculates current marginal prices (excluding fees), multiplies them by the amount of seconds that has passed since last swap, and adds that number to the previous one.

Since Solidity doesn’t support float point division, calculating such prices can be tricky: if, for example, the ratio of two reserves is 2/3, then the price is 0. We need to increase precision when calculating marginal prices, and Unsiwap V2 uses UQ112.112 numbers for that.

***

### CREATE2

CREATE2 doesn’t use external state (like other contract’s nonce) to generate a contract address and lets us fully control how the address is generated. You don’t need to know nonce, you only need to know deployed contract bytecode (which is static) and salt (which is a sequence of bytes chosen by you).

```solidity
bytes memory bytecode = type(ZuniswapV2Pair).creationCode;
bytes32 salt = keccak256(abi.encodePacked(token0, token1));
assembly {
    pair := create2(0, add(bytecode, 32), mload(bytecode), salt)
}
```
**Salt** is a sequence of bytes that’s used to generate new contract’s address deterministically. We’re hashing pair’s token addresses to create the salt–this means that every unique pair of tokens will produce a unique salt, and every pair will have unique salt and address.

The above piece of code generates an address for every unique pair of tokens. We can now retrieve the address using the approach mentioned below!!

```solidity
pairAddress = address(
    uint160(
        uint256(
            keccak256(
                abi.encodePacked(
                    hex"ff",
                    factoryAddress,
                    keccak256(abi.encodePacked(token0, token1)),
                    keccak256(type(ZuniswapV2Pair).creationCode)
                )
            )
        )
    )
);
```

We build a sequence of bytes that includes:
* 0xff – this first byte helps to avoid collisions with CREATE opcode. (More details are in EIP-1014.)
* factoryAddress – factory that was used to deploy the pair.
* salt – token addressees sorted and hashed.
* hash of pair contract bytecode – we hash creationCode to get this value.
* Then, this sequence of bytes gets hashed (keccak256) and converted to address (bytes->uint256->uint160->address).

This whole process is defined in [EIP-1014](https://eips.ethereum.org/EIPS/eip-1014) and implemented in the CREATE2 opcode.

***
#### Link for codebase: https://github.com/Jeiwan/zuniswapv2.git

