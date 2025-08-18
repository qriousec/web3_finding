# Interest Calculation Bug in _handleValidBorrowRequest

## Summary
The incorrect implementation of borrowing accounting in the `_handleValidBorrowRequest` function will cause an undercharging of interest for borrowers as subsequent cross-chain borrows will not properly account for accrued interest on existing debt, allowing users to bypass some interest payments on previous borrowed amounts.

## Root Cause
The `_handleValidBorrowRequest` function incorrectly adds a new borrow amount to an existing principle without normalizing the existing principle to account for accrued interest. Specifically, the code does:

**Reference**: [CrossChainRouter.sol#L711-L712](https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CrossChainRouter.sol#L711-L712)

This rebases the entire combined debt (old principle + new borrow) to the new borrow index, essentially "erasing" the interest that should have accrued on the old debt.

## Pre-conditions

### Internal Pre-conditions
1. A user must have an existing cross-chain borrow position
2. The borrow index must have increased since the user's initial borrow (i.e., interest has accrued)
3. The user must perform a subsequent cross-chain borrow of the same token

### External Pre-conditions
None - this is an internal issue with interest calculation within the protocol itself.

## Attack Path
1. User borrows token X cross-chain with initial principle P1 and index I1
2. Time passes, and interest accrues, causing the borrow index to increase from I1 to I2
3. User borrows token X again with amount P2
4. The function incorrectly sets the new principle to P1 + P2 and updates the borrow index to I2
5. The correct normalized principle should be (P1 * I2 / I1) + P2, but is instead just P1 + P2
6. User avoids paying interest on the previously borrowed amount (P1)

## Impact
The protocol suffers a loss of interest revenue proportional to the amount of interest that would have accrued on previous borrows. The loss can be calculated as:

```
Lost Interest = P1 * (I2 / I1 - 1)
```

For example, if a user borrowed 100 tokens, the index increased by 20% (from 1e18 to 1.2e18), and then they borrow another 50 tokens, the protocol would lose 20 tokens worth of interest revenue.

## Recommended Mitigation
The function should normalize the old principle by the new borrow index before adding the new borrow amount:

```solidity
// Normalize the old principle to account for interest accrued since the last borrow  
uint256 normalizedOldPrinciple = (userBorrows[index].principle * payload.borrowIndex) / userBorrows[index].borrowIndex;  

// Add the new borrow amount to the normalized principle  
userBorrows[index].principle = normalizedOldPrinciple + payload.amount;  
userBorrows[index].borrowIndex = payload.borrowIndex;  
```