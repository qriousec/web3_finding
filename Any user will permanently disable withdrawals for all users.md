# Any user will permanently disable withdrawals for all users

### Summary

The use of a 16-bit global nonce in `DineroWithdrawRequestManager` (`s_batchNonce`) will cause a permanent denial-of-service for every vault user once the nonce reaches its maximum value of 65 535, because any user can trigger an overflow by initiating the 65 536-th withdrawal.

### Root Cause

* In [`src/withdraws/Dinero.sol`](https://github.com/sherlock-audit/2025-06-notional-exponent/blob/main/notional-v4/src/withdraws/Dinero.sol#L10)  the global counter is declared as `uint16 internal s_batchNonce;` and incremented with `++s_batchNonce` (line 32). Because Solidity ≥0.8 reverts on arithmetic overflow, attempting to increment past `65 535` reverts, halting the `_initiateWithdrawImpl` flow.

### Internal Pre-conditions

1. `s_batchNonce` has already reached `65 535` (i.e. 65 535 prior withdrawal initiations were executed).

### External Pre-conditions

 None.

### Attack Path

1. The attacker interacts with an *approved* vault (only a tiny amount of yield tokens is needed).
2. They repeatedly invoke `initiateWithdraw`, which forwards to `_initiateWithdrawImpl` and increments `s_batchNonce` by 1 each time, until the counter reaches its maximum value of **65 535**.
3. The very next call (the 65 536-th) tries to increment the nonce to **65 536**; the arithmetic-overflow check triggers and the transaction reverts.
4. Because the nonce remains at its terminal value, *every* future call to `initiateWithdraw`—by any user or vault—reverts for the same reason, permanently blocking all withdrawals.

### Impact

All vault users are permanently unable to withdraw their assets. The economic value locked equals the full TVL held by all `DineroWithdrawRequestManager`-managed vaults. No direct monetary gain is obtained by the triggering user, but the protocol suffers total loss of functionality (High / Critical severity).

### PoC

_No response_

### Mitigation

* Expand the nonce width (e.g. use `uint64` or `uint256`) and adjust the bit-packing layout, or
* Allow the nonce to wrap by incrementing in an `unchecked { … }` block while ensuring uniqueness through the batch-ID components.
