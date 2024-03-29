0x52

high

# Precision differences when calculating userCollateralRatioMantissa causes major issues for some token pairs

## Summary

When calculating userCollateralRatioMantissa in borrow and liquidate. It divides the raw debt value (in loan token precision) by the raw collateral balance (in collateral precision). This skew is fine for a majority of tokens but will cause issues with specific token pairs, including being unable to liquidate a subset of positions no matter what.

## Vulnerability Detail

https://github.com/sherlock-audit/2023-02-surge/blob/main/surge-protocol-v1/src/Pool.sol#L474

When calculating userCollateralRatioMantissa, both debt value and collateral values are left in the native precision. As a result of this certain token pairs will be completely broken because of this. Other pairs will only be partially broken and can enter state in which it's impossible to liquidate positions.

Imagine a token pair like USDC and SHIB. USDC has a token precision of 6 and SHIB has 18. If the user has a collateral balance of 100,001 SHIB (100,001e18) and a loan borrow of 1 USDC (1e6) then their userCollateralRatioMantissa will actually calculate as zero:

    1e6 * 1e18 / 100,001e18 = 0

There are two issues with this. First is that a majority of these tokens simply won't work. The other issue is that because userCollateralRatioMantissa returns 0 there are states in which some debt is impossible to liquidate breaking a key invariant of the protocol.

Any token with very high or very low precision will suffer from this.

## Impact

Some token pairs will always be/will become broken

## Code Snippet

https://github.com/sherlock-audit/2023-02-surge/blob/main/surge-protocol-v1/src/Pool.sol#L455-L498

## Tool used

[Solidity YouTube Tutorial](https://www.youtube.com/watch?v=dQw4w9WgXcQ)

## Recommendation

userCollateralRatioMantissa should be calculated using debt and collateral values normalized to 18 decimal points