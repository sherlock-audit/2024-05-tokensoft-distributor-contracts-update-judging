Acidic Chrome Capybara

medium

# AdvancedDistributorInitializable sets claim data to empty, making claims fail

## Summary

AdvancedDistributorInitializable.claim overrides the passed-in data with new bytes(0). This data is needed by both PerAddressTrancheVestingMerkleDistributor and PerAddressContinuousVestingMerkleDistributor for claims to work. Therefore, claims do not work.

## Vulnerability Detail


## Impact

Claims do not work

Both PerAddressTrancheVestingMerkleDistributor and PerAddressContinuousVestingMerkleDistributor also pass in `new bytes(0)` to this argument, causing the same issue. However, this is a separate issue per my understanding of Sherlock rules; it is in separate code, and will stay broken if the others are fixed. 

## Code Snippet

AdvancedDistributorInitializable calls DistributorInitializable._executeClaim with data=new bytes(0)

https://github.com/sherlock-audit/2024-05-tokensoft-distributor-contracts-update/blob/main/contracts/packages/hardhat/contracts/claim/factory/AdvancedDistributorInitializable.sol#L106-L113

```solidity
    function _executeClaim(address beneficiary, uint256 totalAmount, bytes memory)
        internal
        virtual
        override
        returns (uint256 _claimed)
    {
        _claimed = super._executeClaim(beneficiary, totalAmount, new bytes(0));
        _reconcileVotingPower(beneficiary);
    }
```

 DistributorInitializable._executeClaim  forwards this data to getClaimableAmount, which forwards it to getVestedFraction

https://github.com/sherlock-audit/2024-05-tokensoft-distributor-contracts-update/blob/main/contracts/packages/hardhat/contracts/claim/factory/DistributorInitializable.sol#L75
https://github.com/sherlock-audit/2024-05-tokensoft-distributor-contracts-update/blob/main/contracts/packages/hardhat/contracts/claim/factory/DistributorInitializable.sol#L113

Many implementations of getVestedFraction attempt to decode this data in ways that break if the data is empty.

E.g.:

https://github.com/sherlock-audit/2024-05-tokensoft-distributor-contracts-update/blob/main/contracts/packages/hardhat/contracts/claim/factory/PerAddressContinuousVestingInitializable.sol#L30-L35

```solidity
    function getVestedFraction(
        address beneficiary,
        uint256 time, // time is in seconds past the epoch (e.g. block.timestamp)
        bytes memory data
    ) public view override returns (uint256) {
        (uint256 start, uint256 cliff, uint256 end) = abi.decode(data, (uint256, uint256, uint256));
```

Putting these together, attempting to claim will be a lot like this Chisel section.

```text
➜ bytes memory data = new bytes(0)
➜ (uint256 start, uint256 cliff, uint256 end) = abi.decode(data, (uint256, uint256, uint256));
Traces:
  [805] 0xBd770416a3345F91E4B34576cb804a576fa48EB1::run()
    └─ ← "EvmError: Revert"

⚒️ Chisel Error: Failed to execute REPL contract!
```

## Tool used

Manual Review

## Recommendation

Pass the data parameter along properly
