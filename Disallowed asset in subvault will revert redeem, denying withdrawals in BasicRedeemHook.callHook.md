# Disallowed asset in subvault will revert redeem, denying withdrawals in BasicRedeemHook.callHook

## Summary
A missing asset-permission check in `BasicRedeemHook.sol` will cause a denial-of-service for vault users: when `callHook` meets a subvault holding the requested asset that is **not** allowed for that subvault, the subsequent `vault.hookPullAssets` call reverts with `NotAllowedAsset`, aborting the whole redeem transaction.

## Root Cause
* In [`src/hooks/BasicRedeemHook.sol`](https://github.com/sherlock-audit/2025-07-mellow-flexible-vaults/blob/main/flexible-vaults/src/hooks/BasicRedeemHook.sol#L9) the `callHook` loop  blindly invokes `vault.hookPullAssets(subvault, asset, value)` without verifying `riskManager().isAllowedAsset(subvault, asset)`.  
* Inside `_pullAssets` → `RiskManager.modifySubvaultBalance` the asset-permission check fails and reverts.

If a subvault happens to hold a token that is NOT whitelisted for it, riskManager.modifySubvaultBalance reverts, and the whole redeem transaction fails before the loop can reach the other, properly-configured subvaults.

## Internal Pre-conditions
*None*

## External Pre-conditions
*None.*

## Attack Path

1. **Attacker** sends (any amount of) the target `asset` to a subvault where that asset is *not* whitelisted. This requires no special privileges—anyone can transfer ERC-20 tokens to any address.
2. A **legitimate user** triggers a redeem of `asset` from the main vault.
3. The vault's liquid balance is insufficient, so `BasicRedeemHook.callHook` iterates through subvaults.
4. The loop reaches the *polluted* subvault first (set ordering is uncontrollable). It sees the non-zero balance and calls `vault.hookPullAssets`.
5. Inside `_pullAssets`, `riskManager.modifySubvaultBalance` reverts with `NotAllowedAsset` because the asset is disallowed for that subvault.
6. The revert aborts the entire redeem transaction, preventing all users from redeeming until the asset is swept or code is fixed — a denial-of-service against the protocol.

## Impact
Users cannot redeem their assets (service is denied). Funds are locked until configuration or code is fixed. No direct financial gain for an attacker, but it hampers protocol usability and could be exploited for griefing.

## Mitigation
Before pulling from a subvault, ensure the asset is allowed:
```solidity
if (!vault.riskManager().isAllowedAsset(subvault, asset)) {
    continue; // skip disallowed subvaults
}
```
