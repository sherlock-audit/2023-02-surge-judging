unforgiven

high

# fund loss because calculated Interest would be 0 in getCurrentState() due to division error

## Summary
function `getCurrentState()` Gets the current state of pool variables based on the current time and other functions use it to update the contract state. it calculates interest accrued for debt from the last timestamp but because of the division error in some cases the calculated interest would be 0 and it would cause borrowers to pay no interest.

## Vulnerability Detail
This is part of `getCurrentState()` code that calculates interest:
```solidity
 // 2. Get the time passed since the last interest accrual
        uint _timeDelta = block.timestamp - _lastAccrueInterestTime;
        
        // 3. If the time passed is 0, return the current values
        if(_timeDelta == 0) return (_currentTotalSupply, _accruedFeeShares, _currentCollateralRatioMantissa, _currentTotalDebt);
        
        // 4. Calculate the supplied value
        uint _supplied = _totalDebt + _loanTokenBalance;
        // 5. Calculate the utilization
        uint _util = getUtilizationMantissa(_totalDebt, _supplied);

        // 6. Calculate the collateral ratio
        _currentCollateralRatioMantissa = getCollateralRatioMantissa(
            _util,
            _lastAccrueInterestTime,
            block.timestamp,
            _lastCollateralRatioMantissa,
            COLLATERAL_RATIO_FALL_DURATION,
            COLLATERAL_RATIO_RECOVERY_DURATION,
            MAX_COLLATERAL_RATIO_MANTISSA,
            SURGE_MANTISSA
        );

        // 7. If there is no debt, return the current values
        if(_totalDebt == 0) return (_currentTotalSupply, _accruedFeeShares, _currentCollateralRatioMantissa, _currentTotalDebt);

        // 8. Calculate the borrow rate
        uint _borrowRate = getBorrowRateMantissa(_util, SURGE_MANTISSA, MIN_RATE, SURGE_RATE, MAX_RATE);
        // 9. Calculate the interest
        uint _interest = _totalDebt * _borrowRate * _timeDelta / (365 days * 1e18); // does the optimizer optimize this? or should it be a constant?
        // 10. Update the total debt
        _currentTotalDebt += _interest;
```
code should support all the ERC20 tokens and those tokens may have different decimals. also different pools may have different values for MIN_RATE, SURGE_RATE, MAX_RATE. imagine this scenario:
1. debt token is USDC and has 6 digit decimals.
2. MIN_RATE is 5% (2 * 1e16) and MAX_RATE is 10% (1e17) and in current state borrow rate is 5% (5 * 1e16)
3. timeDelta is 2 second. (two seconds passed from last accrue interest time)
4. totalDebt is 100M USDC (100 * 1e16).
5. each year has about 31M seconds (31 * 1e6).
6. now code would calculate interest as: `_totalDebt * _borrowRate * _timeDelta / (365 days * 1e18) = 100 * 1e6 * 5 * 1e16 * 2 / (31 * 1e16 * 1e18) = 5 * 2 / 31 = 0`.
7. so code would calculate 0 interest in each interactions and borrowers would pay 0 interest. the debt decimal and interest rate may be different for pools and code should support all of them.

## Impact
borrowers won't pay any interest and lenders would lose funds.

## Code Snippet
https://github.com/Surge-fi/surge-protocol-v1/blob/b7cb1dc2a2dcb4bf22c765a4222d7520843187c6/src/Pool.sol#L105-L156

## Tool used
Manual Review

## Recommendation
don't update contract state(`lastAccrueInterestTime`) when calculated interest is 0.
add more decimal to total debt and save it with extra 1e18 decimals and transferring or receiving debt token convert the token amount to more decimal format or from it.