# Any EVM Address Can Steal Cross-Chain Refunds from Legitimate Recipients

## Summary
A missing receiver check in `GatewayTransferNative.claimRefund` will cause unauthorised asset withdrawals for users expecting refunds for non-EVM chains (Bitcoin, Solana, ...), as any attacker will call `claimRefund` with the leaked `externalId` and bypass the access control.

## Root Cause
In `GatewayTransferNative.sol` the `claimRefund` function sets an incorrect access control check.

**Reference**: [GatewayTransferNative.sol#L664-L668](https://github.com/sherlock-audit/2025-05-dodo-cross-chain-dex/blob/main/omni-chain-contracts/contracts/GatewayTransferNative.sol#L664-L668)

For refunds where `walletAddress` is not 20 bytes (all non-EVM destinations), `receiver` remains `msg.sender`, so the `require` statement always evaluates to true. The contract therefore never verifies that the caller is the rightful owner of the refund.

## Pre-conditions

### Internal Pre-conditions
1. A cross-chain transfer to a non-EVM chain reverts or aborts, causing GatewayTransferNative to store a RefundInfo record and emit `EddyCrossChainRefund` or `EddyCrossChainAbort` with the `externalId`
2. The refund amount `refundInfo.amount` is greater than 0 (any positive balance)

### External Pre-conditions
1. An observer (attacker) monitors events on any of the deployed chains (ZetaChain, Ethereum, Polygon, Arbitrum, Base, BNB Chain) and learns the emitted `externalId`
2. Network conditions allow the attacker to submit a transaction before the intended recipient or a whitelisted bot does (typical)

## Attack Path
1. Attacker watches for `EddyCrossChainRefund` / `Abort` events where `walletAddress.length ≠ 20`
2. Attacker calls `claimRefund(externalId)` on the same chain
3. The `require` passes (`bots[msg.sender] || true`)
4. Contract executes `safeTransfer(refundInfo.token, msg.sender, refundInfo.amount)` and deletes the record
5. Legitimate recipient can no longer claim; funds are permanently lost

## Impact
Affected users permanently lose 100% of every non-EVM refund held by the contract. The attacker gains the full refund value; loss magnitude equals total pending refunds (potentially millions, depending on volume).

## Recommended Mitigation
Upgrade all deployments to check that:

```solidity
require(  
    bots[msg.sender] ||  
    (refundInfo.walletAddress.length == 20 &&  
     msg.sender == address(uint160(bytes20(refundInfo.walletAddress))))  
, "INVALID_CALLER");  
```

Or store an explicit `allowedClaimer` when creating `RefundInfo` and compare to `msg.sender`.