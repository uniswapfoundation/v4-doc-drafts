# Introduction to Dynamic Fees

Uniswap v4 introduces dynamic fees, allowing for flexible and responsive fee structures managed through hooks. This feature enables pools to adapt fees to changing market conditions, potentially improving liquidity provider profitability and overall market efficiency.

## What are Dynamic Fees?

Unlike the static fee tiers in Uniswap v3 (0.05%, 0.30%, 1.0%) or the single fee in v2, dynamic fees in v4 can:

- Adjust in real-time based on various market conditions
- Change on a per-swap basis
- Allow for any fee percentage (e.g., 4.9 bips, 10 bips)
- Be updated at various intervals (yearly, per block, or per transaction)

## Motivation and Benefits of Dynamic Fees

1. **Improved Pricing of Volatility:** Adapt fees to market volatility, similar to traditional exchanges adjusting bid-ask spreads.
2. **Order Flow Discrimination:** Price different types of trades (e.g., arbitrage vs. uninformed) more accurately.
3. **Improved Market Efficiency and Stability:** Fees can adjust to reflect real-time market conditions, optimizing for both liquidity providers and traders. Dynamic fees could help dampen extreme market movements by adjusting incentives in real-time.
4. **Enhanced Capital Efficiency and Liquidity Provider Returns:** By optimizing fees, pools can attract more liquidity and facilitate more efficient trading. More accurate fee pricing could lead to better returns for liquidity providers, potentially attracting more capital to pools.
5. **Better Risk Management:** During high volatility, fees can increase to protect liquidity providers from impermanent loss.
6. **Customizable Strategies:** Enable complex fee strategies for specific token pairs or market segments.

## Dynamic Fees Use Cases

1. **Volatility-Based Fees:** Adjust fees based on the historical or realized volatility of the asset pair.
2. **Volume-Based Fees:** Lower fees during high-volume periods to attract more trades, and increase fees during low-volume periods to compensate liquidity providers.
3. **Time-Based Fees:** Implement different fee structures for different times of day or days of the week, based on historical trading patterns.
4. **Market Depth-Based Fees:** Adjust fees based on the current liquidity depth in the pool.
5. **Cross-Pool Arbitrage Mitigation:** Dynamically adjust fees to discourage harmful arbitrage between different pools or exchanges.
6. **Gas Price-Responsive Fees:** Adjust fees based on network congestion and gas prices to ensure profitability for liquidity providers.
7. **Event-Driven Fees:** Implement special fee structures during significant market events or token-specific occurrences.
8. **Lookback approach:** Set the fee to match the most profitable fee tier of external pools with the same asset pair over a recent period.
9. **Price oracle approach:** Use an external price oracle to determine the correct asset price and adjust fees based on how trades move the pool price relative to this external price.
10. **Price momentum approach:** Analyze recent price history and asymmetrically adjust fees based on trade direction.
11. **Asset composition approach:** Lower fees for trades that balance the pool and higher fees for trades that imbalance it.
12. **Transaction-source based approach:** Provide lower fees for transactions routed through certain aggregators or sources less likely to be arbitrage trades.

## Dynamic Fees Mechanism

### 2.1 Initialize Dynamic Fee Pool

When creating a pool, the creator can set whether the pool has a static fee (and its value) or uses dynamic fees. These capabilities are determined by immutable flags set at pool creation.

Uniswap v4 uses a specific flag to indicate that a pool has dynamic fees:

```solidity
uint24 public constant DYNAMIC_FEE_FLAG = 0x800000;
```

This flag is used when initializing a pool to signal that it will use dynamic fees.

To initialize a pool with dynamic fees, use the `DYNAMIC_FEE_FLAG` in the `PoolKey`:

```solidity
import {LPFeeLibrary} from "v4-core/src/libraries/LPFeeLibrary.sol";

PoolKey memory poolKey = PoolKey({
    currency0: currency0,
    currency1: currency1,
    fee: LPFeeLibrary.DYNAMIC_FEE_FLAG,
    tickSpacing: 60,
    hooks: IHooks(dynamicFeeHook)
});

manager.initialize(poolKey, SQRT_PRICE_1_1, ZERO_BYTES);
```

This code creates a `PoolKey` with the `DYNAMIC_FEE_FLAG`, indicating that the pool uses dynamic fees, allowing the associated hook contract to adjust fees as needed. By default, dynamic-fee-pools initialize with a 0% fee. Pool creators can use the `afterInitialize` hook to set the initial fee.

