unforgiven

high

# User can get liquidated just after the borrowing in the same block

## Summary
when users call `borrow()` and borrow some loan tokens, code allows users to borrow up to `userCollateralRatioMantissa <= _currentCollateralRatioMantissa` but when calculating `debtShare` for user code rounds up and it would cause the user debt to be higher than actual debt in the same block and it can make `userCollateralRatioMantissa > _currentCollateralRatioMantissa` and user debt can be liquidated in the same block.

## Vulnerability Detail
This is `borrow()` code:
```solidity
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

        uint _debtSharesSupply = debtSharesSupply;
        uint userDebt = getDebtOf(debtSharesBalanceOf[msg.sender], _debtSharesSupply, _currentTotalDebt) + amount;
        uint userCollateralRatioMantissa = userDebt * 1e18 / collateralBalanceOf[msg.sender];
        require(userCollateralRatioMantissa <= _currentCollateralRatioMantissa, "Pool: user collateral ratio too high");

        uint _newUtil = getUtilizationMantissa(_currentTotalDebt + amount, (_currentTotalDebt + _loanTokenBalance));
        require(_newUtil <= SURGE_MANTISSA, "Pool: utilization too high");

        uint _shares = tokenToShares(amount, _currentTotalDebt, _debtSharesSupply, true);
        _currentTotalDebt += amount;

        // commit current state
        debtSharesBalanceOf[msg.sender] += _shares;
        debtSharesSupply = _debtSharesSupply + _shares;
        totalSupply = _currentTotalSupply;
        lastTotalDebt = _currentTotalDebt;
        lastAccrueInterestTime = block.timestamp;
        lastCollateralRatioMantissa = _currentCollateralRatioMantissa;
        emit Borrow(msg.sender, amount);
        if(_accruedFeeShares > 0) {
            balanceOf[_feeRecipient] += _accruedFeeShares;
            emit Transfer(address(0), _feeRecipient, _accruedFeeShares);
        }

        // interactions
        safeTransfer(LOAN_TOKEN, msg.sender, amount);
    }
```
As you can see code uses `_shares = tokenToShares(amount, _currentTotalDebt, _debtSharesSupply, true)` to calculate user debt share balance. it would calculate user debt share balance by rounding up and set it to the `debtSharesBalanceOf[]`
This is `liquidate()` code:
```solidity
    function liquidate(address borrower, uint amount) external {
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

        uint collateralBalance = collateralBalanceOf[borrower];
        uint _debtSharesSupply = debtSharesSupply;
        uint userDebt = getDebtOf(debtSharesBalanceOf[borrower], _debtSharesSupply, _currentTotalDebt);
        uint userCollateralRatioMantissa = userDebt * 1e18 / collateralBalance;
        require(userCollateralRatioMantissa > _currentCollateralRatioMantissa, "Pool: borrower not liquidatable");
.........
.........
```
As you can see it uses `getDebtOf(debtSharesBalanceOf[borrower], _debtSharesSupply, _currentTotalDebt)` to calculate user debt and if `userCollateralRatioMantissa > _currentCollateralRatioMantissa` it allows user debt to be liquidated.
This would cause issue if user decides to borrow allowed maximum amount where `userCollateralRatioMantissa == _currentCollateralRatioMantissa`. because code calculates debt share by rounding up and set the value of the `debtSharesBalanceOf[]` then code would calculate higher debt amount by calling `getDebtOf()` and that higher debt amount would cause `userCollateralRatioMantissa > _currentCollateralRatioMantissa`. imagine this scenario:
1. total debt share is 3 and total debt is 100 and `currentCollateralRatio` is 5 (total collateral is 20)
2. user has 10 collateral and would call `borrow(50)` to borrow 50 token and `userCollateralRatio` would be 5 (50/10). `currentCollateralRatio` would be 150/30 = 50. so code would allow borrwoingl.
3. code would calculate user debt share amount as `round_UP(50 * 3 / 100) = 2`. so user debt share would be 2 and total share would be 5 and total debt would be 150.
4. now in the same block user can be liquidated because `getDebtOf()` for the user would return `2 * 150 / 5 = 60` and user collateral ratio would be `60 / 10 = 6` which is higher than `currentCollateralRatio`.

This bug would create a MEV where miners can liquidates users immediately if users try to create maximal loan. and possible sandwich attack opportunities. bots can liquidate users debts right after they create a valid debt.

the issue would happen even if `userCollateralRatioMantissa` was so close to `_currentCollateralRatioMantissa` when borrowing and it doesn't require it to be equal. as interest in the same block doesn't happen so users should be liquidates in the same block where the interest for that block has already been calculated and they created a valid debt.

## Impact
Users and other contract that create a valid debt with maximum allowed debt would be liquidated in the same block and lose funds.

## Code Snippet
https://github.com/Surge-fi/surge-protocol-v1/blob/b7cb1dc2a2dcb4bf22c765a4222d7520843187c6/src/Pool.sol#L480-L485
https://github.com/Surge-fi/surge-protocol-v1/blob/b7cb1dc2a2dcb4bf22c765a4222d7520843187c6/src/Pool.sol#L572-L574

## Tool used
Manual Review

## Recommendation
have different threshold for allowed `userCollateralRatioMantissa` when borrowing and for liquidation. the liquidation threshold should be higher than allowed borrow threshold.