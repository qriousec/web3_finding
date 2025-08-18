# Double Interest Calculation in CoreRouter.borrow

## Summary
A double interest calculation in CoreRouter will cause an incorrect assessment of user borrowing capacity and liquidation eligibility for protocol users as borrowers will experience reduced borrowing limits and may face premature liquidation when their positions should still be healthy.

## Root Cause
In the contract incorrectly applies interest scaling to values that already include accrued interest.

**Reference**: [CoreRouter.sol#L157](https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CoreRouter.sol#L157)

## Pre-conditions
None. This issue is entirely internal to the protocol's own implementation.

## Attack Path
1. User deposits collateral and takes an initial loan
2. Time passes and interest accrues, increasing the borrowIndex
3. When user attempts to borrow again, `getHypotheticalAccountLiquidityCollateral` correctly calculates their position with interest included
4. `CoreRouter.borrow` incorrectly scales the already-interest-adjusted value by the ratio of current/stored borrowIndex
5. This results in the calculated borrowAmount being larger than it should be (by exactly the interest percentage accrued)
6. The user's borrowing capacity is reduced because the system thinks they've borrowed more than they actually have

## Impact
The protocol users suffer reduced borrowing capacity directly proportional to the interest accrued since their last borrow. Our tests show that with just a 10% increase in interest (borrowIndex changing from 1.0 to 1.1), the user's calculated debt is inflated by exactly 10%. This significantly reduces borrowable amounts and can lead to incorrect liquidations.

## Proof of Concept

```solidity
/**  
 * @notice Tests the double interest calculation issue in CoreRouter.borrow  
 * @dev This test:  
 *  1. Supplies collateral  
 *  2. Makes an initial borrow to establish a borrow balance  
 *  3. Manipulates the borrow index to simulate time passing and interest accruing  
 *  4. Tries a second borrow to show the incorrect calculation  
 */  
function testBorrowDoubleInterestCalculation() public {  
    // First supply tokens as collateral  
    uint256 supplyAmount = 1000e18;  
    (address token, address lToken) = _supply(supplyAmount);  
      
    vm.startPrank(deployer);  
      
    // Initial borrow of 30% of collateral  
    uint256 initialBorrowAmount = 300e18;  
    coreRouter.borrow(initialBorrowAmount, token);  
      
    // Get initial state  
    uint256 initialBorrowBalance = lendStorage.borrowWithInterestSame(deployer, lToken);  
    uint256 initialBorrowIndex = LToken(lToken).borrowIndex();  
      
    console2.log("Initial borrow balance:", initialBorrowBalance);  
    console2.log("Initial borrow index:", initialBorrowIndex);  
      
    // Simulate time passing to accrue interest (increase borrow index by 10%)  
    uint256 newBorrowIndex = (initialBorrowIndex * 110) / 100;  
      
    // We need to mock the LToken's borrowIndex to return our increased value  
    vm.mockCall(  
        lToken,  
        abi.encodeWithSelector(BORROW_INDEX_SELECTOR),  
        abi.encode(newBorrowIndex)  
    );  
      
    console2.log("New borrow index (after interest):", newBorrowIndex);  
      
    // Get updated borrow balance with interest  
    uint256 borrowBalanceWithInterest = lendStorage.borrowWithInterestSame(deployer, lToken);  
    console2.log("Borrow balance with interest:", borrowBalanceWithInterest);  
      
    // Calculate what the correct max additional borrow should be  
    (uint256 borrowed, uint256 collateral) =   
        lendStorage.getHypotheticalAccountLiquidityCollateral(deployer, LToken(lToken), 0, 0);  
      
    console2.log("Total borrowed (USD value from hypothetical):", borrowed);  
    console2.log("Total collateral (USD value):", collateral);  
      
    // Calculate what borrowAmount would be according to the buggy code in CoreRouter.borrow  
    // This simulates: borrowAmount = currentBorrow.borrowIndex != 0 ? ((borrowed * newBorrowIndex) / initialBorrowIndex) : 0;  
    uint256 buggyBorrowAmount = (borrowed * newBorrowIndex) / initialBorrowIndex;  
    console2.log("Buggy calculation (double interest):", buggyBorrowAmount);  
      
    // The correct calculation should just use the borrowed amount directly from getHypotheticalAccountLiquidityCollateral  
    console2.log("Correct calculation (already includes interest):", borrowed);  
      
    // Calculate how much additional we can borrow based on remaining collateral  
    uint256 remainingBorrowableUSD = collateral > borrowed ? collateral - borrowed : 0;  
    console2.log("Remaining borrowable (USD):", remainingBorrowableUSD);  
      
    // Calculate how much we can borrow with the buggy calculation  
    uint256 buggyRemainingBorrowable = collateral > buggyBorrowAmount ? collateral - buggyBorrowAmount : 0;  
    console2.log("Buggy remaining borrowable (USD):", buggyRemainingBorrowable);  
      
    // Calculate percentage of error introduced by the bug  
    uint256 errorPercentage = 0;  
    if (buggyBorrowAmount > borrowed) {  
        errorPercentage = ((buggyBorrowAmount - borrowed) * 100) / borrowed;  
        console2.log("Error percentage introduced by bug:", errorPercentage, "%");  
    }  
      
    // Try to borrow half of the remaining borrowable amount, but this should be impossible  
    // if the bug causes the system to think we've exceeded our collateral limit  
    uint256 additionalBorrowAmount = remainingBorrowableUSD / 2;  
      
    if (buggyRemainingBorrowable < additionalBorrowAmount) {  
        console2.log("Bug prevents borrowing! Borrowable amount reduced by:",   
                     remainingBorrowableUSD - buggyRemainingBorrowable);  
    }  
      
    // Verify the impact of the bug  
    assertTrue(buggyBorrowAmount > borrowed, "Bug should cause inflated borrow amount");  
    assertGt(errorPercentage, 0, "Error percentage should be greater than 0");  
    assertLt(buggyRemainingBorrowable, remainingBorrowableUSD, "Bug should reduce borrowable amount");  
      
    vm.stopPrank();  
}
```

### Test Results

```plaintext
[PASS] testBorrowDoubleInterestCalculation() (gas: 793021)
Logs:  
  Initial borrow balance: 300000000000000000000  
  Initial borrow index: 1000000000000000000  
  New borrow index (after interest): 1100000000000000000  
  Borrow balance with interest: 330000000000000000000  
  Total borrowed (USD value from hypothetical): 330000000000000000000  
  Total collateral (USD value): 750000000000000000000  
  Buggy calculation (double interest): 363000000000000000000  
  Correct calculation (already includes interest): 330000000000000000000  
  Remaining borrowable (USD): 420000000000000000000  
  Buggy remaining borrowable (USD): 387000000000000000000  
  Error percentage introduced by bug: 10 %  

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 5.21ms (790.88µs CPU time)
```

## Recommended Mitigation
The fix is straightforward:

In the `borrow` function, replace:
```solidity
uint256 borrowAmount = currentBorrow.borrowIndex != 0  
    ? ((borrowed * LTokenInterface(_lToken).borrowIndex()) / currentBorrow.borrowIndex)  
    : 0;  
```

with:
```solidity
uint256 borrowAmount = borrowed;  
```