### 2.2 PoolManager Interaction

Dynamic fees can be updated by calling the `updateDynamicLPFee` function on the PoolManager contract.

```solidity
function updateDynamicLPFee(PoolKey memory key, uint24 newDynamicLPFee) external {
        if (!key.fee.isDynamicFee() || msg.sender != address(key.hooks)) {
            UnauthorizedDynamicLPFeeUpdate.selector.revertWith();
        }
        newDynamicLPFee.validate();
        PoolId id = key.toId();
        _pools[id].setLPFee(newDynamicLPFee);
    }
```

Alternatively, hooks can use the `beforeSwap` callback to dynamically set fees for each swap.

```solidity
function beforeSwap(IHooks self, PoolKey memory key, IPoolManager.SwapParams memory params, bytes calldata hookData)
    internal
    returns (int256 amountToSwap, BeforeSwapDelta hookReturn, uint24 lpFeeOverride)
{
    // ... (other logic)
    if (key.fee.isDynamicFee()) lpFeeOverride = result.parseFee();
    // ... (additional logic)
}
```

This allows hooks to override the LP fee for each swap in dynamic fee pools.

The `swap` function in PoolManager demonstrates how dynamic fees are applied during swaps. The `lpFeeOverride` parameter allows for dynamic fee adjustment on a per-swap basis:

```solidity
function swap(PoolKey memory key, IPoolManager.SwapParams memory params, bytes calldata hookData)
    external
    override
    onlyWhenUnlocked
    noDelegateCall
    returns (BalanceDelta swapDelta)
{
    // ... (validation and setup)
    int256 amountToSwap;
    uint24 lpFeeOverride;
    (amountToSwap, beforeSwapDelta, lpFeeOverride) = key.hooks.beforeSwap(key, params, hookData);
    
    swapDelta = _swap(
        pool,
        id,
        Pool.SwapParams({
            tickSpacing: key.tickSpacing,
            zeroForOne: params.zeroForOne,
            amountSpecified: amountToSwap,
            sqrtPriceLimitX96: params.sqrtPriceLimitX96,
            lpFeeOverride: lpFeeOverride
        }),
        params.zeroForOne ? key.currency0 : key.currency1
    );
    // ... (post-swap logic)
}
```

### 2.3 Updating Dynamic Fees

There are two main ways to update dynamic fees:

1. The hook contract can call `IPoolManager.updateDynamicLPFee(PoolKey memory key, uint24 newDynamicLPFee)`

```solidity
function updateDynamicLPFee(PoolKey memory key, uint24 newFee) external {
    if (!key.fee.isDynamicFee() || msg.sender != address(key.hooks)) {
        revert UnauthorizedDynamicLPFeeUpdate();
    }
    newFee.validate();
    PoolId id = key.toId();
    _pools[id].setLPFee(newFee);
}
```

This updateDynamicLPFee function updates the LP fee for a dynamic fee pool. It can only be called by the pool's hook contract.

2. Uniswap v4 introduces an override fee mechanism that allows hooks to set a fee for a specific swap:

```solidity
uint24 public constant OVERRIDE_FEE_FLAG = 0x400000;
```

This flag can be used in the `beforeSwap` hook to override the stored LP fee for a specific swap.

To do this, you would use `beforeSwap` and return a valid fee with its 2nd bit set to 1 (i.e., `fee | LPFeeLibrary.OVERRIDE_FEE_FLAG`)

Using `beforeSwap` is more gas-efficient for fees that may change on every swap. **Note:** the fee returned by `beforeSwap` is not saved to the PoolManager contract.

## Volatility-Based Dynamic Fee Hook Implementation

This example demonstrates a complete implementation of a volatility-based dynamic fee hook for Uniswap v4, incorporating all key components and functions.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {BaseHook} from "@uniswap/v4-core/contracts/BaseHook.sol";
import {Hooks} from "@uniswap/v4-core/contracts/libraries/Hooks.sol";
import {IPoolManager} from "@uniswap/v4-core/contracts/interfaces/IPoolManager.sol";
import {PoolKey} from "@uniswap/v4-core/contracts/types/PoolKey.sol";
import {PoolId, PoolIdLibrary} from "@uniswap/v4-core/contracts/types/PoolId.sol";
import {LPFeeLibrary} from "@uniswap/v4-core/contracts/libraries/LPFeeLibrary.sol";
import {BeforeSwapDelta, BeforeSwapDeltaLibrary} from "@uniswap/v4-core/contracts/types/BeforeSwapDelta.sol";

