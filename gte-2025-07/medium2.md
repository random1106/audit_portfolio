# The `PostLimitOrder` function might revert with ZeroCostTrade() even if it is not a zero-cost trade.

### Finding description

Under certain circumstances, `postLimitOrder` function in `CLOB.sol` will revert with ZeroCostTrade(), even if it is not a zero-cost trade. 

### Lines of Code

https://github.com/code-423n4/2025-07-gte-clob/blob/main/contracts/clob/CLOB.sol#L503-L505

https://github.com/code-423n4/2025-07-gte-clob/blob/main/contracts/clob/CLOB.sol#L544-L546

### Root Cause

Let's look at lines `544–546` of the `CLOB.sol` function `_processLimitAskOrder`, which is be called by the function `PostLimitOrder`. 
The issue on lines `503–505` of `CLOB.sol` in the function `_processLimitBidOrder` is similar.


```solidity
if (baseTokenAmountSent != quoteTokenAmountReceived && baseTokenAmountSent & quoteTokenAmountReceived == 0) {
    revert ZeroCostTrade();
}
```

The baseTokenAmountSent (denoted by `x`) and quoteTokenAmountReceived (denoted by `y`) correspond to matching the posted limit order with the overlapping part in the order book.

In short, the code uses the check `(x != y) && (x & y) == 0` to prevent the situation where one of `x` or `y` is zero while the other is not. However, it also incorrectly includes cases where both `x` and `y` are non-zero. More precisely, `x & y` corresponds to a bitwise AND operation, so even if both x and y are non-zero, it is possible that `x & y == 0`, which triggers a revert. For example, as shown in the PoC, if `x = 2 ** 61` and y = `2 ** 62`, this causes a ZeroCostTrade() revert, even though this is NOT a zero-cost trade.
### Impact

The post limit order will revert even though it should be successfully matched and posted. This causes a valid limit order post to revert, resulting in extra gas costs for the client.

### Mitigation

We shall use `(x == 0) != (y == 0)` to represent the situation where one of `x` or `y` is zero and the other is not. Therefore, we propose the following mitigation for `CLOB.sol`.
 
```diff
@@ -500,7 +500,7 @@ contract CLOB is ICLOB, Ownable2StepUpgradeable {

         if (postAmount + quoteTokenAmountSent + baseTokenAmountReceived == 0) revert ZeroOrder();

-        if (baseTokenAmountReceived != quoteTokenAmountSent && baseTokenAmountReceived & quoteTokenAmountSent == 0) {
+        if ((baseTokenAmountReceived == 0) != (quoteTokenAmountSent == 0)) {
             revert ZeroCostTrade();
         }

@@ -541,7 +541,7 @@ contract CLOB is ICLOB, Ownable2StepUpgradeable {

         if (postAmount + quoteTokenAmountReceived + baseTokenAmountSent == 0) revert ZeroOrder();

-        if (baseTokenAmountSent != quoteTokenAmountReceived && baseTokenAmountSent & quoteTokenAmountReceived == 0) {
+        if ((baseTokenAmountSent == 0) != (quoteTokenAmountReceived == 0)) {
             revert ZeroCostTrade();
         }
```

# PoC

### Description

The purpose of this PoC is to demonstrate that even when `baseTokenAmountSent` and `quoteTokenAmountReceived` are both non-zero, a post limit order will revert a `zeroCostTrade()` if `baseTokenAmountSent & quoteTokenAmountReceived = 0`.

### Instruction to run this PoC

Step 1 (Optional) add console.log to CLOB.sol for printing 

```diff
diff --git a/contracts/clob/CLOB.sol.orig b/contracts/clob/CLOB.sol
index 2131fcc..d1d69b2 100644
--- a/contracts/clob/CLOB.sol.orig
+++ b/contracts/clob/CLOB.sol
@@ -21,6 +21,8 @@ import {SafeCastLib} from "@solady/utils/SafeCastLib.sol";
 import {FixedPointMathLib} from "@solady/utils/FixedPointMathLib.sol";
 import {Ownable2StepUpgradeable} from "@openzeppelin-contracts-upgradeable/access/Ownable2StepUpgradeable.sol";

+import {console} from "forge-std/Test.sol";
+
 /**
  * @title CLOB
  * Main spot market contract for trading asset pairs on an orderbook
@@ -542,6 +544,8 @@ contract CLOB is ICLOB, Ownable2StepUpgradeable {
         if (postAmount + quoteTokenAmountReceived + baseTokenAmountSent == 0) revert ZeroOrder();

         if (baseTokenAmountSent != quoteTokenAmountReceived && baseTokenAmountSent & quoteTokenAmountReceived == 0) {
+            console.log("baseTokenAmountSent", baseTokenAmountSent);
+            console.log("quoteTokenAmountReceived", quoteTokenAmountReceived);
             revert ZeroCostTrade();
         }
```

