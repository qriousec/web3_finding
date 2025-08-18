# Cross-Chain Router _checkLiquidationValid Liquidation Validation Bug

## Summary
An incorrect parameter placement in the `_checkLiquidationValid` function will cause an inaccurate assessment of borrower solvency for borrowers during cross-chain liquidations as liquidators may be incorrectly prevented from executing valid liquidations or allowed to execute invalid ones.

## Root Cause
In [CrossChainRouter.sol#L433](https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CrossChainRouter.sol#L433) the parameters in the `getHypotheticalAccountLiquidityCollateral` function call are misplaced, modeling a liquidation as an increase in borrowing rather than a decrease in collateral.

The issue is that `payload.amount` (representing tokens to seize) is passed as the 4th parameter (`borrowAmount`) instead of the 3rd parameter (`redeemTokens`).

## Pre-conditions
The price of collateral asset must drop enough to make the position eligible for liquidation, but not by too much (to highlight the bug's impact).

## Attack Path
1. A borrower supplies collateral on Chain A and borrows assets on Chain B via cross-chain borrowing
2. The price of the collateral asset drops slightly, making the position eligible for liquidation
3. A liquidator calls `liquidateCrossChain` on Chain B to initiate the liquidation
4. The cross-chain message is sent to Chain A where `_checkLiquidationValid` is called to verify if the liquidation is valid
5. Due to the bug, `_checkLiquidationValid` incorrectly models the liquidation as an increase in borrowing rather than a decrease in collateral
6. This leads to an incorrect assessment of the borrower's solvency state

Depending on the specific scenario:
- The function may return `false` for positions that should be liquidated, preventing valid liquidations
- The function may return `true` for positions that should not be liquidated, allowing invalid liquidations

## Impact
Both borrowers and liquidators suffer from uncertainty and unpredictability in the liquidation process:

- Borrowers may have their positions improperly liquidated when they shouldn't be, resulting in unnecessary loss of collateral

## Recommended Mitigation
The fix is to swap the parameters in the call to `getHypotheticalAccountLiquidityCollateral` to correctly model the effect of seizing collateral:

```solidity
function _checkLiquidationValid(LZPayload memory payload) private view returns (bool) {  
    (uint256 borrowed, uint256 collateral) = lendStorage.getHypotheticalAccountLiquidityCollateral(  
        payload.sender, LToken(payable(payload.destlToken)), payload.amount, 0  
    );  
    return borrowed > collateral;  
}