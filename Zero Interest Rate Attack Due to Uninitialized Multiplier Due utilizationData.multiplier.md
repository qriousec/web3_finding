# Zero Interest Rate Attack Due to Uninitialized Multiplier Due utilizationData.multiplier

## Summary
The uninitialized `utilizationData.multiplier` (defaults to 0) will cause a complete loss of interest revenue for lenders as an attacker will frontrun the first `rate()` call by pushing utilization above the kink point, permanently locking the multiplier at zero.

## Root Cause
In [`VaultAdapter.sol:88-89`](https://github.com/sherlock-audit/2025-07-cap/blob/main/cap-contracts/contracts/oracle/libraries/VaultAdapter.sol#L88C42-L88C57) the multiplication operation `utilizationData.multiplier = utilizationData.multiplier * (1e27 + ...) / 1e27` starts with an uninitialized multiplier value of 0. When utilization exceeds the kink point during the first call, this results in `0 * anything = 0`, permanently setting the multiplier to zero. Additionally, the high utilization branch lacks the `minMultiplier` bounds checking that exists in the low utilization branch .
```solidity
    utilizationData.multiplier = utilizationData.multiplier
         * (1e27 + (1e27 * excess / (1e27 - slopes.kink)) * (_elapsed * $.rate / 1e27)) / 1e27;
```
## Internal Pre-conditions
1. No previous call to `rate()` function has been made for the specific (vault, asset) pair

## External Pre-conditions
1. Borrowing demand needs to be high enough to push utilization above the kink point immediately after liquidity provision
2. Market conditions need to maintain high utilization to keep the multiplier permanently at zero

## Attack Path
1. Attacker observes a new vault-asset pair deployment with fresh liquidity
2. Liquidity providers deposit funds into the vault
3. Attacker (or coordinated borrowers) immediately borrows enough to push utilization above `slopes.kink` before any `rate()` call
4. First call to `rate()` function occurs with utilization > kink, triggering the vulnerable branch
5. `utilizationData.multiplier` gets calculated as `0 * (1e27 + excess_calculation) / 1e27 = 0`
6. All subsequent calls to `rate()` return 0 interest rate since `interestRate = (...) * 0 / 1e27 = 0`
7. Borrowers continue to benefit from 0% interest rates while lenders receive no compensation

## Impact
The lenders suffer a complete loss of interest revenue that could amount to millions of dollars depending on the vault size and duration. The attacker gains zero-cost loans for an indefinite period until utilization drops below the kink point or admin intervention occurs. 

## Mitigation
Initialize `utilizationData.multiplier` to `minMultiplier` when it is detected to be zero.
