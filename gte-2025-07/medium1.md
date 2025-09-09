# FOK fill orders reverts when amount rounded off is left unfilled

### Summary
On the audit page of GTE, it was written:

*On CLOB fill, filled amounts are rounded down to the nearest lot. FOK fill orders should not revert if only the amount rounded off is left unfilled, and the user is not charged for the dust. How we state we handle it may have valid issues to target in case any severe loss or otherwise unintended consequence can be demonstrated.*

However, we found that in some circumstances, FOK fill orders do revert solely because the rounded-off amount is left unfilled.

### Finding description

The issue happens during the following call of functions:

```
postFillOrder -> _processFillAskOrders -> _matchIncomingAsk -> _matchIncomingOrder 
```

For `postFillOrder` function in `CLOB.sol`, for the input args (which is struct PostFillOrderArgs) when `arg.amountIsBase = true`, `arg.amount % lotSizeInBase != 0` and the order type is `FILL_OR_KILL`, even if there is enough amount of matching order in book, the fillorder would revert. This is inconsistent of the desired behavior described in the audit page. 

Note that `arg.amount % lotSizeInBase != 0` is likely to happen, since in the Q&A thread created by BenRai, the sponsor mentioned:

*Lot size will usually be larger than 1*.

### Lines of Code

https://github.com/code-423n4/2025-07-gte-clob/blob/2783108d14010ae50f2a5f65c3a9c94e2e7845b9/contracts/clob/CLOB.sol#L819

### Root Cause

Let's look at code from line 817 - 828 of `CLOB.sol` function `_matchIncomingOrder`,

```solidity
if (amountIsBase) {
            // denominated in base
            matchData.baseDelta = (matchedBase.min(takerOrder.amount) / lotSize) * lotSize;
            matchData.quoteDelta = ds.getQuoteTokenAmount(matchedPrice, matchData.baseDelta);
            matchData.matchedAmount = matchData.baseDelta;
        } else {
            // denominated in quote
            matchData.baseDelta =
                (matchedBase.min(ds.getBaseTokenAmount(matchedPrice, takerOrder.amount)) / lotSize) * lotSize;
            matchData.quoteDelta = ds.getQuoteTokenAmount(matchedPrice, matchData.baseDelta);
            matchData.matchedAmount = matchData.baseDelta != matchedBase ? takerOrder.amount : matchData.quoteDelta;
        }
```

There is logic mistake when `amountisBase = true`. In fact, we can see the logic is different for computing `matchData.matchedAmount` when amountIsBase = true or false. When `amountIsBase = true`, 
```solidity
matchData.matchedAmount = matchData.baseDelta.
```
while when `amountIsBase = false`, 
```solidity
matchData.matchedAmount =  matchData.baseDelta != matchedBase ? takerOrder.amount : matchData.quoteDelta;
```
The logic is correct `when amountIsBase = false`. However, `when amountIsBase = true`, a dust amount of `takerOrder.amount % lotSizeInBase` remains unmatched, which causes `newOrder.amount > 0` and results in a revert due to line `CLOB.sol` line 469 in function `_processFillAskOrder`:

`if (args.fillOrderType == FillOrderType.FILL_OR_KILL && newOrder.amount > 0) revert FOKOrderNotFilled();`

### Impact

The FOK order will revert when `arg.amountIsBase = true` and `arg.amount % lotSizeInBase != 0` due to a dust amount, even when there is enough matching volume in the book, causing unnecessary gas costs for the client.

### Mitigation

I propose the following as mitigation. 


```diff
@@ -818,7 +818,7 @@ contract CLOB is ICLOB, Ownable2StepUpgradeable {
            // denominated in base
            matchData.baseDelta = (matchedBase.min(takerOrder.amount) / lotSize) * lotSize;
            matchData.quoteDelta = ds.getQuoteTokenAmount(matchedPrice, matchData.baseDelta);
-            matchData.matchedAmount = matchData.baseDelta;
+            matchData.matchedAmount = matchData.baseDelta != matchedBase ? takerOrder.amount : matchData.baseDelta;
        } else {
            // denominated in quote
            matchData.baseDelta =
```

# PoC

### Description

The purpose of this PoC is to demonstrate that even when there is enough matching amount in the book, a FOK fill order will revert if `amountIsBase = true`, and `amount % lotSizeInBase != 0`.

To further confirm that the revert is due to the dust amount, one can modify the line
`uint256 baseTokenAmount = 1 ether + 1;` in the PoC to either `uint256 baseTokenAmount = 1 ether;`
or `uint256 baseTokenAmount = 1 ether + 10;`. In both cases, the `FOKOrderNotFilled()` revert no longer occurs.

### Instruction to run this PoC

1. Modify `lotSizeInBase` in `PoCTestBase.t.sol` on line 174 from `1`  to `10`. 

```diff
diff --git a/test/c4-poc/PoCTestBase.t.sol.orig b/test/c4-poc/PoCTestBase.t.sol
index 7e0734e..65152b6 100644
--- a/test/c4-poc/PoCTestBase.t.sol.orig
+++ b/test/c4-poc/PoCTestBase.t.sol
@@ -171,7 +171,7 @@ contract PoCTestBase is CLOBTestBase, DeployPermit2 {
             maxLimitsPerTx: 20,
             minLimitOrderAmountInBase: 0.005 ether,
             tickSize: 0.0001 ether,
-            lotSizeInBase: 1
+            lotSizeInBase: 10
         });

         // Create the market using the factory
```

2. Run the following PoC using

forge test --match-test submissionValidity

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {ICLOB} from "contracts/clob/ICLOB.sol";
import {PoCTestBase} from "./PoCTestBase.t.sol";
import {Side} from "contracts/clob/types/Order.sol";

contract PoC is PoCTestBase {
    function test_submissionValidity() external {
        uint256 quoteTokenAmount = 10 ether;
        uint256 baseTokenAmount = 1 ether + 1;
        uint256 price = 1 ether;
        ICLOB.PostLimitOrderArgs memory argsMaker = ICLOB.PostLimitOrderArgs({
            amountInBase: quoteTokenAmount,
            clientOrderId: 0,
            price: price,
            cancelTimestamp: uint32(block.timestamp + 1 days),
            side: Side.BUY,
            limitOrderType: ICLOB.LimitOrderType.POST_ONLY
        });
        ICLOB.PostFillOrderArgs memory argsTaker = ICLOB.PostFillOrderArgs({
            amount: baseTokenAmount,
            priceLimit: price,
            side: Side.SELL,
            amountIsBase: true,
            fillOrderType: ICLOB.FillOrderType.FILL_OR_KILL
        });
        
        quoteToken = tokenA; 
        baseToken = tokenB; 
        quoteToken.mint(julien, quoteTokenAmount);
        vm.startPrank(julien); 
        quoteToken.approve(address(accountManager), quoteTokenAmount);
        accountManager.deposit(julien, address(quoteToken), quoteTokenAmount);
        ICLOB(abCLOB).postLimitOrder(julien, argsMaker);
        vm.stopPrank();

        baseToken.mint(rite, baseTokenAmount);
        vm.startPrank(rite);
        baseToken.approve(address(accountManager), baseTokenAmount);
        accountManager.deposit(rite, address(baseToken), baseTokenAmount);
        vm.expectRevert("FOKOrderNotFilled()");
        ICLOB(abCLOB).postFillOrder(rite, argsTaker);
        vm.stopPrank();
    }
}
```