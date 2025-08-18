# Library Misreads Flag Non-Zero Byte on decompressAccounts

## Summary
An incorrect 32-byte load for a 1-byte boolean in the `decompressAccounts` routine will cause systematic flag flipping and transaction reverts for GatewayCrossChain users as a malicious (or even honest) sender supplies an account list whose `isWritable` byte is `0x00` but is followed by any non-zero byte, making the library mis-read the flag and the Solana program abort.

## Root Cause
In `AccountEncoder.sol` the boolean flag is read with a full-word `mload` but the pointer is advanced by only one byte, so 31 irrelevant bytes are also tested:

**Reference**: [AccountEncoder.sol#L19](https://github.com/sherlock-audit/2025-05-dodo-cross-chain-dex/blob/main/omni-chain-contracts/contracts/libraries/AccountEncoder.sol#L19)

```solidity
function decompressAccounts(bytes memory input) internal pure returns (Account[] memory accounts) {  
    assembly {  
        // ...  
        // Load publicKey (32 B)  
        mstore(acc, mload(ptr))  
        ptr := add(ptr, 32)  

        // BUG: tests 32-byte word, advances only 1 byte  
        mstore(add(acc, 32), iszero(iszero(mload(ptr))))  
        ptr := add(ptr, 1)  
        // ...  
    }  
}
```

Because `mload(ptr)` returns the next 32-byte word, a `0x00` flag immediately followed by any non-zero byte is interpreted as true.

## Pre-conditions

### Internal Pre-conditions
1. Relayer / user must invoke `GatewayCrossChain.onCall` with `decoded.dstChainId == SOLANA_EDDY` so that the contract executes:
```solidity
AccountEncoder.encodeInput(AccountEncoder.decompressAccounts(decoded.accounts), ...)
```
2. Relayer / user sets at least one account entry where byte N (the flag) = `0x00` and byte N + 1 ... N + 31 contain a non-zero byte.

### External Pre-conditions
N/A

## Attack Path
1. Sender prepares `decoded.accounts` with a legitimate read-only account (flag = `0x00`).
2. First byte of the subsequent public key is almost certainly non-zero.
3. GatewayCrossChain calls `decompressAccounts`; the flag is loaded with `mload`, found non-zero, and stored as true.
4. The re-encoded payload is forwarded via `withdrawAndCall` to Solana.
5. Solana runtime / program detects an unexpected writable account and the transaction fails.
6. Gateway triggers `onRevert` or `onAbort`; funds move into the refund queue, user loses gas and fees.
7. A sender can repeat indefinitely, effectively causing a denial-of-service and user fund lock-up for every Solana swap attempt.

## Impact
The affected users lose the origin-chain gas, the platform fee (≈ 0.5 %), and endure delayed access to their assets (locked until manual refund).

## Recommended Mitigation
Fix the load:

```solidity
// Only test the first byte  
mstore(add(acc, 32), byte(0, mload(ptr)))   // or shr(248, mload(ptr))  
ptr := add(ptr, 1)  
```