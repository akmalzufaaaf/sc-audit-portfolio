## High

### [H-1] Incorrect fee calculation on `TSwapPool::getInputAmountBasedOnOutput` causes protocol to take too many tokens from users, resulting in lost fees

**Description:** The `getInputAmountBasedOnOutput` function is intended to calculated the amount of tokens users need to deposit based on the amount of output token. However, the function miscalculates the result amount. When calculating the amount is scales by 10_000 instead of 1_000.

**Impact:** Users swapping tokens via the `swapExactOutput` function will pay far more tokens than expected for their trades. This becomes particularly risky for users that provide infinite allowance to the `TSwapPool` contract. Moreover, note that the issue is worsened by the fact that the `swapExactOutput` function does not allow users to specify a maximum of input tokens, as is described in another issue in this report.

**Proof of Concept:**

```js
function test_miscalculationInputAmountBasedOnOutput() public {
        vm.startPrank(liquidityProvider);
        weth.approve(address(pool), 100e18);
        poolToken.approve(address(pool), 100e18);
        pool.deposit(100e18, 100e18, 100e18, uint64(block.timestamp));
        vm.stopPrank();

        console.log("WETH before swapping:", weth.balanceOf(user));
        console.log("poolToken before swapping:", poolToken.balanceOf(user));
        
        vm.startPrank(user);
        poolToken.approve(address(pool), 100e18);
        pool.swapExactOutput(poolToken, weth, 1e18, uint64(block.timestamp));
        // because the ratio is 1:1, the poolToken balance of user should be
        // 100 - 1 = 99
        // and weth balance should be: 10 + 1 = 11
        vm.stopPrank();

        console.log("WETH after swapping:", weth.balanceOf(user));
        console.log("poolToken after swapping:", poolToken.balanceOf(user));

        // poolToken balance shouldnt less then 9e18
        assertFalse(poolToken.balanceOf(user) >= 99e18);
    }
```

The balanceOf(poolToken) should be equals to:
100e18 - 1e18 = 99e18
<==> 9.9e18

but from the test above we got 8.986e19, event if pool takes fee it will not less then 9.8e19.

**Recommended Mitigation:** 

```diff
function getInputAmountBasedOnOutput(
        uint256 outputAmount,
        uint256 inputReserves,
        uint256 outputReserves
    )
        public
        pure
        revertIfZero(outputAmount)
        revertIfZero(outputReserves)
        returns (uint256 inputAmount)
    {
        return
-           ((inputReserves * outputAmount) * 10000) /
+           ((inputReserves * outputAmount) * 10000) /
            ((outputReserves - outputAmount) * 997);
    }

## Medium

### [M-1] `TswapPool::deposit` is missing deadline check causing transactions to complete even after the deadline

**Description:** The deposit function accepts a deadline parameter, which according to the documentation is "The deadline for the transaction to be completed by". However, this parameter is never used. As a consequence, operations that add liquidity to the pool might be executed at unexpected times, in market conditions where the deposit rate is unfavorable.

**Impact:** Transactions could be sent when market conditions are unfavorable, even when adding a deadline parameter.

**Proof of Concept:** The deadline parameter is unused

```js
Warning (5667): Unused function parameter. Remove or comment out the variable name to silence this warning.
   --> src/TSwapPool.sol:126:9:
    |
126 |         uint64 deadline 
    |         ^^^^^^^^^^^^^^^
```

**Recommended Mitigation:** Consider making the following changes to the function:

```diff
function deposit(
        uint256 wethToDeposit,
        uint256 minimumLiquidityTokensToMint,
        uint256 maximumPoolTokensToDeposit,
        uint64 deadline 
    )
        external
        revertIfZero(wethToDeposit)
+       revertIfDeadlinePassed(deadline)
        returns (uint256 liquidityTokensToMint)
    {...}
```

### [H-2] Lacks of slippage protection in `TSwapPool::swapExactOutput`, causes users got charged way more expensive than the users expected

**Description:** The `swapExactOutput` function does not include any sort of slippage protection. This function similar to `TSwapPool::swapExactInput`, where the function specifies a `minOutputAmount`, `swapExactOutput` function should specify a `maxInputAmount`.

**Impact:** If market conditions change before the transaction preoceeses, the user could get a much worse swap.

**Proof of Concept:**
1. The price of 1 WETH right now is 1,000 USDC
2. User input `swapExactOutput` looking for 1 WETH
   1. inputToken = USDC
   2. outputToken = WETH
   3. outputAmount = 1
   4. deadline = whatever
3. The function does not offer a maxInput amount
4. As the transaction pending in the mempool, the market changes. Now 1 WETH = 10,000 USDC. 10x more than what user expected
5. The transaction completes, but the user sent the protocol 10,000 USDC instead of 1,000 USDC

**Recommended Mitigation:** We should include `maxInputAmount` so the user only spend up to a specific amount instead of spending whatever the protocol ask.

```diff
    function swapExactOutput(
        IERC20 inputToken,
+       uint256 maxInputAmount,
.
.
.
.
        inputAmount = getInputAmountBasedOnOutput(
            outputAmount,
            inputReserves,
            outputReserves
        );

+        if inputAmount > maxInputAmount {
+            revert()
+        }

        _swap(inputToken, inputAmount, outputToken, outputAmount);
    }
