0x52

medium

# Fee share calculation is incorrect

## Summary

Fees are given to the feeRecipient by minting them shares. The current share calculation is incorrect and always mints too many shares the fee recipient, giving them more fees than they should get.

## Vulnerability Detail

The current equation is incorrect and will give too many shares, which is demonstrated in the example below.

Example:

    _supplied = 100
    _totalSupply = 100
    
    _interest = 10
    fee = 2

Calculate the fee with the current equation:

    _accuredFeeShares = fee * _totalSupply / supplied = 2 * 100 / 100 = 2

This yields 2 shares. Next calculate the value of the new shares:

    2 * 110 / 102 = 2.156

The value of these shares yields a larger than expected fee. Using a revised equation gives the correct amount of fees:

    _accuredFeeShares = (_totalSupply * fee) / (_supplied + _interest - fee) = 2 * 100 / (100 + 10 - 2) = 1.852
    
    1.852 * 110 / 101.852 = 2

This new equation yields the proper fee of 2.

## Impact

Fee recipient is given more fees than intended, which results in less interest for LPs

## Code Snippet

https://github.com/sherlock-audit/2023-02-surge/blob/main/surge-protocol-v1/src/Pool.sol#L161-L165

## Tool used

[Solidity YouTube Tutorial](https://www.youtube.com/watch?v=dQw4w9WgXcQ)

## Recommendation

Use the modified equation shown above:

        uint fee = _interest * _feeMantissa / 1e18;
        // 13. Calculate the accrued fee shares
    -   _accruedFeeShares = fee * _totalSupply / _supplied; // if supplied is 0, we will have returned at step 7
    +   _accruedFeeShares = fee * (_totalSupply * fee) / (_supplied + _interest - fee); // if supplied is 0, we will have returned at step 7
        // 14. Update the total supply
        _currentTotalSupply += _accruedFeeShares;