# NoOp hooks

One feature enabled by custom accounting​​​​‌ NoOp swap. This feature allows hook developers to change the flow of the swap. 


This means you can replace Uniswap’s internal core logic for how to handle swaps. For example, if the user said they want to sell 1 ETH for USDC, you can take 0.5 ETH of that 1 ETH and route it through separate logic (i.e. swap it using a custom curve), while letting the other 0.5 pass through normally.

Note: NoOp can depend on custom logic and does not have particular restrictions. This means hook developers could do potentially harmful things with it (like taking all swap amounts for themselves, charging extra fees, etc.). So the hook code should be examined before using it.

To enable this developers should use the `BEFORE_SWAP_RETURNS_DELTA_FLAG` or/and `AFTER_SWAP_RETURNS_DELTA_FLAG` in their hook permissions to noop swap. To noop liquidity provision the `AFTER_ADD_LIQUIDITY_RETURNS_DELTA_FLAG` or/and `AFTER_REMOVE_LIQUIDITY_RETURNS_DELTA_FLAG` are also available.

```solidity HookCSMM.sol
import {BaseHook} from "v4-periphery/BaseHook.sol";
// ...

contract HookCSMM is BaseHook {
    // ...

    function getHookPermissions() public pure virtual override returns (Hooks.Permissions memory) {
        return Hooks.Permissions({
            beforeInitialize: false,
            afterInitialize: false,
            beforeAddLiquidity: false,
            afterAddLiquidity: false,
            beforeRemoveLiquidity: false,
            afterRemoveLiquidity: false,
            beforeSwap: false,
            afterSwap: true,
            beforeDonate: false,
            afterDonate: false,
            beforeSwapReturnDelta: true,
            afterSwapReturnDelta: false,
            afterAddLiquidityReturnDelta: false,
            afterRemoveLiquidityReturnDelta: false
        });
    }

    // ...
}
```


## beforeSwap

If the swap amount is designed to go through the separate logic *beforeSwap* **must** return `BeforeSwapDelta`. This is the deltas sorted by *specifiedCurrency* and *unspecifiedCurrency*. The developer should provide the amount of currencies witch need to be omitted from the swap.

In this example below, all amounts are going along CSMM curve (i.e. `amountInput` = `amountOutput`), so we want to take control of the whole `params.amountSpecified`.

The funds' movements are as follows:

1. The caller (swap router)​​​​ transfers funds to the *PoolManager*
2. From the eyes of the *PoolManager* it now has debit which should be taken
3. Hook claims this debit by minting the token
4. Then it transfers the equivalent amounts of opposite currency to *PoolManager* by forming a new debit
5. Now the swap router could mint/transfer tokens from the *PoolManager* back to the user zeroing out the deltas


```solidity HookCSMM.sol
contract HookCSMM is BaseHook {
     // ...

    function beforeSwap(
        address,
        PoolKey calldata key,
        IPoolManager.SwapParams calldata params,
        bytes calldata
    ) external override returns (bytes4, BeforeSwapDelta) {
        uint256 amountInOutPositive = params.amountSpecified > 0
            ? uint256(params.amountSpecified)
            : uint256(-params.amountSpecified);

        BeforeSwapDelta beforeSwapDelta = toBeforeSwapDelta(
            int128(-params.amountSpecified),
            int128(params.amountSpecified)
        );

        if (params.zeroForOne) {
            // If user is selling Token 0 and buying Token 1

            // They will be sending Token 0 to the PM, creating a debit of Token 0 in the PM
            // We will take claim tokens for that Token 0 from the PM and keep it in the hook
            // and create an equivalent credit for that Token 0 since it is ours!
            key.currency0.take(
                poolManager,
                address(this),
                amountInOutPositive,
                true
            );

            // They will be receiving Token 1 from the PM, creating a credit of Token 1 in the PM
            // We will burn claim tokens for Token 1 from the hook so PM can pay the user
            // and create an equivalent debit for Token 1 since it is ours!
            key.currency1.settle(
                poolManager,
                address(this),
                amountInOutPositive,
                true
            );
        } else {
            key.currency0.settle(
                poolManager,
                address(this),
                amountInOutPositive,
                true
            );
            key.currency1.take(
                poolManager,
                address(this),
                amountInOutPositive,
                true
            );
        }

        return (this.beforeSwap.selector, beforeSwapDelta);
    }
}
```

## Testing

To verify what the swap went along the CSMM curve developer should perform the swap and verify:
- the swapper balance of tokenB increased by the same amount as the balance of tokenA decreased
- the hook balance of tokenA increased by the same amount the balance of tokenB decreased


```solidity CSMMTest.t.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Test.sol";
import {IHooks} from "v4-core/interfaces/IHooks.sol";
import {Hooks} from "v4-core/libraries/Hooks.sol";
import {TickMath} from "v4-core/libraries/TickMath.sol";
import {IPoolManager} from "v4-core/interfaces/IPoolManager.sol";
import {PoolKey} from "v4-core/types/PoolKey.sol";
import {BalanceDelta} from "v4-core/types/BalanceDelta.sol";
import {PoolId, PoolIdLibrary} from "v4-core/types/PoolId.sol";
import {CurrencyLibrary, Currency} from "v4-core/types/Currency.sol";
import {PoolSwapTest} from "v4-core/test/PoolSwapTest.sol";
import {Deployers} from "v4-core-test/utils/Deployers.sol";
import {CSMM} from "../src/CSMM.sol";
import {IERC20Minimal} from "v4-core/interfaces/external/IERC20Minimal.sol";

contract CSMMTest is Test, Deployers {
    // ...

    function test_swap_exactInput_zeroForOne() public {
        PoolSwapTest.TestSettings memory settings = PoolSwapTest.TestSettings({
            takeClaims: false,
            settleUsingBurn: false
        });

        // Swap exact input 100 Token A
        uint balanceOfTokenABefore = key.currency0.balanceOfSelf();
        uint balanceOfTokenBBefore = key.currency1.balanceOfSelf();
        swapRouter.swap(
            key,
            IPoolManager.SwapParams({
                zeroForOne: true,
                amountSpecified: -100e18,
                sqrtPriceLimitX96: TickMath.MIN_SQRT_PRICE + 1
            }),
            settings,
            ZERO_BYTES
        );
        uint balanceOfTokenAAfter = key.currency0.balanceOfSelf();
        uint balanceOfTokenBAfter = key.currency1.balanceOfSelf();

        assertEq(balanceOfTokenBAfter - balanceOfTokenBBefore, 100e18);
        assertEq(balanceOfTokenABefore - balanceOfTokenAAfter, 100e18);
    }
}
```
