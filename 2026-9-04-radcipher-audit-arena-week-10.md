## CRITICAL

### [C-1] Signature Replay Attack in `SignedVoucherSale::buyWithVoucher` Causing Users Can Use the Same Voucher Repeatedly, Possibly Making Protocol Loss Lots of Liquidity

**Description:** There is no mechanism to track used voucher in `SignedVoucherSale::buyWithVoucher`, makes the user to used it again and again. Causing a big liquidity loss to the protocol. Protocol can get both, regular and cross-chain replay attack.

```js
    function buyWithVoucher(
            uint256 discountedPrice,
            uint256 expiry, 
            uint8 v,
            bytes32 r,
            bytes32 s
        ) external payable { 
            require(block.timestamp <= expiry, "Expired");
            require(discountedPrice < fullPrice, "No discount");
            require(msg.value == discountedPrice, "Wrong ETH"); 

            bytes32 voucherHash = keccak256(abi.encodePacked(discountedPrice, expiry)); 
            bytes32 ethSignedHash = keccak256( 
                abi.encodePacked("\x19Ethereum Signed Message:\n32", voucherHash)
            );

            address recovered = ecrecover(ethSignedHash, v, r, s);
            require(recovered == signer, "Bad signature"); 
        }
```

**Impact:** Protocol can get both, regular and cross-chain replay attack. Causing a big loss, both in liquidity and the main functionality of the protocol. 

**Proof of Concept:** 

Place it in test file:
<details>

```js
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {Test, console} from "forge-std/Test.sol";
import {SignedVoucherSale} from "../src/SignedVoucherSale.sol";

contract SignedVoucherSaleTest is Test {
    SignedVoucherSale public sale;

    uint256 internal signerPk;
    address internal signer;

    address[] internal buyers;

    uint256 internal constant FULL_PRICE = 1 ether;

    function setUp() public {
        signerPk = 0xA11CE;
        signer = vm.addr(signerPk);

        sale = new SignedVoucherSale(signer, FULL_PRICE);

        for (uint256 i = 0; i < 5; i++) {
            address buyer = makeAddr(string(abi.encodePacked("buyer", vm.toString(i))));
            buyers.push(buyer);
            vm.deal(buyer, 10 ether);
        }
    }

    function _toEthSignedMessageHash(bytes32 hash) internal pure returns (bytes32) {
        return keccak256(abi.encodePacked("\x19Ethereum Signed Message:\n32", hash));
    }

    function _signVoucher(
        uint256 discountedPrice,
        uint256 expiry,
        uint256 pk
    ) internal pure returns (uint8 v, bytes32 r, bytes32 s) {
        bytes32 voucherHash = keccak256(abi.encodePacked(discountedPrice, expiry));
        bytes32 ethSignedHash = _toEthSignedMessageHash(voucherHash);
        return vm.sign(pk, ethSignedHash);
    }

    function test_buyWithVoucher_success() public {
        uint256 discountedPrice = 0.5 ether;
        uint256 expiry = block.timestamp + 1 days;
        (uint8 v, bytes32 r, bytes32 s) = _signVoucher(discountedPrice, expiry, signerPk);

        uint8 num_of_transaction = 5;

        while (num_of_transaction > 0) {
            vm.prank(buyers[0]);
            sale.buyWithVoucher{value: discountedPrice}(discountedPrice, expiry, v, r, s);
            num_of_transaction--;
        }

        console.log("Contract balance:", address(sale).balance);
        assertEq(address(sale).balance, discountedPrice * 5);
    }
```
</details>

**Recommended Mitigation:** 

There are two main methods to prevent signature replay attacks:

1. Keep a record of used signatures, such as recording the addresses that have already use the voucher in the `addressAlreadyUsedVoucher` mapping, to prevent the reuse of signatures:

``` diff
+   mapping(address => bool) public addressAlreadyUsedVoucher;

    function buyWithVoucher(
            ...
        ) external payable { 
            require(block.timestamp <= expiry, "Expired");
            require(discountedPrice < fullPrice, "No discount");
            require(msg.value == discountedPrice, "Wrong ETH");
+           require(!addressAlreadyUsedVoucher[msg.sender], "User already used the voucher") 

            bytes32 voucherHash = keccak256(abi.encodePacked(discountedPrice, expiry)); 
            bytes32 ethSignedHash = keccak256( 
                abi.encodePacked("\x19Ethereum Signed Message:\n32", voucherHash)
            );

            address recovered = ecrecover(ethSignedHash, v, r, s);
            require(recovered == signer, "Bad signature"); 

+           addressAlreadyUsedVoucher[msg.sender] = true;
        }
```

2. Include `nonce` (incremented for each transaction) and `chainid` (chain ID) in the signed message to prevent both regular replay and cross-chain replay attacks:

``` diff
+   uint258 nonce;

    function buyWithVoucher(
            ...
        ) external payable { 
            require(block.timestamp <= expiry, "Expired");
            require(discountedPrice < fullPrice, "No discount");
            require(msg.value == discountedPrice, "Wrong ETH");

+           bytes32 voucherHash = keccak256(abi.encodePacked(discountedPrice, expiry, nonce, block.chainid)); 
-           bytes32 voucherHash = keccak256(abi.encodePacked(discountedPrice, expiry)); 
            bytes32 ethSignedHash = keccak256( 
                abi.encodePacked("\x19Ethereum Signed Message:\n32", voucherHash)
            );

            address recovered = ecrecover(ethSignedHash, v, r, s);
            require(recovered == signer, "Bad signature"); 

+           nonce++;
        }
```
   
## MEDIUM

### Business Logic Flaws in `SignedVoucherSale::buyWithVoucher`, It Accept a Payment from User but No Items Delivered. Makes Users Loss of Money.

**Description:** There is no items delivered/minted after confirming that user's transaction was valid in `SignedVoucherSale::buyWithVoucher`, this function is just a fraud!

``` diff
    function buyWithVoucher(
            ...
        ) external payable { 
            require(block.timestamp <= expiry, "Expired");
            require(discountedPrice < fullPrice, "No discount");
            require(msg.value == discountedPrice, "Wrong ETH");

            bytes32 voucherHash = keccak256(abi.encodePacked(discountedPrice, expiry)); 
            bytes32 ethSignedHash = keccak256( 
                abi.encodePacked("\x19Ethereum Signed Message:\n32", voucherHash)
            );

            address recovered = ecrecover(ethSignedHash, v, r, s);
            require(recovered == signer, "Bad signature"); 

+           // PLEASE DO SOMETHING!
        }
```

**Impact:** Loss a trust from users makes this protocol will die.

**Proof of Concept:**

1. Deployer sets FULLPRICE = 1 Ether
2. Signer makes a voucher with 0.5 ether discount, return (v, r, s)
3. User make a call to `buyWithVoucher`, send 0.5 ether and the v, r, s
4. Transaction valid, but user doesn't get anything, no token, no nft, nothing.

**Recommended Mitigation:** 

Consider adding minting or transfer after confirming the transaction, but if the intention of the function is just as a helper then marked it as internal to prevent other user/contract to interact with it.

```diff
    function buyWithVoucher(
            ...
+       ) internal payable { 
-       ) external payable { 
            ...
    }
```

