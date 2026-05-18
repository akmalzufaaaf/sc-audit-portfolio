## Medium

### [M-1] `Vesting` is used as a parent of `SymmVesting`, so `__vesting_init` should use `onlyInitializing` instead of `initializer`

**Description:**

`Vesting` is an abstract base that `SymmVesting` inherits (`contract SymmVesting is Vesting`). Its initializer is declared with the `initializer` modifier:

```solidity
// token/contracts/vesting/Vesting.sol:76
function __vesting_init(address admin, uint256 _lockedClaimPenalty, address _lockedClaimPenaltyReceiver) public initializer {
```

The derived contract's `initialize` is *also* an `initializer`, and it calls the parent init from inside its body:

```solidity
// token/contracts/vesting/SymmVesting.sol:65-77
function initialize(...) public initializer {
    ...
    __vesting_init(admin, 500000000000000000, _lockedClaimPenaltyReceiver);
    ...
}
```

With OpenZeppelin `Initializable` (v4.x), the `initializer` modifier is a single-use, top-level guard. The first `initializer` (`SymmVesting.initialize`) sets `_initialized = 1` and `_initializing = true`, then runs its body. When it reaches the nested `__vesting_init`, the `initializer` modifier re-evaluates:

- `isTopLevelCall = !_initializing` → `false` (we are already initializing)
- require clause: `(false && _initialized < 1) || (!isContract(address(this)) && _initialized == 1)` → `false || (false && true)` → `false`

so it reverts with `Initializable: contract is already initialized`. A base contract's init function that is invoked from a derived `initializer` must use `onlyInitializing`, which only asserts `_initializing == true` and does not re-claim the one-time lock.

**Impact:** `SymmVesting.initialize` always reverts. The contract can never be initialized, so the proxy is permanently unusable (no admin/roles set, no penalty config, no liquidity functions reachable). This is a deployment-bricking / denial-of-service condition — arguably High severity since the contract is completely non-functional, though no funds are at direct risk because it never becomes operational.

**Proof of Concept:**

1. Deploy the `SymmVesting` implementation behind a proxy.
2. Call `initialize(admin, penaltyReceiver, pool, router, permit2, vault, symm, usdc, symmLp)` with all non-zero addresses.
3. Execution enters `SymmVesting.initialize` (`initializer`: `_initialized = 1`, `_initializing = true`).
4. It calls `__vesting_init(...)`, whose `initializer` modifier sees `_initializing == true` and `_initialized == 1` on a contract address.
5. The `require` in the `initializer` modifier fails → the whole transaction reverts with `Initializable: contract is already initialized`.

Result: no valid argument set can ever initialize `SymmVesting`.

**Recommended Mitigation:**

Change the parent initializer to `onlyInitializing` so it can run as part of the derived contract's single initialization flow. Following the OZ convention, also make it `internal` rather than `public` so it cannot be invoked standalone:

```diff
- function __vesting_init(address admin, uint256 _lockedClaimPenalty, address _lockedClaimPenaltyReceiver) public initializer {
+ function __vesting_init(address admin, uint256 _lockedClaimPenalty, address _lockedClaimPenaltyReceiver) internal onlyInitializing {
```

`onlyInitializing` requires `_initializing == true` and does not touch `_initialized`, so the single top-level `initializer` (`SymmVesting.initialize`) remains the sole one-time entry point while the parent state still gets configured.

## Informational

### [I-1] Using Floating Pragma in Multiple Contract, can introduce bugs from different pragma version

**Description:** 
The project contains many instances of floating pragma. Contracts should be deployed with the same compiler version and flags that they have been tested with thoroughly. Locking the pragma helps to ensure that contracts do not accidentally get deployed using, for example, either an outdated compiler version that might introduce bugs that affect the contract system negatively or a pragma version too new which has not been extensively tested.

**Recommended Mitigation:** 
Consider using specific solidity version instead of open ended like `>=0.8.x` or `^0.8.x`, to more specific versionn like `0.8.x` to avoid any accidental bugs from compiling contract with different/untested solidity version.

Do the changes in the contract with floating pragma, like in `SymmStaking` contract. 

```diff
-   pragma solidity >=0.8.18;
+   pragma solidity 0.8.18;
```