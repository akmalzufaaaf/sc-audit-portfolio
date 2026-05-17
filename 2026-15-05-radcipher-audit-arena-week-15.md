## Critical

### [C-1] No Amount Checks in `TierMerkleAirdrop::claim` enables users to set it greater than it actually permitted to, causing contract drained

**Description:** No Amount Checks in `TierMerkleAirdrop::claim` Enables users to set it greater than it actually permitted to, causing drain in the contract. After checking proof it didn't checks the `amount` the user's pass as an argument. 

```js
    function claim(uint256 tier, uint256 amount, bytes32[] calldata proof) external {
        require(!claimed[msg.sender], "Already claimed");

        bytes32 leaf = keccak256(abi.encodePacked(tier));
        require(_verify(proof, tierRoot[tier], leaf), "Bad proof");

        claimed[msg.sender] = true;
@>      require(token.transfer(msg.sender, amount), "Transfer failed");
    }
```

**Impact:** Without proper checks on the `amount` users can claim airdrop token as much as they want. Causing protocol drained after only one malicious transaction.

**Proof of Concept:**

1. Alice is eligible to claim airdrop token as tier1, which is 1oo ether.
2. Alice call `claim` with amount greater than it should get, it is 300 ether
3. Because the proof is valid and no checking for the `amount`, instead of getting 100 ether Alice got 300 ether

Place this code in the `TierMarkleAirdrop.t.sol`:

<details>
<summary> PoC </summary>

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {Test, console} from "forge-std/Test.sol";
import {TierMerkleAirdrop, IERC20} from "../src/TierMerkleAirdrop.sol";
import {MockERC20} from "./mocks/MockERC20.sol";

contract TierMerkleAirdropTest is Test {
    MockERC20 internal token;
    TierMerkleAirdrop internal airdrop;

    address internal alice = address(1);
    address internal bob = address(2);
    address internal carol = address(3);

    uint256 internal constant TIER1_AMOUNT = 100 ether;
    uint256 internal constant TIER2_AMOUNT = 200 ether;
    uint256 internal constant TIER3_AMOUNT = 300 ether;

    function setUp() public {
        token = new MockERC20("Test Token", "TST");
        airdrop = new TierMerkleAirdrop(IERC20(address(token)));

        token.mint(address(airdrop), 1_000_000 ether);

        // Single-leaf tree: root == leaf, proof is empty.
        // leaf = keccak256(abi.encodePacked(tier))
        airdrop.setTier(1, keccak256(abi.encodePacked(uint256(1))), TIER1_AMOUNT);
        airdrop.setTier(2, keccak256(abi.encodePacked(uint256(2))), TIER2_AMOUNT);
        airdrop.setTier(3, keccak256(abi.encodePacked(uint256(3))), TIER3_AMOUNT);
    }
    function testClaim_ArbitraryAmount() public {
        bytes32[] memory proof = new bytes32[](0);

        vm.prank(alice);
        airdrop.claim(1, TIER3_AMOUNT, proof);

        assertEq(token.balanceOf(alice), TIER3_AMOUNT);
        assertTrue(airdrop.claimed(alice));
        console.log("Alice claimed tier 1: %e", token.balanceOf(alice));
    }
}
```
</details>

Run `forge test --mt testClaim_ArbitraryAmount`, you will see that instead of getting 100 ether Alice got 300 ether:

```js
[PASS] testClaim_ArbitraryAmount() (gas: 81997)
Logs:
  Alice claimed tier 1: 3e20
```

**Recommended Mitigation:** Adding checks after users verified (it needs whitelist mechanism) or bound the `leaf` with the msg.sender and amount.

### [C-2] `leaf` not bounded to msg.sender and amount enables any user to claim token, causing unauthorized claims

**Description:** `leaf` not bounded to msg.sender and amount enables any user to claim. Also there is no other mechanism to checks if the address called `claim` is authorized, making anyone can claim airdrop token as long as they provide the true `proof`.

```js
    function claim(uint256 tier, uint256 amount, bytes32[] calldata proof) external {
        require(!claimed[msg.sender], "Already claimed");

@>      bytes32 leaf = keccak256(abi.encodePacked(tier));
        require(_verify(proof, tierRoot[tier], leaf), "Bad proof");

        claimed[msg.sender] = true;
        require(token.transfer(msg.sender, amount), "Transfer failed");
    }
```

**Impact:**

1. Unauthorized claim causing assets loss to the protocol.
2. User eligible, can claim greatest tier (Tier3) eventhough they only eligible for low tier (Tier1 or Tier2)

**Proof of Concept:**

1. Alice is not eligible to claim the airdrop token, but she is know the

**Recommended Mitigation:**

## Medium

### [M-1] Centralized control by the `owner` causing `owner` able to change tier info at any time without any limitation

**Description:** There is no limit mechanism, like timelock or multisig in the `setTier` enables `owner` to do any change any time.

Vulnerabilities found in:

```js
    function setTier(uint256 tier, bytes32 root, uint256 defaultAmount) external {
        require(msg.sender == owner, "Not owner");
        tierRoot[tier] = root;
        tierDefaultAmount[tier] = defaultAmount;
    }
```

**Impact:**

1. If owner change the tierRoot without any announcement to users, it makes any claim fail.
2. When `owner` change the `tierDefaultAmount` with higher value, users who have claimed before will be unable to claim the additional value.

### [M-2] Users can only claim once eventhough they eligible for multiple tier, causing dissatisfaction

**Description:** `claimed` used to map the claim status of the users, but it only map based on user's address, so user who eligible for multiple tier can only claim once.

**Impact:** User that eligible for multiple tier can only claim once. If user doesn't know this they might claim the one with lower tier it can actually claim, 

**Proof of Concept:**

1. Alice eligible for Tier1, Tier2, and Tier3, but she doesn't know that she can only claim one tier reward.
2. Alice call `claim` with argument tier = 1. 
3. Claim success. 
4. But when she want to claim for another tier, it failed, cause `claimed` already set Alice have claimed the reward.
5. Alice got angry and post it in X, broke the protocol's reputation.

**Recommended Mitigation:**

Consider changing the `claimed` that include tier mapping:

```diff
-    mapping(address => bool) public claimed;
+    mapping(uint256 => mapping(address => bool)) public claimed;
```

Notes: 
You need to do changes to all the code that interact with `claimed` map, to adapt with this new mechanism.

## Low

### [L-1] No event emitted when state changed make it harder to track on-chain state changes and debug issues

**Description:** Eventhough it doesn't affect the functionality but reduces transparency and traceability. It's common practice to emit events for state changes, so this omission is notable.

**Recommended Mitigation:**

```diff
contract TierMerkleAirdrop {
    .
    .
    .

+   event TierSet(bytes32 newRoot, uint256 newDefaultAmount)
+   event Claimed(address indexed account, uint256 amount, uint256 tier)

    function setTier(uint256 tier, bytes32 root, uint256 defaultAmount) external {
        require(msg.sender == owner, "Not owner");
        tierRoot[tier] = root;
        tierDefaultAmount[tier] = defaultAmount;
+       emit TierSet(root, defaultAmount)
    }

    function claim(uint256 tier, uint256 amount, bytes32[] calldata proof) external {
        require(!claimed[msg.sender], "Already claimed");

        bytes32 leaf = keccak256(abi.encodePacked(tier));
        require(_verify(proof, tierRoot[tier], leaf), "Bad proof");

        claimed[msg.sender] = true;
+       emit Claimed(msg.sender, amount, tier)
        require(token.transfer(msg.sender, amount), "Transfer failed");
    }

    .
    .
    .
}
```