```

### [H-3] `TSwapPool::sellPoolTokens` miscalculates amount of token bought

**Description:** `sellPoolToken` function intended to allow user to sell some amount of poolToken and receive WETH in exchanges. However, the function miscalculate the swapped amount, due to wrong function call. Instead of using `swapExactOutput`, the `swapExactInput` should be the one that get called, because the `sellPoolToken` function takes poolTokenAmount as a parameter, and return wethAmount.

**Impact:** Users will swapped the wrong amount of pool token, which is severe disruption of protocol functionality.

**Proof of Concept:**

1. User A have 100 poolToken
2. User want to sell 10 poolToken via seelPoolToken
   1. poolTokenAmount = 10e18
   2. weth.balanceOf(user) should be = initial + 10e18
3. Because the function called `swapExactOutput` instead of `swapExactInput` the intended functionality is not working properly. The parameter `poolTokenAmount` is used as the outputTokenAmount. So, if user enter 10e18 as an ardument it will swap the amount of WETh as much as 10e18 (poolTokenAmount).
4. poolToken.balanceOf(User) is not: initial - 10e18, but initial - (10e18 WETH in poolToken). let's say 1e18 WETH = 10 poolToken, then it will initial - (100e18). 

```js
    function test_sellPoolTokens() public {
        vm.startPrank(liquidityProvider);
        weth.approve(address(pool), 100e18);
        poolToken.approve(address(pool), 100e18);
        pool.deposit(100e18, 100e18, 100e18, uint64(block.timestamp));
        vm.stopPrank();
        
        vm.startPrank(user);
        poolToken.approve(address(pool), 100e18);
        uint256 returnedWethAmount = pool.sellPoolTokens(1e18);
        vm.stopPrank();

        // 100e18 - 1e18 = 99e18
        // poolToken balance shouldnt less then 99e18
        // but it got 89e18, 10e18 less than expected
        assertFalse(poolToken.balanceOf(user) >= 99e18);
    }
