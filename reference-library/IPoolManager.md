# IPoolManager

The `IPoolManager` interface defines the main methods for interacting with the Uniswap V4 pool manager contract. It exposes the core liquidity management and swap functionality.

## Methods

### initialize

```solidity
function initialize(PoolKey memory key, uint160 sqrtPriceX96, bytes calldata hookData)
    external
    returns (int24 tick);
```

Initializes the state for a given pool ID with the specified parameters.

| Param Name    | Type      | Description                                     |
|---------------|-----------|--------------------------------------------------|
| key           | PoolKey   | The key defining the pool to initialize          |
| sqrtPriceX96  | uint160   | The initial sqrt price of the pool as a Q64.96 value |
| hookData      | bytes     | Any additional data to pass to our hook contract on the before/after initialization hooks    |

Returns the initial tick value of the pool.

### unlock

```solidity
function unlock(bytes calldata data) external returns (bytes memory);
```

Provides a single entry point for all pool operations. The provided data is passed to the callback for execution.

| Param Name | Type  | Description                                                                         |
|------------|-------|--------------------------------------------------------------------------------------|
| data       | bytes | Any data to pass to the callback via `IUnlockCallback(msg.sender).unlockCallback(data)` |

Returns the data returned by the callback.

### modifyLiquidity

```solidity
function modifyLiquidity(
    PoolKey memory key,
    ModifyLiquidityParams memory params,
    bytes calldata hookData
) external returns (BalanceDelta, BalanceDelta);
```

Modifies the liquidity for the given pool. Can be used to add, remove, or update a liquidity position.

| Param Name | Type                  | Description                                     |
|------------|------------------------|--------------------------------------------------|
| key        | PoolKey               | The key of the pool to modify liquidity in       |
| params     | ModifyLiquidityParams | The parameters for modifying the liquidity position |
| hookData   | bytes                 | Any data to pass to a hook contract on the before/add liquidity hooks              |

Returns the balance delta for the caller (total of principal and fees) and the fee delta generated in the liquidity range.

### swap

```solidity
function swap(PoolKey memory key, SwapParams memory params, bytes calldata hookData)
    external
    returns (BalanceDelta);
```

Executes a swap against the given pool using the provided parameters.

| Param Name | Type       | Description                             |
|------------|------------|-----------------------------------------|
| key        | PoolKey    | The key of the pool to swap in          |
| params     | SwapParams | The parameters for executing the swap   |
| hookData   | bytes      | Any data to pass to a hook contract on the before/afterSwap hooks     |

Returns the balance delta for the address initiating the swap. Note swapping on low liquidity pools may cause unexpected amounts. Interacting with certain hooks may also alter the swap amounts.

### donate

```solidity
function donate(PoolKey memory key, uint256 amount0, uint256 amount1, bytes calldata hookData)
    external
    returns (BalanceDelta);
```

Donates the specified currency amounts to the pool.

| Param Name | Type     | Description                         |
|------------|----------|-------------------------------------|
| key        | PoolKey  | The key of the pool to donate to    |
| amount0    | uint256  | The amount of token0 to donate      |
| amount1    | uint256  | The amount of token1 to donate      |
| hookData   | bytes    | Any data to pass to a hook contract  on the before/afterDonate hooks|

Returns the balance delta representing the donated amounts.
