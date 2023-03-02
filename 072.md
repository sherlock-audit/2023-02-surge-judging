rvierdiiev

medium

# Pool decimals is 18, but it's actually loanToken.decimals

## Summary
Pool decimals is 18, but it's actually loanToken.decimals. Because of that protocol that will integrate surge, will have some problems.
## Vulnerability Detail
Pool.decimals [is hardcoded to 18](https://github.com/sherlock-audit/2023-02-surge/blob/main/surge-protocol-v1/src/Pool.sol#L26).
But actually, the precision of Pool token [depends on loan token](https://github.com/sherlock-audit/2023-02-surge/blob/main/surge-protocol-v1/src/Pool.sol#L324).

https://github.com/sherlock-audit/2023-02-surge/blob/main/surge-protocol-v1/src/Pool.sol#L199-L204
```solidity
    function tokenToShares (uint _tokenAmount, uint _supplied, uint _sharesTotalSupply, bool roundUpCheck) internal pure returns (uint) {
        if(_supplied == 0) return _tokenAmount;
        uint shares = _tokenAmount * _sharesTotalSupply / _supplied;
        if(roundUpCheck && shares * _supplied < _tokenAmount * _sharesTotalSupply) shares++;
        return shares;
    }
```

As you can see, amount of Pool tokens that will be minted, depends on loan token and it's precision. 
## Impact
This can have bad impact for the protocols, that will integrate surge protocol. This can break token calculations for them as they will use `decimals` variable.
## Code Snippet
Provided above
## Tool used

Manual Review

## Recommendation
You need to convert to 18 decimals or you need to initialize Pool with loan token's decimals amount.