# Issue M-1: AdvancedDistributorInitializable sets claim data to empty, making claims fail 

Source: https://github.com/sherlock-audit/2024-05-tokensoft-distributor-contracts-update-judging/issues/11 

## Found by 
0xboriskataa, 1337web3, BiasedMerc, Honour, Ironsidesec, Varun\_05, aman, hunter\_w3b, jkoppel, merlin, nfmelendez, robertodf, samuraii77
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



## Discussion

**ArshaanB**

Yes, can confirm, this seems like an issue.

`ContinuousVestingInitializable`, `TrancheVestingInitializable` — the `getVestedFraction()` doesn't care about `data`
`PerAddressContinuousVestingInitializable`, `PerAddressTrancheVestingInitializable` — the `getVestedFraction()` does care about `data`

**sherlock-admin2**

The protocol team fixed this issue in the following PRs/commits:
https://github.com/SoftDAO/contracts/pull/47


# Issue M-2: PerAddressTrancheVestingMerkleDistributor.claim always reverts because it checks the Merkle proof incorrectly 

Source: https://github.com/sherlock-audit/2024-05-tokensoft-distributor-contracts-update-judging/issues/15 

## Found by 
jkoppel, samuraii77
## Summary

Both the `claim` and `initializeDistributionRecord` methods of PerAddressTrancheVestingMerkleDistributor do not include the tranches when checking the Merkle proof (and do not even accept the tranches as a parameter). Both methods therefore always revert, before the function body is even entered.

## Vulnerability Detail

We know from PerAddressTrancheVestingMerkle that the Merkle trees for this kind of distributor include the list of tranches for each address (and also because they have to be; there's no other way to specify the tranches). However,  PerAddressTrancheVestingMerkleDistributor omits the tranches when checking the Merkle proofs. The Merkle proofs therefore always revert.

## Impact

PerAddressTrancheVestingMerkleDistributor is completely inoperable.

## Code Snippet

Notice the lack of tranches here:

https://github.com/sherlock-audit/2024-05-tokensoft-distributor-contracts-update/blob/main/contracts/packages/hardhat/contracts/claim/factory/PerAddressTrancheVestingMerkleDistributor.sol#L42-L58

```solidity
    function initializeDistributionRecord(
        uint256 index, // the beneficiary's index in the merkle root
        address beneficiary, // the address that will receive tokens
        uint256 amount, // the total claimable by this beneficiary
        bytes32[] calldata merkleProof
    ) external validMerkleProof(keccak256(abi.encodePacked(index, beneficiary, amount)), merkleProof) {
        _initializeDistributionRecord(beneficiary, amount);
    }

    function claim(
        uint256 index, // the beneficiary's index in the merkle root
        address beneficiary, // the address that will receive tokens
        uint256 totalAmount, // the total claimable by this beneficiary
        bytes32[] calldata merkleProof
    )
        external
        validMerkleProof(keccak256(abi.encodePacked(index, beneficiary, totalAmount)), merkleProof)
```

Contrast the code in PerAddressTrancheVestingMerkle, which checks the Merkle proof like this:

https://github.com/sherlock-audit/2024-05-tokensoft-distributor-contracts-update/blob/main/contracts/packages/hardhat/contracts/claim/PerAddressTrancheVestingMerkle.sol#L45

```solidity
    validMerkleProof(keccak256(abi.encodePacked(index, beneficiary, amount, abi.encode(tranches))), merkleProof)
```

## Tool used

Manual Review

## Recommendation

Add the tranches parameter to both methods and check in the Merkle proof



## Discussion

**ArshaanB**

Yes, `tranches` should be passed to `validMerkleProof()` in order to properly validate for these two functions in `PerAddressTrancheVestingMerkleDistributor.sol`.

