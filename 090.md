ctf_sec

medium

# Utilization rate is vulnerable to manipulation by directly injecting token balance

## Summary

Utilization rate is vulnerable to manipulation by directly injecting token balance

## Vulnerability Detail

In the current implementation, the spot balance of the contract is used to calculate the UtilizationMantissa

the code is used in all function (deposit / withdraw / removeColalteral / repay / liquidating)

```solidity
  uint _loanTokenBalance = LOAN_TOKEN.balanceOf(address(this));
  (address _feeRecipient, uint _feeMantissa) = FACTORY.getFee();
  (  
      uint _currentTotalSupply,
      uint _accruedFeeShares,
      uint _currentCollateralRatioMantissa,
      uint _currentTotalDebt
  ) = getCurrentState(
      _loanTokenBalance,
      _feeMantissa,
      lastCollateralRatioMantissa,
      totalSupply,
      lastAccrueInterestTime,
      lastTotalDebt
  );
```

which calls:

```solidity
    function getCurrentState(
        uint _loanTokenBalance,
        uint _feeMantissa,
        uint _lastCollateralRatioMantissa,
        uint _totalSupply,
        uint _lastAccrueInterestTime,
        uint _totalDebt
        ) internal view returns (
            uint _currentTotalSupply,
            uint _accruedFeeShares,
            uint _currentCollateralRatioMantissa,
            uint _currentTotalDebt
        ) {
        
        // 1. Set default return values
        _currentTotalSupply = _totalSupply;
        _currentTotalDebt = _totalDebt;
        _currentCollateralRatioMantissa = _lastCollateralRatioMantissa;
        // _accruedFeeShares = 0;

        // 2. Get the time passed since the last interest accrual
        uint _timeDelta = block.timestamp - _lastAccrueInterestTime;
        
        // 3. If the time passed is 0, return the current values
        if(_timeDelta == 0) return (_currentTotalSupply, _accruedFeeShares, _currentCollateralRatioMantissa, _currentTotalDebt);
        
        // 4. Calculate the supplied value
        uint _supplied = _totalDebt + _loanTokenBalance;
        // 5. Calculate the utilization
        uint _util = getUtilizationMantissa(_totalDebt, _supplied);
```

the usage also includes the code below in the borrow function:

```solidity
uint _newUtil = getUtilizationMantissa(_currentTotalDebt + amount, (_currentTotalDebt + _loanTokenBalance));
require(_newUtil <= SURGE_MANTISSA, "Pool: utilization too high");
```

which calls

```solidity
  /// @notice Gets the current pool utilization rate in mantissa (scaled by 1e18)
  /// @param _totalDebt The total debt of the pool
  /// @param _supplied The total supplied loan tokens of the pool
  /// @return uint The pool utilization rate in mantissa (scaled by 1e18)
  function getUtilizationMantissa(uint _totalDebt, uint _supplied) internal pure returns (uint) {
      if(_supplied == 0) return 0;
      return _totalDebt * 1e18 / _supplied;
  }
```

However, because the code use LOAN_TOKEN.balanceOf(address(this)) spot balancer, a user transfer the LOAN_TOKEN directly to the Pool contract to inflate the _loanTokenBalance, this can just inlfate the UtilizationMantissa and revert the borrow transaction

## Impact

Inflated loan token balance impact protocol accounting and can revert the borrow transaction.

## Code Snippet

https://github.com/sherlock-audit/2023-02-surge/blob/main/surge-protocol-v1/src/Pool.sol#L308

## Tool used

Manual Review

## Recommendation

We recommend use a variable to keep track of the loan amount instead of using the spot balance from balanceOf(address(this))