interface IVolatilityOracle {
    function realizedVolatility() external view returns (uint256);
    function latestTimestamp() external view returns (uint256);
}

contract VolatilityBasedFeeHook is BaseHook {
    using PoolIdLibrary for PoolKey;

    uint256 public constant HIGH_VOLATILITY_TRIGGER = 1400; // 14%
    uint256 public constant MEDIUM_VOLATILITY_TRIGGER = 1000; // 10%
    uint24 public constant HIGH_VOLATILITY_FEE = 10000; // 1%
    uint24 public constant MEDIUM_VOLATILITY_FEE = 3000; // 0.3%
    uint24 public constant LOW_VOLATILITY_FEE = 500; // 0.05%

    IVolatilityOracle public immutable volatilityOracle;
    uint256 public lastFeeUpdate;
    uint256 public constant FEE_UPDATE_INTERVAL = 1 hours;

    constructor(IPoolManager _poolManager, IVolatilityOracle _volatilityOracle) BaseHook(_poolManager) {
        volatilityOracle = _volatilityOracle;
    }

    function getHookPermissions() public pure override returns (Hooks.Permissions memory) {
        return Hooks.Permissions({
            beforeInitialize: false,
            afterInitialize: true,
            beforeModifyLiquidity: false,
            afterModifyLiquidity: false,
            beforeSwap: true,
            afterSwap: false,
            beforeDonate: false,
            afterDonate: false,
            noOp: false
        });
    }

    function afterInitialize(address, PoolKey calldata key, uint160, int24, bytes calldata)
        external
        override
        returns (bytes4)
    {
        uint24 initialFee = getFee();
        poolManager.updateDynamicLPFee(key, initialFee);
        return IHooks.afterInitialize.selector;
    }

    function beforeSwap(address, PoolKey calldata key, IPoolManager.SwapParams calldata, bytes calldata)
        external
        override
        returns (bytes4, BeforeSwapDelta, uint24)
    {
        if (block.timestamp >= lastFeeUpdate + FEE_UPDATE_INTERVAL) {
            uint24 newFee = getFee();
            lastFeeUpdate = block.timestamp;
            return (IHooks.beforeSwap.selector, BeforeSwapDeltaLibrary.ZERO_DELTA, newFee | LPFeeLibrary.OVERRIDE_FEE_FLAG);
        }
        return (IHooks.beforeSwap.selector, BeforeSwapDeltaLibrary.ZERO_DELTA, 0);
    }

    function getFee(address, PoolKey calldata) public view returns (uint24) {
    uint256 realizedVolatility = volatilityOracle.realizedVolatility();
    if (realizedVolatility > HIGH_VOLATILITY_TRIGGER) {
        return HIGH_VOLATILITY_FEE;
    } else if (realizedVolatility > MEDIUM_VOLATILITY_TRIGGER) {
        return MEDIUM_VOLATILITY_FEE;
    } else {
        return LOW_VOLATILITY_FEE;
    }
  }
}
```

### 3.1 Volatility-Based Fee Structure

This hook contract example sets up the structure for a volatility-based fee system, defining thresholds and corresponding fees:

```solidity
contract VolatilityBasedFeeHook is BaseHook {
    uint256 public constant HIGH_VOLATILITY_TRIGGER = 1400; // 14%
    uint256 public constant MEDIUM_VOLATILITY_TRIGGER = 1000; // 10%
    uint24 public constant HIGH_VOLATILITY_FEE = 10000; // 1%
    uint24 public constant MEDIUM_VOLATILITY_FEE = 3000; // 0.3%
    uint24 public constant LOW_VOLATILITY_FEE = 500; // 0.05%

    IVolatilityOracle public immutable volatilityOracle;

    constructor(IPoolManager _poolManager, IVolatilityOracle _volatilityOracle) BaseHook(_poolManager) {
        volatilityOracle = _volatilityOracle;
    }

    // Implementation of getFee and other functions...
}
```

where:

- High volatility tier: > 14% (fee: 1%)
- Medium volatility tier: 10%-14% (fee: 0.30%)
- Low volatility tier: < 10% (fee: 0.05%)

The constructor sets up the initial parameters and connections to required contracts (PoolManager and VolatilityOracle).

### 3.2 Realized Volatility Oracle

The contract utilizes an oracle to provide historical data on price movements for informed fee adjustments. This could be implemented as an external service or an on-chain mechanism tracking recent price changes.

```solidity
interface IVolatilityOracle {
    function realizedVolatility() external view returns (uint256);
    function latestTimestamp() external view returns (uint256);
}
```

### 3.3 getFee Function Implementation

The getFee function calculates the fee based on the current volatility level and returns the appropriate fee rate. This function implements the logic for dynamically calculating fees based on current conditions. The getFee function should return a fee value based on your chosen criteria (e.g., volatility, volume, etc.).

```solidity
function getFee(address, PoolKey calldata) external view returns (uint24) {
    uint256 realizedVolatility = volatilityOracle.realizedVolatility();
    if (realizedVolatility > HIGH_VOLATILITY_TRIGGER) {
        return HIGH_VOLATILITY_FEE;
    } else if (realizedVolatility > MEDIUM_VOLATILITY_TRIGGER) {
        return MEDIUM_VOLATILITY_FEE;
    } else {
        return LOW_VOLATILITY_FEE;
    }
}
```

### 3.4 beforeSwap Hook Callback

The `beforeSwap` hook is used to update fees before each swap, ensuring they reflect the most recent market conditions:

```solidity
function beforeSwap(address, PoolKey calldata key, IPoolManager.SwapParams calldata, bytes calldata)
    external
    override
    returns (bytes4, BeforeSwapDelta, uint24)
{
    if (block.timestamp >= lastFeeUpdate + FEE_UPDATE_INTERVAL) {
        uint24 newFee = getFee();
        lastFeeUpdate = block.timestamp;
        return (IHooks.beforeSwap.selector, BeforeSwapDeltaLibrary.ZERO_DELTA, newFee | LPFeeLibrary.OVERRIDE_FEE_FLAG);
    }
    return (IHooks.beforeSwap.selector, BeforeSwapDeltaLibrary.ZERO_DELTA, 0);
}
```

This implementation calculates the new fee and returns it with the `OVERRIDE_FEE_FLAG`, allowing for per-swap fee updates without calling `updateDynamicLPFee`.

### 3.5 afterInitialize Hook Callback

The `afterInitialize` hook is used to set the initial fee for the dynamic fee pool:

```solidity
function afterInitialize(address, PoolKey calldata key, uint160, int24, bytes calldata)
    external
    override
    returns (bytes4)
{
    uint24 initialFee = getFee();
    poolManager.updateDynamicLPFee(key, initialFee);
    return IHooks.afterInitialize.selector;
}
```

This ensures that the pool starts with an appropriate fee based on current market conditions.

## Gas Considerations

- Updating fees on each swap introduces additional gas overhead.
- Using `beforeSwap` to override fees is more gas-efficient than calling `updateDynamicLPFee` for fees that change frequently.
- The fee returned by `beforeSwap` is not saved to the PoolManager, making it suitable for per-swap fee adjustments.
- Alternative approaches, such as using an external service to update fees periodically, may be more gas-efficient but less responsive to rapid market changes.

For instance, to optimize gas usage, consider updating fees less frequently:

```solidity
uint256 public constant FEE_UPDATE_INTERVAL = 1 hours;
uint256 public lastFeeUpdate;

