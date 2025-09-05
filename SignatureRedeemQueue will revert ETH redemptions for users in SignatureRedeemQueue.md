# SignatureRedeemQueue will revert ETH redemptions for users in SignatureRedeemQueue

## Summary
The lack of a payable `receive()`/`fallback()` in `SignatureRedeemQueue` will cause **all ETH redemptions to revert**, preventing users from receiving their ETH, because the vault (`ShareModule.callHook`) sends raw ETH to the queue which cannot accept it.

## Root Cause
* In [`SignatureRedeemQueue.sol`](https://github.com/sherlock-audit/2025-07-mellow-flexible-vaults/blob/main/flexible-vaults/src/queues/SignatureRedeemQueue.sol#L6) the contract does **not** implement a payable `receive()` or `fallback()`.
* [`ShareModule.callHook`](src/modules/ShareModule.sol) unconditionally executes `TransferLibrary.sendAssets(asset, queue, assets)` which calls `Address.sendValue(payable(queue), assets)` when `asset == ETH`.
* Without a payable endpoint, that low-level call reverts.

## Internal Pre-conditions
1. A vault is configured with the native ETH pseudo-token `0xEeee…` as an asset.
2. A `SignatureRedeemQueue` instance for that asset is registered.
3. A user obtains a valid signed redeem order and calls `SignatureRedeemQueue.redeem()`.

## External Pre-conditions
* None (the issue is entirely internal to the protocol).

## Attack Path
1. User calls `redeem()` on the queue with an order whose `asset` is ETH.
2. Shares are burned and `ShareModule.callHook(order.requested)` is invoked.
3. `callHook` tries to transfer the requested ETH to the queue.
4. Transfer reverts because the queue cannot receive ETH, causing the whole transaction to revert.

## Impact
* Users **cannot redeem** their shares for ETH. Funds remain locked until the contract is upgraded or migrated.
* High availability impact; potential loss of trust/liquidity. No direct monetary gain for an attacker (griefing vector).

## Mitigation
Add a payable entry point to `SignatureRedeemQueue`, e.g.
```solidity
/// @notice Allows the contract to receive native ETH from the vault
receive() external payable {}
