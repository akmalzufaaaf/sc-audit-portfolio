## CRITICAL

### [C-1] `SimplePaymentSplitter::releasable` using Dynamic Value to Determine the Amount Payees Will Get, Causing Inconsistent Result of The Calculation 

**Description:** `SimplePaymentSplitter::releasable` using Dynamic Value (`address(this)`) to determine the amount payees will get. Using `address(this)` to calculate it makes the result differ from time-to-time making payee with the same amount of share can get different amount of Ether.

Besides that, this calculations can makes ether stuck in the contract.

```js
    function releasable(address payee) public view returns (uint256) {
@>      uint256 gross = (address(this).balance * shares[payee]) / totalShares;
        return gross - released[payee];
    }
```

**Impact:** 

1. `payee` with the same amount of share can get different amount of ether.
2. Ether get stuck in contract

**Proof of Concept:** Place this code in the test file
<details>
<summary>Code</summary>

```js
contract SimplePaymentSplitterTest is Test {
    SimplePaymentSplitter splitter;

    address payable internal alice = payable(address(0xA11CE));
    address payable internal bob = payable(address(0xB0B));

    function setUp() public {
        address[] memory payees = new address[](2);
        uint256[] memory shares_ = new uint256[](2);

        payees[0] = alice;
        payees[1] = bob;
        shares_[0] = 1;
        shares_[1] = 1;

        splitter = new SimplePaymentSplitter(payees, shares_);
    }

    function test_misscalculationOfPayment() public {
        vm.deal(address(this), 1 ether);
        (bool sent, ) = address(splitter).call{value: 1 ether}("");
        require(sent, "funding failed");

        assertEq(splitter.releasable(alice), 0.5 ether);
        assertEq(splitter.releasable(bob), 0.5 ether);

        console.log("alice's balance before release: ", alice.balance);
        console.log("bob's balance before release: ", bob.balance);

        splitter.release(alice);
        splitter.release(bob);

        console.log("alice's balance after release: ", alice.balance);
        console.log("bob's balance after release: ", bob.balance);
        console.log("splitter's balance after release: ", address(splitter).balance);

        // Contract Balance: 1 Ether
        // Alice should get: 0.5 Ether
        // Bob should get: 0.5 Ether
        assertEq(alice.balance, 0.5 ether);
        assertEq(bob.balance, 0.5 ether);
    }
```

</details>
You will get this error log:

```js
Ran 1 test for test/SimplePaymentSplitter.t.sol:SimplePaymentSplitterTest
[FAIL: assertion failed: 250000000000000000 != 500000000000000000] test_misscalculationOfPayment() (gas: 182676)
Logs:
  alice's balance before release:  0
  bob's balance before release:  0
  alice's balance after release:  500000000000000000
  bob's balance after release:  250000000000000000
  splitter's balance after release:  250000000000000000
```

**Recommended Mitigation:** Consider adding `totalReceived` to track the total ether contract have received to ensure the consistency of the gross calculation result.

```diff
+   uint256 public totalReceived;

    receive() external payable {
+       totalReceived += msg.value;
    }

    function releasable(address payee) public view returns (uint256) {
+       uint256 gross = (totalReceived * shares[payee]) / totalShares;
-       uint256 gross = (address(this) * shares[payee]) / totalShares;
        return gross - released[payee];
    }
```

