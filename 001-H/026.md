0Kage

high

# First depositor into a new pool can manipulate the pool share price

## Summary
A surge pool share price can be manipulated by first depositor who can first deposit a dust amount of loan tokens & follow it up with a much bigger donation. This inflates the pool price & can cause losses to subsequent small depositors.

## Vulnerability Detail
`tokenToShares` calculates number of pool token shares for a given amount of loan token deposit as shown below.

```solidity
    function tokenToShares (uint _tokenAmount, uint _supplied, uint _sharesTotalSupply, bool roundUpCheck) internal pure returns (uint) {
        if(_supplied == 0) return _tokenAmount;
        uint shares = _tokenAmount * _sharesTotalSupply / _supplied;
        if(roundUpCheck && shares * _supplied < _tokenAmount * _sharesTotalSupply) shares++;
        return shares;
    }
```

- A malicious user can first deposit 1 wei of loan token to get 1 wei of shares
- She can follow it up with a large loan token donation directly into the pool contract. Since this is a direct transfer, `1 wei` of share now represents a much larger value of loan tokens
- In the above formula, while `_supplied` increases (`supplied = loan token balance in pool + debt`), whereas `_sharesTotalSupply` remains the same
- Any subsequent depositor depositing smaller values of loan tokens can get zero shares (due to inflated share price )

## Impact
Small depositors who deposit after the first depositor will get 0 pool shares. This results in total loss of loan tokens deposited

## Code Snippet

https://github.com/Surge-fi/surge-protocol-v1/blob/b7cb1dc2a2dcb4bf22c765a4222d7520843187c6/src/Pool.sol#L201
https://github.com/Surge-fi/surge-protocol-v1/blob/b7cb1dc2a2dcb4bf22c765a4222d7520843187c6/src/Pool.sol#L324

## Tool used

Manual Review

## Recommendation

This is a standard issue & Uniswap V2 has solved this by burning the first 1e-15 minted shares. This dramatically increases the cost for the attacker to execute this attack.

[Find more details here](https://ethereum.stackexchange.com/questions/132491/why-minimum-liquidity-is-used-in-dex-like-uniswap)