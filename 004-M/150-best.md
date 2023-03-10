TrungOre

medium

# `lastAccrueInterestTime` and `lastCollateralRatioMantissa` should be initialized instead of taking 0 as their default value

## Summary
The first value of `collateralRatioMantissa` can be high compare to the current market price which make the pool unprofitable to deposit in.
 
## Vulnerability Detail
As we can see in the [contructor](https://github.com/sherlock-audit/2023-02-surge/blob/main/surge-protocol-v1/src/Pool.sol#L47) of contract `Pool.sol` there is no initialization for 2 variables `lastAccrueInterestTime` and `lastCollateralRatioMantissa`, so these values will get 0 as their initial value. 
When the first time `Pool.getCurrentState()` is called, the `_timeDelta` calculated at [step 3](https://github.com/sherlock-audit/2023-02-surge/blob/main/surge-protocol-v1/src/Pool.sol#L128-L129) will get a enormous value since the current block.timestamp is bigger than 1677572699. Beside the value of `_util` which was calculated at [step 5](https://github.com/sherlock-audit/2023-02-surge/blob/main/surge-protocol-v1/src/Pool.sol#L133-L134) is equal to 0 <= `surgeMantissa` because there is no debt before. It will make the function `getCollateralRatioMantissa()` calling at [step 6](https://github.com/sherlock-audit/2023-02-surge/blob/main/surge-protocol-v1/src/Pool.sol#L136-L146) returns `_maxCollateralRatioMantissa`. 
```solidity=
/// url = https://github.com/sherlock-audit/2023-02-surge/blob/main/surge-protocol-v1/src/Pool.sol#L230-L245
if(_util <= _surgeMantissa) {
    // The collateral ratio can only increase if it is less than the max collateral ratio
    if(_lastCollateralRatioMantissa == _maxCollateralRatioMantissa) return _lastCollateralRatioMantissa;


    // If the collateral ratio can increase, we calculate the increase
    uint timeDelta = _now - _lastAccrueInterestTime;
    uint change = timeDelta * _maxCollateralRatioMantissa / _collateralRatioRecoveryDuration;


    // If the change in collateral ratio is greater than the max collateral ratio, we set the collateral ratio to the max collateral ratio
    if(_lastCollateralRatioMantissa + change >= _maxCollateralRatioMantissa) {
        return _maxCollateralRatioMantissa;
    } else {
        // Otherwise we increase the collateral ratio by the change
        return _lastCollateralRatioMantissa + change;
    }
} 
```
The reason for this return value is the `timeDelta` has a large value which makes `change > _collateralRatioRecoveryDuration * _maxCollateralRatioMantissa / _collateralRatioRecoveryDuration = _maxCollateralRatioMantissa`. 

Assume that the current price of COLLATERAL_TOKEN to LOAN_TOKEN is pretty low compare to `_maxCollateralRatio`. So when a depositor deposits LOAN_TOKEN into the pool, the borrowers can immediately add the collateral to the pool then borrow an amount of LOAN_TOKEN. It will profit the borrowers since the real price is low but the pool's collateral ratio is high, so he can get more LOAN_TOKEN than the value of COLLATERAL_TOKEN he staked in. 

## Impact
Pool are unprofitable to deposit loan token into when it is created

## Code Snippet
https://github.com/sherlock-audit/2023-02-surge/blob/main/surge-protocol-v1/src/Pool.sol#L235-L236

## Tool used
Manual review 

## Recommendation
Consider to let pool's creator initialize the value of `lastAccrueInterestTime` and `lastCollateralRatioMantissa`.