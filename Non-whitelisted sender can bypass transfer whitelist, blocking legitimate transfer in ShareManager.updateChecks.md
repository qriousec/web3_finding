# Non-whitelisted sender can bypass transfer whitelist, blocking legitimate transfer in ShareManager.updateChecks

## Summary
A logical inversion in `ShareManager.updateChecks` causes unauthorized transfers and blocks legitimate ones: the condition `info.canTransfer || !$.accounts[to].canTransfer` reverts when both parties are whitelisted and allows transfers when the sender is not. Thus, the whitelist is bypassed.

## Root Cause
In [`src/managers/ShareManager.sol`](https://github.com/sherlock-audit/2025-07-mellow-flexible-vaults/blob/main/flexible-vaults/src/managers/ShareManager.sol#L138C13-L142C18) the boolean condition is inverted; it should revert when **either** party is **not** whitelisted, not when at least one **is**.

## Internal Pre-conditions
*None*

## External Pre-conditions
*None*

## Attack Path
1. Attacker (non-whitelisted) calls `transfer(to, amount)` where `to` is whitelisted.
2. In `updateChecks` the expression `false || !true` evaluates to `false`, so no revert.
3. Transfer succeeds, bypassing whitelist.

## Impact
• Whitelist control is ineffective: unauthorized addresses can move tokens.
• Whitelisted users cannot transfer among themselves, disrupting normal operation.

## Mitigation
Change the check to:
```solidity
if (!info.canTransfer || !$.accounts[to].canTransfer) {
    revert TransferNotAllowed(from, to);
}
```
This requires both `from` and `to` to be whitelisted for a transfer to proceed.
"
