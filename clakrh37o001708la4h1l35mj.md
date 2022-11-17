# How GMX Limit Order and Long/Short Work

We can trade cryptos at market price with fully decentralized exchanges on blockchains. If we use limit orders or open long/short positions, the exchange platform cannot be 100% decentralized because blockchains cannot trigger some actions by themselves. There must be off-chain components watching the prices and executing your orders. Let's see how a half-decentralized exchange like [GMX](https://gmx.io/) works.

# Architecture overview

In general, there are 4 main components

+ Order managers: periphery contracts, e.g. `Router`, `PositionRouter`, `OrderBook`, etc. are the API (application program interface) for GMX. They receive user's orders, transfer fund, and store on-chain order info.
+ `Vault` is the contract that holds liquidity and handle trading function, e.g. swap, open/close/liquidate positions.
+ Price feeds: contracts that contain prices from multiple sources, e.g. CEX, on-chain pool, Chainlink, off-chain oracles.
+ Executors: off-chain components that update price to price feed contracts. They also check if the conditions of user's orders are met to submit execution transactions to order manager contracts.

![gmx-architecture-overview.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1667987306811/N3H23UZMI.png align="left")

In short, users submit orders to order managers, then executors watch the conditions. If the conditions are met, executors send execution signals to order managers. The order managers check the condition again, then call trading functions of the vault. The vault will use price from price feeds for calculations.

# Use case flow

On GMX, we can trade in spot and perpetual markets. Each type also has limit and market orders. Market orders of the spot exchange do not require an executor to watch the conditions, but the perpetual exchange and limit orders of the spot exchange do require. So, there are 3 main use cases:

+ **Market orders of spot exchange**: Clients call `swap()` to `Router` contract. The `Router` will unwrap token if the token is gas token and transfer fund from the client to `Vault` contract. Then, `Router` call `Vault` to swap tokens and transfer swapped token to the client. This process is done in an atomic on-chain transaction.
+ **Limit orders of spot exchange**: Clients call `createSwapOrder()` to `OrderBook` contract. The `OrderBook` will get fund from user and store the order in a queue. Off-chain components called order keepers will watch orders from the queue. If the order conditions are met, order keepers will call `executeSwapOrder()` to `PositionManager` contract, which is forwarded to `OrderBook`. Then, `OrderBook` checks conditions again and calls `swap()` in `Vault`.
+ **Perpetual exchange**: Clients call `createIncreasePosition` for opening and `createDecreasePosition` for closing a position to `PositionRouter` contract. This function call includes collateral and long/short info, e.g. index token, accepted price, size, etc. The `PositionRouter` will get fund (collateral) from the client, store the order and emit an event. Off-chain components called position keepers will read events, then they update prices and execute orders together.

![gmx-flow.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1668051716921/ixj2dQHNO.png align="left")

*Note: Executors is called keepers in GMX.

For price contracts, `Vault` only need to interact with `VaultPriceContract` which aggregates prices from multiple sources: Chainlink, on-chain AMM pools, predefined stable tokens ($1), and `FastPriceFeed` contract. Be careful, GMX can control the price when updating `FastPriceFeed`.

# Perpetual use case examples

## Long positions

To open long positions on GMX, users must use index token (long token) as collateral. If they use other tokens, GMX will swap to index token. When opening long positions, GMX only cares about entry price, collateral amount and position size. The leverage (`size/collateral`) and liquidation price can be deduced from that parameters.

A snapshot of the collateral is taken when the position is opened, so in this example, the collateral would be recorded as $10k and will not change even if the price of ETH changes.

For detail liquidation price calculation, please refer [here](https://gmxio.gitbook.io/gmx/trading#partial-liquidations). In general, there is 1 rule: loss must be less than collateral:

![long-liquidation.png.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1668661661297/XEWZTWnwP.png align="left")

![long-example.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1668660440705/zUcFGQyMi.png align="left")

## Short positions

To open short positions on GMX, users must use stable token, e.g. USDC, as collateral. If they use other tokens, GMX will swap to stable token. The rest are like long positions, except how to calculate lost and liquidation price:

![short-liquidation.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1668661669483/z9BqDLh-0.png align="left")

![short-example.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1668660954835/kHH8ocvwM.png align="left")
