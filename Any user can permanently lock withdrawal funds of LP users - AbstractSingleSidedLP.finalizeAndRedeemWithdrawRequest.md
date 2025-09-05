# Any user can permanently lock withdrawal funds of LP users - AbstractSingleSidedLP.finalizeAndRedeemWithdrawRequest

## Summary
The missing zero-value check in `AbstractSingleSidedLP.finalizeAndRedeemWithdrawRequest()` will cause a complete loss of access to funds for vault users as any user (including the victim themself) triggers a division-by-zero revert when finalising a withdrawal that involves an underlying token with **no** pending withdrawal request.

## Root Cause
* In [`src/single-sided-lp/AbstractSingleSidedLP.sol`](https://github.com/sherlock-audit/2025-06-notional-exponent/blob/main/notional-v4/src/single-sided-lp/AbstractSingleSidedLP.sol#L392) the division

```solidity
uint256 yieldTokensBurned = uint256(w.yieldTokenAmount) * sharesToRedeem / w.sharesAmount;
```

is executed without verifying that `w.sharesAmount > 0`. When no withdrawal request exists for a given token, `getWithdrawRequest()` returns a zeroed struct, so `w.sharesAmount == 0`, leading to a division-by-zero revert.

## Internal Pre-conditions
1. Vault exits the LP and receives **zero** balance for at least one secondary token.
2. `initiateWithdraw()` therefore **does not** create a withdrawal request for that zero-balance token.
3. After the cooldown period, the user calls `redeemNative()` → `finalizeAndRedeemWithdrawRequest()`.

## External Pre-conditions
* None required – all actions occur within the protocol.

## Attack Path
1. User supplies liquidity and later calls `initiateWithdrawNative()` to redeem all shares.
2. Pool exit returns 0 of Token B (but >0 of Token A); a withdrawal request is created only for Token A.
3. After cooldown, user (or any helper) calls `redeemNative()`.
4. Inside `finalizeAndRedeemWithdrawRequest()` the loop reaches Token B, fetches an empty request and divides by `w.sharesAmount == 0`.
5. Transaction reverts → withdrawal cannot be completed → funds remain permanently escrowed.

## Impact
Vault users cannot finalise their withdrawals, effectively locking **all** their escrowed assets. This breaks core functionality and puts 100 % of affected users’ funds at risk. No attacker profit is required; the denial is automatic.

## POC
N/A
## Mitigation
Skip tokens with no pending request or explicitly require a non-zero `sharesAmount` before performing the division, e.g.

```solidity
(w, ) = manager.getWithdrawRequest(address(this), sharesOwner);
if (w.sharesAmount == 0) continue; // or revert with a clear error
```

