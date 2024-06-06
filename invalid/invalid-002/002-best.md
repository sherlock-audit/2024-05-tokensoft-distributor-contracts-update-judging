Old Saffron Skunk

medium

# Unable to claim additional allocated tokens

## Summary
Users are unable to claim additional allocated tokens in the following scenario:

1. A beneficiary has already claimed all initially allocated 100e18 tokens.
2. The contract owner allocates an additional 500e18 tokens to the beneficiary.
3. All the additional allocated tokens are unlocked.
4. The transaction to claim the additional tokens reverts.

## Vulnerability Detail
When the beneficiary claims unlocked tokens, the total claimed amount is tracked by `records[beneficiary].claimed`, [Distributor::_executeClaim()](https://github.com/sherlock-audit/2024-05-tokensoft-distributor-contracts-update/blob/67edd899612619b0acefdcb0783ef7a8a75caeac/contracts/packages/hardhat/contracts/claim/abstract/Distributor.sol#L82):

```solidity
    records[beneficiary].claimed += claimableAmount;
```

When additional tokens are allocated and unlocked, the `Distributor::getClaimableAmount()` function returns zero because the claimable amount is incorrectly calculated. The [Distributor::getClaimableAmount()](https://github.com/sherlock-audit/2024-05-tokensoft-distributor-contracts-update/blob/67edd899612619b0acefdcb0783ef7a8a75caeac/contracts/packages/hardhat/contracts/claim/abstract/Distributor.sol#L126-L128)  function checks if the total claimed tokens are greater than or equal to the claimable tokens, causing the function to return zero if the claimed amount is more than the additional allocated tokens:

```solidity
    return
      record.claimed >= claimable // @audit: if record.claimed = 100e18 and claimable = 500e18, the check condition is evaluated to true
        ? 0 // no more tokens to claim
        : claimable - record.claimed; // claim all available tokens
  }
```
The root cause is that the contract tracks the token distribution data using a mapping `mapping(address => DistributionRecord) internal records`, which cannot cover the senario that allocating tokens multiple times to a beneficiary.

## Impact
Beneficiaries cannot claim any additional tokens allocated to them after the initial allocation has been claimed.

## Code Snippet
https://github.com/sherlock-audit/2024-05-tokensoft-distributor-contracts-update/blob/67edd899612619b0acefdcb0783ef7a8a75caeac/contracts/packages/hardhat/contracts/claim/abstract/Distributor.sol#L126-L128

https://github.com/sherlock-audit/2024-05-tokensoft-distributor-contracts-update/blob/67edd899612619b0acefdcb0783ef7a8a75caeac/contracts/packages/hardhat/contracts/claim/abstract/Distributor.sol#L82

## Tool used

Manual Review

## Recommendation
Ensuring the distribution records are correctly tracked, for example, replacing the mapping with `beneficiary => leafHash => DistributionRecord`.