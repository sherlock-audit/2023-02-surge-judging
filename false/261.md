cccz

medium

# When amount == 0, the logic in borrow and withdraw is incorrect

## Summary
In withdraw, when amount == 0, it will decrease the user's 1 wei share
In borrow, when amount == 0, it will increase the user's 1 wei debtshare
## Vulnerability Detail
In withdraw, when amount == 0, since roundUpCheck in tokenToShares is true, it returns 1 wei share, which will eventually reduce the user's 1 wei share
In borrow, when amount == 0, since roundUpCheck in tokenToShares is true, it returns 1 wei share, which will eventually increase the user's 1 wei debtshare
## Impact
In withdraw, when amount == 0, it will decrease the user's 1 wei share
In borrow, when amount == 0, it will increase the user's 1 wei debtshare
## Code Snippet
https://github.com/sherlock-audit/2023-02-surge/blob/main/surge-protocol-v1/src/Pool.sol#L455-L498
https://github.com/sherlock-audit/2023-02-surge/blob/main/surge-protocol-v1/src/Pool.sol#L348-L389
## Tool used

Manual Review

## Recommendation
```diff
    function withdraw(uint amount) external {
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
+       require(amount > 0, "Pool: 0 amount");
        uint _shares;
...
    function borrow(uint amount) external {
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
+       require(amount > 0, "Pool: 0 amount");
        uint _debtSharesSupply = debtSharesSupply;
```
