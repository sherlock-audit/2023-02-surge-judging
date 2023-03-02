rvierdiiev

medium

# Pool.repay can be frontruned in order to prevent repaying

## Summary
Pool.repay can be frontruned in order to prevent repaying
## Vulnerability Detail
When user wants to repay loan tokens, he calls `Pool.repay`.
https://github.com/sherlock-audit/2023-02-surge/blob/main/surge-protocol-v1/src/Pool.sol#L504-L547
```solidity
        function repay(address borrower, uint amount) external {
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


        uint _shares;
        if(amount == type(uint).max) {
            amount = getDebtOf(debtSharesBalanceOf[borrower], _debtSharesSupply, _currentTotalDebt);
            _shares = debtSharesBalanceOf[borrower];
        } else {
            _shares = tokenToShares(amount, _currentTotalDebt, _debtSharesSupply, false);
        }
        _currentTotalDebt -= amount;


        // commit current state
        debtSharesBalanceOf[borrower] -= _shares;
        debtSharesSupply = _debtSharesSupply - _shares;
        totalSupply = _currentTotalSupply;
        lastTotalDebt = _currentTotalDebt;
        lastAccrueInterestTime = block.timestamp;
        lastCollateralRatioMantissa = _currentCollateralRatioMantissa;
        emit Repay(borrower, msg.sender, amount);
        if(_accruedFeeShares > 0) {
            balanceOf[_feeRecipient] += _accruedFeeShares;
            emit Transfer(address(0), _feeRecipient, _accruedFeeShares);
        }


        // interactions
        safeTransferFrom(LOAN_TOKEN, msg.sender, address(this), amount);
    }
```
This function has `amount` param that user wants to repay.
Later this, amount is converted to shares count that will be removed from user's balance.
```solidity
        uint _shares;
        if(amount == type(uint).max) {
            amount = getDebtOf(debtSharesBalanceOf[borrower], _debtSharesSupply, _currentTotalDebt);
            _shares = debtSharesBalanceOf[borrower];
        } else {
            _shares = tokenToShares(amount, _currentTotalDebt, _debtSharesSupply, false);
        }
        _currentTotalDebt -= amount;
```

In case if user wants to repay all balance he can provide `amount` as `type(uint).max`. Also he can just provide amount he wants to repay(for example his full debt amount).

Attacker can frontrun user, repay some part of shares, in order to make user's tx revert(as user now don't have enough amount of shares) and leave him in the pool(to liquidate it later).

Scenario.
1.User has 1e18 amount of loan assets as debt.
2.He calls repay with 1e18 amount.
3.Attacker frontruns him and repays 1 wei on behalf of user.
4.Users tx reverts. Maybe after that attacker can liquidate user's position.
## Impact
Repayment is blocked.
## Code Snippet
Provided above
## Tool used

Manual Review

## Recommendation
In case if user provided amount that is bigger than his total debt, just reduce it.