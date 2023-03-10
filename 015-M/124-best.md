0x52

medium

# Operator can cause fee shares to be minted to address(0)

## Summary

When setting the fee rate it is required that the fee recipient is NOT address(0). An operator can bypass this check by changing the fee recipient to address(0) after setting fee.

## Vulnerability Detail

https://github.com/sherlock-audit/2023-02-surge/blob/main/surge-protocol-v1/src/Factory.sol#L60-L65

When setting the fee it is required that if the fee != 0 then the fee recipient != address(0)

https://github.com/sherlock-audit/2023-02-surge/blob/main/surge-protocol-v1/src/Factory.sol#L52-L55

When setting the fee recipient there is no similar check. This means that an operator can bypass the check in setFeeMantissa by setting the fee recipient to address(0) after setting a nonzero fee value.

## Impact

Operator can bypass fee recipient check

## Code Snippet

https://github.com/sherlock-audit/2023-02-surge/blob/main/surge-protocol-v1/src/Factory.sol#L52-L55

## Tool used

[Solidity YouTube Tutorial](https://www.youtube.com/watch?v=dQw4w9WgXcQ)

## Recommendation

Implement a check similar to the one in setFeeMantissa that doesn't allow a nonzero fee when fee recipient = address(0)