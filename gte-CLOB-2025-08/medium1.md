### The logic of `_decreaseFeeShares` is incorrect in `LaunchToken.sol`

### Severity: Medium

### Line of Code

https://github.com/code-423n4/2025-08-gte-perps/blob/f43e1eedb65e7e0327cfaf4d7608a37d85d2fae7/contracts/launchpad/LaunchToken.sol#L147

### Summary

The line 147 (in the function `_decreaseFeeShares`) 

```
if (totalFeeShare == 0 && !unlocked) _endRewards();
```

is incorrect in logic.

### Finding Description

The intention of line 147 is to stop `GTELaunchPadV2Pair` from distributing part of the swap fee to the reward pool once there are no fee shares. However, this should only be done **after** the token is unlocked, which corresponds to the graduation of the curve and the creation of the `GTELaunchPadV2Pair`. Therefore, the current logic that ends rewards while the token is still locked is incorrect.  

In addition, the current design might cause an unintended DoS when `totalFeeShare = 0`. When the `LaunchToken` is still locked, `_endRewards()` eventually calls the `GTELaunchPadV2Pair`, which has not yet been created (the pair is only created once the curve graduates and the token is unlocked).

Function call order:

`LaunchToken._decreaseFeeShares -> LaunchToken._endRewards -> LaunchPad.endRewards -> Distributor.endRewards -> pair.endRewardsAccrual()`


### Impact

- The current implementation does not match the intended design of stopping reward distribution in `GTELaunchPadV2Pair`.  
- May cause a Denial of Service when a user calls the launchpad `sell` function to sell all their `LaunchToken` (see the PoC).  

### Mitigation

```diff
diff --git a/LaunchToken.sol.orig b/LaunchToken.sol
index 67967e4..4f50cc2 100755
--- a/LaunchToken.sol.orig
+++ b/LaunchToken.sol
@@ -144,7 +144,7 @@ contract LaunchToken is ERC20 {
             bondingShare[account] -= amount;
         }
 
-        if (totalFeeShare == 0 && !unlocked) _endRewards();
+        if (totalFeeShare == 0 && unlocked) _endRewards();
 
         ILaunchpad(launchpad).decreaseStake(account, uint96(amount));
     }
```

### PoC

Copy the following code into `test/c4-poc/PoCLaunchPad.t.sol` and run `forge test --match-test test_submissionValidity`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {LaunchpadTestBase} from "./LaunchpadTestBase.sol";
import {ILaunchpad} from "contracts/launchpad/interfaces/ILaunchpad.sol";
import {LaunchToken} from "contracts/launchpad/LaunchToken.sol";

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

            The user bob buys 500_000_000 ether of launchtoken and then immediately sell it, 
            this will trigger endRewards, which in the end calls pair.endRewardsAccrual(), despite 
            the pair has not been created yet, which causes a revert.
        */

        address bob = makeAddr("Bob");
        quoteToken.mint(bob, 10 ether); 
        vm.startPrank(bob);
        quoteToken.approve(address(launchpad), type(uint256).max);
        LaunchToken(token).approve(address(launchpad), type(uint256).max);
        ILaunchpad.BuyData memory buyData = ILaunchpad.BuyData(
            {
                account: bob,
                token: token,
                recipient: bob, 
                amountOutBase: 500_000_000 ether, 
                maxAmountInQuote: type(uint256).max
            }
        );
        launchpad.buy(buyData);
        assertEq(LaunchToken(token).balanceOf(bob), 500_000_000 ether);
        vm.expectRevert();
        launchpad.sell(bob, token, bob, 500_000_000 ether, 0);
        vm.stopPrank();
    }
}
```

Further running `forge test --match-test test_submissionValidity -vvvv`, we can see that the revert indeed occurs during 

`Distributor.endRewards`, which calls the non-existent `pair.endRewardsAccrual()`.

```
│   │   │   ├─ [5569] 0xDc785bf5611c8A6e164EC81771f4d9e60e1426f5::endRewards()
    │   │   │   │   ├─ [5291] Launchpad::endRewards() [delegatecall]
    │   │   │   │   │   ├─ [3221] Distributor::endRewards(0xd370E4bF0dD427FDA9B0c686748615039376e46D)
    │   │   │   │   │   │   └─ ← [Revert] EvmError: Revert
    │   │   │   │   │   └─ ← [Revert] EvmError: Revert
    │   │   │   │   └─ ← [Revert] EvmError: Revert
    │   │   │   └─ ← [Revert] EvmError: Revert
    │   │   └─ ← [Revert] TransferFromFailed()
```