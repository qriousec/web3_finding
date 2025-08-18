# Stale-rate Payout Error in CoreRouter.redeem()

## Summary
Taking a pre-redeem exchange-rate snapshot will cause systematic under-payment of underlying tokens for any user who redeems lTokens, as the CoreRouter will first redeem with LToken at a fresh (higher) rate but then pay the user with an out-of-date (lower) rate.

## Root Cause
In `CoreRouter.sol`: the router reads `exchangeRateStored()` before calling `LToken.redeem()`, derives the expected underlying amount, and immediately uses that value to `transfer()` funds to the user.

**Reference**: [CoreRouter.sol#L118](https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CoreRouter.sol#L118)

In `LToken.sol` (`redeemInternal → redeemFresh`) the `redeem()` path begins with `accrueInterest();` and recalculates the exchange rate, so the amount of underlying returned to CoreRouter is `_amount × rate_after`, which is almost always **greater** than `expectedUnderlying`.

## Pre-conditions

### Internal Pre-conditions
1. A holder calls `CoreRouter.redeem(redeemTokens, lToken)` with `redeemTokens > 0`
2. `LToken.redeem()` performs `accrueInterest()` (always true) so `exchangeRate_after > exchangeRate_before`
3. CoreRouter pays the user based on the stale `exchangeRate_before`

### External Pre-conditions
N/A

## Attack Path
1. User submits a redeem for T lTokens through CoreRouter
2. `LToken.redeem()` accrues interest and transfers `redeemAmount = T × rate_after / 1e18` to CoreRouter
3. CoreRouter pays the user `expectedUnderlying = T × rate_before / 1e18`
4. Difference `D = redeemAmount – expectedUnderlying` accumulates inside CoreRouter; the user is short-changed by D

## Impact
Over many redemptions this silently transfers all interest-accrual delta from users to the protocol (or, in a rare negative-rate swing, leaves the protocol under-collateralised).

## Recommended Mitigation
Compute `exchangeRateCurrent()` after `redeem()` and recalculate the payout amount based on the current rate rather than the stored rate.