2. Run the following test/c4-poc/PoC.t.sol using

`forge test --match-test submissionValidity -vv`

```solidity
### Description

The purpose of this PoC is to demonstrate that even when `baseTokenAmountSent` and `quoteTokenAmountReceived` are both non-zero, a post limit order will revert a `zeroCostTrade()`.

### Instruction to run this PoC

Step1: Add console.log to CLOB.sol for printing 

```diff
diff --git a/contracts/clob/CLOB.sol.orig b/contracts/clob/CLOB.sol
index 2131fcc..d1d69b2 100644
--- a/contracts/clob/CLOB.sol.orig
+++ b/contracts/clob/CLOB.sol
@@ -21,6 +21,8 @@ import {SafeCastLib} from "@solady/utils/SafeCastLib.sol";
 import {FixedPointMathLib} from "@solady/utils/FixedPointMathLib.sol";
 import {Ownable2StepUpgradeable} from "@openzeppelin-contracts-upgradeable/access/Ownable2StepUpgradeable.sol";

+import {console} from "forge-std/Test.sol";
+
 /**
  * @title CLOB
  * Main spot market contract for trading asset pairs on an orderbook
@@ -542,6 +544,8 @@ contract CLOB is ICLOB, Ownable2StepUpgradeable {
         if (postAmount + quoteTokenAmountReceived + baseTokenAmountSent == 0) revert ZeroOrder();

         if (baseTokenAmountSent != quoteTokenAmountReceived && baseTokenAmountSent & quoteTokenAmountReceived == 0) {
+            console.log("baseTokenAmountSent", baseTokenAmountSent);
+            console.log("quoteTokenAmountReceived", quoteTokenAmountReceived);
             revert ZeroCostTrade();
         }
```

Step2: Run the following test/c4-poc/PoC.t.sol using

`forge test --match-test submissionValidity -vv`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {ICLOB} from "contracts/clob/ICLOB.sol";
import {PoCTestBase} from "./PoCTestBase.t.sol";
import {Side} from "contracts/clob/types/Order.sol";

contract PoC is PoCTestBase {
    function test_submissionValidity() external {
        uint256 price = 2 ether;
        uint256 baseTokenAmountRite = 2305843009213693952; // 2**61
        uint256 baseTokenAmountJulien = 10 ether;
        uint256 quoteTokenAmountJulien = baseTokenAmountJulien * price / 1e18;
        
        ICLOB.PostLimitOrderArgs memory argsMaker = ICLOB.PostLimitOrderArgs({
            amountInBase: baseTokenAmountJulien,
            clientOrderId: 0,
            price: price,
            cancelTimestamp: uint32(block.timestamp + 1 days),
            side: Side.BUY,
            limitOrderType: ICLOB.LimitOrderType.GOOD_TILL_CANCELLED
        });

        ICLOB.PostLimitOrderArgs memory argsTaker = ICLOB.PostLimitOrderArgs({
            amountInBase: baseTokenAmountRite,
            clientOrderId: 0,
            price: price,
            cancelTimestamp: uint32(block.timestamp + 1 days),
            side: Side.SELL,
            limitOrderType: ICLOB.LimitOrderType.GOOD_TILL_CANCELLED
        });
        
        quoteToken = tokenA; 
        baseToken = tokenB; 

        quoteToken.mint(julien, quoteTokenAmountJulien);
        vm.startPrank(julien); 
        quoteToken.approve(address(accountManager), quoteTokenAmountJulien);
        accountManager.deposit(julien, address(quoteToken), quoteTokenAmountJulien);
        ICLOB(abCLOB).postLimitOrder(julien, argsMaker);
        vm.stopPrank();

        baseToken.mint(rite, baseTokenAmountRite);
        vm.startPrank(rite);
        baseToken.approve(address(accountManager), baseTokenAmountRite);
        accountManager.deposit(rite, address(baseToken), baseTokenAmountRite);
        
        vm.expectRevert("ZeroCostTrade()"); 
        ICLOB(abCLOB).postLimitOrder(rite, argsTaker);
        vm.stopPrank();
    }
}
```

We shall see the test passed since a ZeroCostTrade() occurs, however the log prints `2**61` and `2**62`, which is non-zero. 

```
Logs:
  baseTokenAmountSent 2305843009213693952
  quoteTokenAmountReceived 4611686018427387904
```

```

