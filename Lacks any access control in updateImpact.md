# Lacks Access Control in updateImpact Function

## Summary
The `updateImpact(uint256 virtualId, uint256 proposalId)` function in `ServiceNft.sol` is declared public but lacks any access control mechanisms (like `onlyOwner` or checking `msg.sender`).

## Finding Description
When `mint()` is legitimately called (by the virtualId's DAO), it calculates `_maturities[proposalId]` for the new service and then calls `updateImpact(virtualId, proposalId)` internally.

Inside this first `updateImpact` call:
1. `prevServiceId = _coreServices[virtualId][_cores[proposalId]]` correctly fetches the ID of the previous core service for that type
2. The `rawImpact` is then calculated as the difference in maturity between the new service (`_maturities[proposalId]`) and this previous service (`_maturities[prevServiceId]`)
3. After the internal `updateImpact` call completes, the `mint()` function updates the state: `_coreServices[virtualId][_cores[proposalId]] = proposalId`. This now registers the newly minted `proposalId` as the current active service for its core type.

## Attack Path
An attacker (any external account) can then call the public `updateImpact(virtualId, proposalId)` function again, using the same `virtualId` and the newly minted `proposalId`.

In this second, attacker-initiated `updateImpact` call:
1. `prevServiceId` will now be equal to `proposalId` itself (due to the state update at the end of `mint()`).
2. The `rawImpact` calculation becomes `(_maturities[proposalId] > _maturities[proposalId]) ? _maturities[proposalId] - _maturities[proposalId] : 0`, which will always result in 0.
3. Consequently, `_impacts[proposalId]` is set to 0. If a `datasetId` is associated, `_impacts[datasetId]` also becomes 0.

## Impact
**Nullification of Impact Scores**: An attacker can set the legitimately calculated impact score of a new core service NFT (and its associated dataset NFT, if any) to zero immediately after it's minted.

## Recommended Mitigation Steps
Making the function `internal` ensures it can only be called from within the `ServiceNft` contract itself (specifically, by the `mint` function) or by contracts that inherit from `ServiceNft`. This directly blocks the external attacker's call.