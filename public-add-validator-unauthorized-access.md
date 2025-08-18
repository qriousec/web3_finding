# Publicly Accessible addValidator Function Allows Unauthorized Validator Registration

## Summary
The `addValidator` function in `AgentNftV2.sol` is publicly accessible without proper access control, allowing unauthorized users to manipulate validator sets.

## Finding Description
The `mint` function calls `_addValidator(virtualId, founder)` with proper access control (`onlyRole(MINTER_ROLE)`). However, a separate `addValidator` function lacks any access control:

```solidity
function addValidator(uint256 virtualId, address validator) public {
    if (isValidator(virtualId, validator)) {
        return;
    }
    _addValidator(virtualId, validator); // Internal function from ValidatorRegistry.sol
    _initValidatorScore(virtualId, validator);
}
```

The function directly calls the internal `_addValidator` function (from the inherited `ValidatorRegistry` contract) and `_initValidatorScore` without verifying if the `msg.sender` has the appropriate permissions to manage validators for the given `virtualId`.

## Impact
The `addValidator` vulnerability provides a direct avenue to attack the governance layer (AgentDAO) of any Virtual Agent by manipulating its validator set, which is managed by `AgentNftV2`.

## Recommended Mitigation Steps
Apply appropriate access control modifier to restrict the `addValidator` function to authorized roles only.