function beforeSwap(address, PoolKey calldata key, IPoolManager.SwapParams calldata, bytes calldata)
    external
    override
    returns (bytes4, BeforeSwapDelta, uint24)
{
    if (block.timestamp >= lastFeeUpdate + FEE_UPDATE_INTERVAL) {
        uint24 newFee = getFee(address(0), key);
        lastFeeUpdate = block.timestamp;
        return (BaseHook.beforeSwap.selector, BeforeSwapDeltaLibrary.ZERO_DELTA, newFee | LPFeeLibrary.OVERRIDE_FEE_FLAG);
    }
    return (BaseHook.beforeSwap.selector, BeforeSwapDeltaLibrary.ZERO_DELTA, 0);
}
```

This implementation updates the fee only once per hour, reducing gas costs for frequent swaps.

## Considerations and Best Practices

- The optimal fee depends on at least two factors: **asset volatility** and **volume of uninformed flow.**
- For volatile pairs in systems like Uniswap v3, which don't discriminate between flows, low fee-tier pools are only sensible when uninformed flow is large and asset volatility is relatively low.
- Performance implications of frequent fee updates should be carefully considered.
- Security measures should be implemented to prevent manipulation of fee-setting mechanisms.
- Balance responsiveness with gas costs to optimize for both performance and cost-effectiveness.