```

**Recommended Mitigation:** Consider changing the implementation to `swapExactInput` instead of `swapExactOutput`.  Note that this would also require to change the `sellPoolTokens` function to accept a new parameter (e.g., `minWethToReceive`) to be passed down to `swapExactInput`.

```diff
    function sellPoolTokens(
        uint256 poolTokenAmount,
+       uint256 minWethToReceive
    ) external returns (uint256 wethAmount) {
        return
-             swapExactOutput(
+             swapExactInput(
                i_poolToken,
                poolTokenAmount,
                i_wethToken,
+               minWethToReceive,
                uint64(block.timestamp)
            );
    }
```

### [S-#] TITLE (Root Cause + Impact)

**Description:** 

**Impact:** 

**Proof of Concept:**

**Recommended Mitigation:** 

## Low

### [L-1] `TSwapPool::LiquidityAdded` event has parameter out of order

**Description** What `LiquidityAdded` event emitted in the `TSwapPool::_addLiquidityMintAndTransfer` function logs values in incorect order. The `poolTokenToDeposit` should be the third parameter, and `wethToDeposit` should be the second. 

**Impact** Event emission is incorrect, leading to off-chain functions potentially malfunctioning.

**Recommended Mitigation**

```diff
-   emit LiquidityAdded(msg.sender, poolTokensToDeposit, wethToDeposit);
+   emit LiquidityAdded(msg.sender, wethToDeposit, poolTokensToDeposit);
```

### [L-2] `TSwapPool::swapExactInput` expected to return the outputAmount, but no code that returned that value. Causing giving misinformation to caller.

**Description:** The `swapExactInput` function is expected to return the actual amount of tokens bought by the caller. However, while it declares the named return value output, it never assigns a value to it, nor uses an explicit return statement.

As a result, the function will always return zero. Consider modifying the function so that it always return the correct amount of tokens bought by the caller.

**Impact:** The return will always be zero, giving incorrect information to the caller.

**Proof of Concept:**

```js
function test_returnedValueOfSwapExactInput() public {
        vm.startPrank(liquidityProvider);
        weth.approve(address(pool), 100e18);
        poolToken.approve(address(pool), 100e18);
        pool.deposit(100e18, 100e18, 100e18, uint64(block.timestamp));
        vm.stopPrank();
        
        vm.startPrank(user);
        poolToken.approve(address(pool), 100e18);
        uint256 returnedValue = pool.swapExactInput(poolToken, 1e18, weth, 0, uint64(block.timestamp));
        vm.stopPrank();

        console.log("Returned Value:", returnedValue);

        assertEq(returnedValue, 0);
    }
```

**Recommended Mitigation:** There is 2 recommendation:
1. Delete returns statement
2. Add return in the code

## Informationals

### [I-] `PoolFactory::PoolFactory__PoolDoesNotExist` is not used and should be removed

```diff
-   error PoolFactory__PoolDoesNotExist(address tokenAddress);
```

### [I-] `PoolFactory::constructor` is lack of zero address checks

```diff
    constructor(address wethToken) {
+       require(wethToken != address(0), "WETH address cannot be zero");
        i_wethToken = wethToken;
    }
```

### [I-] `PoolFactory::liquidityTokenSymbol` should use .symbol() instead of .name()

```diff
-    string memory liquidityTokenSymbol = string.concat("ts", IERC20(tokenAddress).name());
+    string memory liquidityTokenSymbol = string.concat("ts", IERC20(tokenAddress).symbol());
```

### [I-] Lack of Zero Address Checks in `TSwapPool::constructor`

### [I-] Use SCREAMING_SNAKE_CASE for constant naming

**Description:** Use SCREAMING_SNAKE_CASE for naming constant variable in state variable, see `i_wethToken`, `i_poolToken`

### [I-] Consider using "e" for large numbers for readability

**Description:** Consider using "e" for large numbers for readability in `MINIMUM_WETH_LIQUIDITY`

### [I-] Mutable variables should use misedCase, `swapCount` instead of `swap_count`

**Description:** Mutable variables should use misedCase, `swapCount` instead of `swap_count`.


**Description:** Lack of zero address cheks in `TSwapPool::constructor`

**Impact:** Users who makes new pool will be able to passed zero address as a weth or poolToken address.

**Proof of Concept:**

```diff
    constructor(
        address poolToken,
        address wethToken,
        string memory liquidityTokenName,
        string memory liquidityTokenSymbol
    ) ERC20(liquidityTokenName, liquidityTokenSymbol) {
+       require(address(poolToken) != address(0), "Can't passed zero address");
+       require(address(wethToken) != address(0), "Can't passed zero address");
        i_wethToken = IERC20(wethToken);
        i_poolToken = IERC20(poolToken);
    }
```

### [I-] Event is missing `indexed` fields

Index event fields make the field more quickly accessible to off-chain tools that parse events. However, note that each index field costs extra gas during emission, so it's not necessarily best to index the maximum allowed per event (three fields). Each event should use three indexed fields if there are three or more fields, and gas usage is not particularly of concern for the events in question. If there are fewer than three fields, all of the fields should be indexed.

<details><summary>4 Found Instances</summary>


- Found in src/PoolFactory.sol [Line: 36](src/PoolFactory.sol#L36)

    ```solidity
        event PoolCreated(address tokenAddress, address poolAddress);
    ```

- Found in src/TSwapPool.sol [Line: 52](src/TSwapPool.sol#L52)

    ```solidity
        event LiquidityAdded(
    ```

- Found in src/TSwapPool.sol [Line: 57](src/TSwapPool.sol#L57)

    ```solidity
        event LiquidityRemoved(
    ```

- Found in src/TSwapPool.sol [Line: 62](src/TSwapPool.sol#L62)

    ```solidity
        event Swap(
    ```

</details>



## Gas

### [G-1] `poolTokenReserves` is unused variable in `TSwapPool::deposit` and should be removed

**Description:** From the comment in the function, `poolTokenReserves` will be used to calculate `poolTokensToDeposit`, but the protocol already makes a helper function to do the work it is `getPoolTokensToDepositBasedOnWeth`. So, there is no need to calculate it manually. 

**Recommended Mitigation:** 

```diff
    function deposit(
        ...
    ) 
     ...
    {
        ...
        if (totalLiquidityTokenSupply() > 0) {
            uint256 wethReserves = i_wethToken.balanceOf(address(this));
-           uint256 poolTokenReserves = i_poolToken.balanceOf(address(this));

            uint256 poolTokensToDeposit = getPoolTokensToDepositBasedOnWeth(
                wethToDeposit
            );
    }
    ...
```
