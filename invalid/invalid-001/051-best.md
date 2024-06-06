Mysterious Mandarin Crow

medium

# Every user can claim all the tokens instantly without waiting for a vesting period to unlock the tokens because of hardcoded bytes32(0) passed in the claim function.

## Summary
Currently due to not encoding the cliff,start,end time in the claim function a user is able to claim all the tokens instantly which allows the users to unlock all the tokens instantly .
## Vulnerability Detail
Following  is claim function in PerAddressContinuousVestingMerkle.sol
```solidity
 function claim(
    uint256 index, // the beneficiary's index in the merkle root
    address beneficiary, // the address that will receive tokens
    uint256 totalAmount, // the total claimable by this beneficiary
    uint256 start, // the start of the vesting period
    uint256 cliff, // cliff time
    uint256 end, // the end of the vesting period
    bytes32[] calldata merkleProof
  )
    external
    validMerkleProof(keccak256(abi.encodePacked(index, beneficiary, totalAmount, start, cliff, end)), merkleProof)
    nonReentrant
  {
    // effects
    uint256 claimedAmount = _executeClaim(beneficiary, totalAmount, new bytes(0));
    // interactions
    _settleClaim(beneficiary, claimedAmount);
  }
```
##From above note that in _executeClaim function new bytes(0) is passed as the third parameter.
_executeClaim function is as follows 
```solidity
function _executeClaim(
    address beneficiary,
    uint256 totalAmount,
    bytes memory data
  ) internal virtual override returns (uint256 _claimed) {
    _claimed = super._executeClaim(beneficiary, totalAmount, data);
    _reconcileVotingPower(beneficiary);
  }
```
It calls the _executeClaim function in the parent contract which is as follows(remember data = 0)
```solidity
function _executeClaim(
    address beneficiary,
    uint256 _totalAmount,
    bytes memory data
  ) internal virtual returns (uint256) {
    uint120 totalAmount = uint120(_totalAmount);

    // effects
    if (records[beneficiary].total != totalAmount) {
      // re-initialize if the total has been updated
      _initializeDistributionRecord(beneficiary, totalAmount);
    }
    
    uint120 claimableAmount = uint120(getClaimableAmount(beneficiary, data));
    require(claimableAmount > 0, 'Distributor: no more tokens claimable right now');

    records[beneficiary].claimed += claimableAmount;
    claimed += claimableAmount;

    return claimableAmount;
  }
```
From above claimable amount is calculated using the getClaimableAmount function
 uint120 claimableAmount = uint120(getClaimableAmount(beneficiary, data));
data field is still equal to zero.

getClaimableAmount function is as follows
```solidity
function getClaimableAmount(address beneficiary, bytes memory data) public view virtual returns (uint256) {
    require(records[beneficiary].initialized, 'Distributor: claim not initialized');

    DistributionRecord memory record = records[beneficiary];

    uint256 claimable = (record.total * getVestedFraction(beneficiary, block.timestamp, data)) /
      fractionDenominator;
    return
      record.claimed >= claimable
        ? 0 // no more tokens to claim
        : claimable - record.claimed; // claim all available tokens
  }
```
In order to calculate claimable amount getVestedFraction function is used 
From above getVestedFraction is passed with the values beneficiary,block.timestamp,data ( data = bytes(0))
getVestedFraction function is as follows
```solidity
function getVestedFraction(
		address beneficiary,
		uint256 time, // time is in seconds past the epoch (e.g. block.timestamp)
    bytes memory data
	) public view override returns (uint256) {
    (uint256 start, uint256 cliff, uint256 end) = abi.decode(data, (uint256, uint256, uint256));

		uint256 delayedTime = time- getFairDelayTime(beneficiary);
		// no tokens are vested
		if (delayedTime <= cliff) {
			return 0;
		}

		// all tokens are vested
		if (delayedTime >= end) {
			return fractionDenominator;
		}

		// some tokens are vested
		return (fractionDenominator * (delayedTime - start)) / (end - start);
	}
```
Now as data field passed was bytes(0) start,cliff,end all are equal to zero.
Now delayedtime would be always  be >= 0 as it is uint (also block.timestamp is a big number ) 
So the following condition would always hold
// all tokens are vested
		if (delayedTime >= end) {
			return fractionDenominator;
		}
And thus fractionDenominator is returned even if all the tokens were not vested.

Now going back to the calculation of claimable amount which was as follows

uint256 claimable = (record.total * getVestedFraction(beneficiary, block.timestamp, data)) /
      fractionDenominator;
This would always be equal to record.total thus a user can claim all the tokens even if all the tokens are not vested.


This is a valid finding because in the PerAddressTrancheVestingMerkle.sol claim function which is as follows 
```solidity
 function claim(
    uint256 index, // the beneficiary's index in the merkle root
    address beneficiary, // the address that will receive tokens
    uint256 totalAmount, // the total claimable by this beneficiary
    // TODO: should we be providing the tranches already abi encoded to save gas?
    Tranche[] calldata tranches, // the tranches for the beneficiary (users can have different vesting schedules)
    bytes32[] calldata merkleProof
  )
    external
    validMerkleProof(keccak256(abi.encodePacked(index, beneficiary, totalAmount, abi.encode(tranches))), merkleProof)
    nonReentrant
  {
    bytes memory data = abi.encode(tranches);
    // effects
    uint256 claimedAmount = _executeClaim(beneficiary, totalAmount, data);
    // interactions
    _settleClaim(beneficiary, claimedAmount);
  }
```
Above it can be seen clearly that data field should be encoded with the suitable data . In the above function getVestedFraction function is implemented in a different manner therefore tranches are encoded and are passed into the _executeClaim function.
Similarly the values of start,cliff,end should be encoded otherwise it doesn't makes any sense to use the following getVestedFraction function
```solidity
function getVestedFraction(
		address beneficiary,
		uint256 time, // time is in seconds past the epoch (e.g. block.timestamp)
    bytes memory data
	) public view override returns (uint256) {
    (uint256 start, uint256 cliff, uint256 end) = abi.decode(data, (uint256, uint256, uint256));

		uint256 delayedTime = time- getFairDelayTime(beneficiary);
		// no tokens are vested
		if (delayedTime <= cliff) {
			return 0;
		}

		// all tokens are vested
		if (delayedTime >= end) {
			return fractionDenominator;
		}

		// some tokens are vested
		return (fractionDenominator * (delayedTime - start)) / (end - start);
	}
```



## Impact
A user is able to claim all the tokens instantly even if they have zero vested tokens.
## Code Snippet
https://github.com/sherlock-audit/2024-05-tokensoft-distributor-contracts-update/blob/67edd899612619b0acefdcb0783ef7a8a75caeac/contracts/packages/hardhat/contracts/claim/PerAddressContinuousVestingMerkle.sol#L67
## Tool used

Manual Review

## Recommendation
Make the following changes to the claim function 
```solidity
 function claim(
    uint256 index, // the beneficiary's index in the merkle root
    address beneficiary, // the address that will receive tokens
    uint256 totalAmount, // the total claimable by this beneficiary
    uint256 start, // the start of the vesting period
    uint256 cliff, // cliff time
    uint256 end, // the end of the vesting period
    bytes32[] calldata merkleProof
  )
    external
    validMerkleProof(keccak256(abi.encodePacked(index, beneficiary, totalAmount, start, cliff, end)), merkleProof)
    nonReentrant
  {
 @==>    bytes memory data = abi.encode(start,cliff,end);
    // effects
    uint256 claimedAmount = _executeClaim(beneficiary, totalAmount, data);
    // interactions
    _settleClaim(beneficiary, claimedAmount);
  }
```