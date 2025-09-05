# Any user will block multi-token withdrawals for WETH-based vault depositors on CurveConvex2Token.unstakeAndExitPool

### Summary

The failure to update exitBalances after wrapping native ETH into WETH inside `CurveConvexLib.unstakeAndExitPool` will cause a complete block of multi-token redemptions for vault depositors. Any user who attempts a multi-token redemption on a WETH-based vault (whose underlying Curve pool uses native ETH) will hit a deterministic revert and be unable to withdraw their funds.



### Root Cause

* In [`CurveConvex2Token.sol`](https://github.com/sherlock-audit/2025-06-notional-exponent/blob/main/notional-v4/src/single-sided-lp/CurveConvex2Token.sol#L205) the contract wraps withdrawn ETH into WETH but **does not zero-out the corresponding element in `exitBalances`**:
  ```solidity
  if (ASSET == address(WETH)) {
      if (TOKEN_1 == ETH_ADDRESS) {
          WETH.deposit{value: exitBalances[0]}();
          // exitBalances[0] should be set to 0 here
      } else if (TOKEN_2 == ETH_ADDRESS) {
          WETH.deposit{value: exitBalances[1]}();
          // exitBalances[1] should be set to 0 here
      }
  }
```

The stale array tells upstream logic that the vault still holds ETH, leading to a trade of nonexistent funds.

### Internal Pre-conditions

A vault is deployed whose primary asset is WETH while the Curve pool holds native ETH (e.g. stETH/ETH).
A user possesses yield-token shares and calls redeem(...) requesting a multi-token exit (i.e. redemptionTrades.length > 0).



### External Pre-conditions

Normal network operation;


### Attack Path

1. User calls redeem() with multi-token exit parameters. _redeemShares → CurveConvexLib.unstakeAndExitPool withdraws N ETH, sets exitBalances = [N, 0].
2. Function wraps N ETH into N WETH but leaves exitBalances unchanged.
3. Control returns; _executeRedemptionTrades sees a non-zero ETH balance and tries to sell N ETH.
4.Contract holds zero ETH, so the external trade reverts; the entire redemption transaction fails.
5.The user’s withdrawal remains unfulfilled; funds stay locked in the vault.



### Impact

Effect: Deposit tokens are unredeemable via multi-token path; users’ funds are effectively frozen until a contract upgrade or emergency withdrawal is executed.


### PoC

_No response_

### Mitigation

After each `WETH.deposit{value: ...}` set the corresponding `exitBalances[index] = 0`.  
