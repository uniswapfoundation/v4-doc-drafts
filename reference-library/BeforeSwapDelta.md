# BeforeSwapDelta

`BeforeSwapDelta` is a custom type used in Uniswap V4 hook contracts to represent balance changes during swap operations. It is specifically designed to handle the return value of the `beforeSwap` hook and to be compatible with the `afterSwap` hook.

Before explaining `BeforeSwapDelta` in detail, it is worth noting that in the context of Uniswap V4 swaps:

- The "specified" token is the one for which the user specifies an exact input or output amount.
- The "unspecified" token is the counterpart in the swap, whose amount is determined by the pool's pricing mechanism.

## Purpose

The main purpose of `BeforeSwapDelta` is to efficiently encode and decode balance changes for both specified and unspecified tokens in a single `int256` value. This compact representation allows for gas-efficient operations and seamless integration with Uniswap V4's hook system.

`BeforeSwapDelta` is essential for:

- Allowing hooks to modify swap parameters or override default swap behavior
- Enabling information passing between `beforeSwap` and `afterSwap` hooks
- Providing fine-grained control over balance adjustments resulting from swaps
- Optimizing gas usage by packing two `int128` values into a single `int256`

To summarise, `BeforeSwapDelta` is used to ensure that the net balance change for each token is zero after the hook's functionality is executed. This is important for maintaining the integrity of the pool's balances and ensuring that the hooks do not introduce any unexpected or unauthorized balance changes.

## Type Definition

```solidity
type BeforeSwapDelta is int256;
```

The `BeforeSwapDelta` type is an alias for int256, where:

- The upper 128 bits represent the delta in specified tokens
- The lower 128 bits represent the delta in unspecified tokens

## Using Directives

```solidity
using BeforeSwapDeltaLibrary for BeforeSwapDelta global;
using SafeCast for int256;
```

These using directives enable library functions to be used directly on `BeforeSwapDelta` values and provide safe casting operations for int256 values.

## Functions

### toBeforeSwapDelta

```solidity
function toBeforeSwapDelta(int128 deltaSpecified, int128 deltaUnspecified) pure returns (BeforeSwapDelta beforeSwapDelta);
```

Creates a `BeforeSwapDelta` value from two `int128` values representing `deltaSpecified` and `deltaUnspecified`.

| Param Name | Type    | Description                          |
|------------|---------|--------------------------------------|
| deltaSpecified    | int128  | The balance change for the specified token       |
| deltaUnspecified    | int128  | The balance change for the unspecified token      |

Returns the created `BeforeSwapDelta` value.

This function uses bitwise operations in assembly for gas-efficient packing of the two int128 values:

```solidity
assembly ("memory-safe") {
    beforeSwapDelta := or(shl(128, deltaSpecified), and(sub(shl(128, 1), 1), deltaUnspecified))
}
```

## Library Functions

### ZERO_DELTA

```solidity
BeforeSwapDelta public constant ZERO_DELTA = BeforeSwapDelta.wrap(0);
```

A constant representing a zero delta (no balance changes).

### getSpecifiedDelta

```solidity
function getSpecifiedDelta(BeforeSwapDelta delta) internal pure returns (int128 deltaSpecified);
```

Extracts the specified token delta from a `BeforeSwapDelta` value.

| Param Name | Type         | Description                          |
|------------|--------------|--------------------------------------|
| delta      | BeforeSwapDelta | The `BeforeSwapDelta` value       |

Returns the extracted specified token delta as an `int128`.

### getUnspecifiedDelta

```solidity
function getUnspecifiedDelta(BeforeSwapDelta delta) internal pure returns (int128 deltaUnspecified);
```

Extracts the unspecified token delta from a BeforeSwapDelta value.

| Param Name | Type         | Description                          |
|------------|--------------|--------------------------------------|
| delta          | BeforeSwapDelta | The `BeforeSwapDelta`       |

Returns the extracted unspecified token delta as an `int128`.

## Usage in Hooks

When a hook is called during a swap operation, it can perform custom logic and interact with the pool's balances. The `beforeSwap` hook returns a `BeforeSwapDelta` value to indicate any balance changes it introduces. This value can then be forwarded to the `afterSwap` hook, allowing for proper accounting and balance reconciliation.
The `BeforeSwapDelta` can also be forwarded to the `afterAddLiquidity` and `afterRemoveLiquidity` hooks, enabling consistent balance tracking across various pool operations.

## Usage in PoolManager.sol

Here's how `BeforeSwapDelta` is used in the PoolManager contract:

1. In the `swap` function of the PoolManager contract, the `beforeSwap` hook is called:

