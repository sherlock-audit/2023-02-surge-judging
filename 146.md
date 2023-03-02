TrungOre

medium

# Lack of slippage check when calling repay can lead to loss of fund for user

## Summary
Function `Pool.repay()`, which has no check whether the `shares` will be burned is bigger than 0 or not, can make user lose LOAN_TOKEN without burning any debt shares 
 
## Vulnerability Detail
Function `Pool.repay()` is used to repay loan tokens debt. The function provides 2 options which users can specify the exact amount they want to repay or they can repay all the debt by passing `type(uint).max` into parameter `amount`. 
```solidity=
/// url = https://github.com/sherlock-audit/2023-02-surge/blob/main/surge-protocol-v1/src/Pool.sol#L504-L547
function repay(address borrower, uint amount) external {
    /// ... 

    uint _shares;
    if(amount == type(uint).max) {
        amount = getDebtOf(debtSharesBalanceOf[borrower], _debtSharesSupply, _currentTotalDebt);
        _shares = debtSharesBalanceOf[borrower];
    } else {
        _shares = tokenToShares(amount, _currentTotalDebt, _debtSharesSupply, false);
    }
    _currentTotalDebt -= amount;

    /// ... 
}
```
Obviously, the user doesn't know about which block their repay transaction will be executed, so they can't aware of the exact interest at that time. So in case when the user wants to pay a specific amount, the `tokenToShares()` can return value 0. This scenario happens when `amount * _debtSharesSupply / _currentTotalDebt < 1`. It will make the user lose their loan token without decreasing any debt shares. 

## Impact
User lose their tokens without burning any debt shares 

## Code Snippet
https://github.com/sherlock-audit/2023-02-surge/blob/main/surge-protocol-v1/src/Pool.sol#L528

## Tool used
Manual review 

## Recommendation
* Require the `_shares` burned `> 0`. 
* Should have another function to let the user specify the `shares` will be burned. 
