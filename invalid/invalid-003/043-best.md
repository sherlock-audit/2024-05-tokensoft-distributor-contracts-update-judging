Lucky Crimson Platypus

high

# Users may suffer from loss of tokens if owner decreases their claimable tokens.

## Summary
User's claimable tokens are not claimed and settled during the `adjust` tx. If an user's current claimable tokens exceeds the new claimable tokens (been `adjust`ed downwards), the user will not be able to claim the excess ones later and will suffer a token loss.

## Vulnerability Detail
Owner can decrease user's total claimable tokens. But the tokens that currently claimable before `adjust` should be claimed and settled and transfer to user, as they are belong to user now and should not be revoked by owner.

For example:
1. Initially, Alice's total claimable tokens is 100 (`records[Alice].total=100, records[Alice].claimed=0`).
2. After some time, the tokens can be claimed by Alice is 60. These 60 tokens should nominally belong to alice, but she has not claim them yet (`getClaimableAmount(Alice, ...)=60, records[Alice].claimed=0`).
3. At this moment, owner `adjust`s Alice's total claimable tokens downwards to 50 (i.e. `records[Alice].total=50, records[Alice].claimed=0`).
4. Afterwards, Alice can claim 50 tokens at most, but she should have 60 tokens (`getClaimableAmount(Alice, ...)=50`).
```solidity
119:  function adjust(address beneficiary, int256 amount) external onlyOwner {
120:    DistributionRecord memory distributionRecord = records[beneficiary];
121:    require(distributionRecord.initialized, 'must initialize before adjusting');
122:
123:    uint256 diff = uint256(amount > 0 ? amount : -amount);
124:    require(diff < type(uint120).max, 'adjustment > max uint120');
125:
126:    if (amount < 0) {
127:      // decreasing claimable tokens
128:      require(total >= diff, 'decrease greater than distributor total');
129:      require(distributionRecord.total >= diff, 'decrease greater than distributionRecord total');
130:      total -= diff;
131:@>    records[beneficiary].total -= uint120(diff);
132:      token.safeTransfer(owner(), diff);
133:    } else {
134:      // increasing claimable tokens
135:      total += diff;
136:      records[beneficiary].total += uint120(diff);
137:    }
138:    _reconcileVotingPower(beneficiary);
139:    emit Adjust(beneficiary, amount);
140:  }
```
https://github.com/sherlock-audit/2024-05-tokensoft-distributor-contracts-update/blob/main/contracts/packages/hardhat/contracts/claim/abstract/AdvancedDistributor.sol#L119-L140

`AdvancedDistributorInitializable.adjust` has the same problem.
```solidity
124:    function adjust(address beneficiary, int256 amount) external onlyOwner {
125:        DistributionRecord memory distributionRecord = records[beneficiary];
126:        require(distributionRecord.initialized, "must initialize before adjusting");
127:
128:        uint256 diff = uint256(amount > 0 ? amount : -amount);
129:        require(diff < type(uint120).max, "adjustment > max uint120");
130:
131:        if (amount < 0) {
132:            // decreasing claimable tokens
133:            require(total >= diff, "decrease greater than distributor total");
134:            require(distributionRecord.total >= diff, "decrease greater than distributionRecord total");
135:            total -= diff;
136:@>          records[beneficiary].total -= uint120(diff);
137:            token.safeTransfer(owner(), diff);
138:        } else {
139:            // increasing claimable tokens
140:            total += diff;
141:            records[beneficiary].total += uint120(diff);
142:        }
143:        _reconcileVotingPower(beneficiary);
144:        emit Adjust(beneficiary, amount);
145:    }
```
https://github.com/sherlock-audit/2024-05-tokensoft-distributor-contracts-update/blob/main/contracts/packages/hardhat/contracts/claim/factory/AdvancedDistributorInitializable.sol#L124-L145

## Impact
User may suffer from token loss if their total claimable tokens get decreased.

## Code Snippet
https://github.com/sherlock-audit/2024-05-tokensoft-distributor-contracts-update/blob/main/contracts/packages/hardhat/contracts/claim/abstract/AdvancedDistributor.sol#L119-L140

https://github.com/sherlock-audit/2024-05-tokensoft-distributor-contracts-update/blob/main/contracts/packages/hardhat/contracts/claim/factory/AdvancedDistributorInitializable.sol#L124-L145

## Tool used

Manual Review

## Recommendation
At the begining of the `adjust` function, claim and settle user's claimable tokens first.