```solidity
function swap(PoolKey memory key, IPoolManager.SwapParams memory params, bytes calldata hookData)
    external
    override
    onlyWhenUnlocked
    noDelegateCall
    returns (BalanceDelta swapDelta)
{
    // ... (other code)

    BeforeSwapDelta beforeSwapDelta;
    {
        int256 amountToSwap;
        uint24 lpFeeOverride;
        (amountToSwap, beforeSwapDelta, lpFeeOverride) = key.hooks.beforeSwap(key, params, hookData);

        // ... (swap execution)
    }

    // ... (other code)
}
```

The `beforeSwap` hook returns a `BeforeSwapDelta` value along with other parameters.

- This `beforeSwapDelta` can then be passed to the `afterSwap` hook:

```solidity
BalanceDelta hookDelta;
(swapDelta, hookDelta) = key.hooks.afterSwap(key, params, swapDelta, hookData, beforeSwapDelta);
```

The `BeforeSwapDelta` serves several important purposes in this context:

1. **Customization of Swap Behavior:** It allows hooks to modify the swap parameters or even completely override the default swap behavior.
2. **Information Passing:** It provides a way for the `beforeSwap` hook to pass information to the `afterSwap` hook, enabling more complex and stateful hook logic.
3. **Balance Adjustment:** The delta values can be used to adjust the final balance changes resulting from the swap, giving hooks fine-grained control over the swap's outcome.
4. **Gas Optimization:** By packing two `int128` values into a single `int256`, it reduces the number of stack variables and can lead to gas savings.

This usage of `BeforeSwapDelta` is central to the flexibility and extensibility of Uniswap V4's hook system, allowing for custom swap logic while maintaining a standardized interface for all pools.

## Perspective

It's important to note that the `BeforeSwapDelta` is from the perspective of the hook itself, not the user. For example, if a user swaps 1 USDC for 1 USDT:

- User gives 1 USDC: balance0OfUser decreases
- Hook gets 1 USDC: balance0OfHook increases

This perspective is key to correctly interpreting and manipulating the delta values within hook implementations.

## Implementation Details

The `BeforeSwapDelta` type and its associated functions use low-level assembly code for efficient bit manipulation and gas optimization:

- The `toBeforeSwapDelta` function uses bitwise operations (`shl`, `or`, `and`, `sub`) to pack two `int128` values into a single `int256`.
- The `getSpecifiedDelta` function uses the `sar` (shift arithmetic right) operation to extract the upper 128 bits.
- The `getUnspecifiedDelta` function uses the `signextend` operation to extract and sign-extend the lower 128 bits.

The `toBeforeSwapDelta` function combines the specified and unspecified deltas into a single `int256` value using bitwise operations. The `getSpecifiedDelta` and `getUnspecifiedDelta` functions extract the respective deltas using bit shifting and sign extension.

By leveraging this compact representation and efficient arithmetic operations, Uniswap V4 can perform complex balance calculations and updates in a gas-optimized manner, reducing the overall cost of executing pool-related operations.

## Comparison with BalanceDelta

`BeforeSwapDelta` shares a similar structure with `BalanceDelta`, both packing two `int128` values into a single `int256`. However, there are key differences:

- `BalanceDelta` represents amount0 and amount1.
- `BeforeSwapDelta` represents specified and unspecified amounts, which may not directly correspond to token0 and token1, depending on the swap direction.

## Best Practices

When working with `BeforeSwapDelta`, consider the following best practices:

- Always use the provided library functions (`getSpecifiedDelta` and `getUnspecifiedDelta`) to extract delta values.
- Ensure that the signs of the delta values are correct from the hook's perspective.
- Use `SafeCast` when converting between different integer types to prevent overflow/underflow errors.

## Error Handling and Edge Cases

- **Overflow/Underflow:** Ensure that the input int128 values do not exceed their range when packing into BeforeSwapDelta.
- **Zero Values:** ZERO_DELTA represents no balance changes. Be cautious when interpreting zero values in specific contexts.
- **Sign Mismatch:** Ensure that the signs of the delta values correctly represent the intended balance changes from the hook's perspective.

## Example Usage in a Hook

Here's a simple example of how `BeforeSwapDelta` might be used in a `beforeSwap` hook:

```solidity
function beforeSwap(address, PoolKey calldata, IPoolManager.SwapParams calldata params, bytes calldata)
    external
    override
    returns (int256 amountIn, BeforeSwapDelta delta, uint24)
{
    int128 specifiedAmount = params.amountSpecified.toInt128();
    int128 unspecifiedAmount = 0; // Calculated based on your custom logic

    delta = toBeforeSwapDelta(specifiedAmount, unspecifiedAmount);
    amountIn = params.amountSpecified;

    return (amountIn, delta, 0);
}
```
