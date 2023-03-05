0x52

medium

# First depositor can abuse exchange rate to steal funds from later depositors

## Summary

Classic issue with vaults. First depositor can deposit a single wei then donate to the vault to greatly inflate share ratio. Due to truncation when converting to shares this can be used to steal funds from later depositors.

## Vulnerability Detail

See summary.

## Impact

First depositor can steal funds due to truncation

## Code Snippet

https://github.com/sherlock-audit/2023-02-surge/blob/main/surge-protocol-v1/src/Pool.sol#L307-L343

## Tool used

[Solidity YouTube Tutorial](https://www.youtube.com/watch?v=dQw4w9WgXcQ)

## Recommendation

Either during creation of the vault or for first depositor, lock a small amount of the deposit to avoid this.