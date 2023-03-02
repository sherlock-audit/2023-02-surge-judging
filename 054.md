bytes032

high

# You can brick the pool by depositing a small amount of ether

## Summary

A first time depositor can brick the whole pool by depositing a very small amount of ether (1 wei).


## Vulnerability Detail

The `deposit` function allows users to deposit a specified amount of a loan tokens into a pool. The function then calculates the current state of the pool based on the current balances of the loan token, the pool token, and the fees charged by the pool.

Next, it calculates the number of pool tokens that the user will receive in exchange for the deposited token using the `tokenToShares` function.

The `tokenToShares` function calculates the number of pool tokens that will be minted when a certain amount of a token is deposited into the pool based on the total token balance held by the pool and the total number of pool tokens in circulation, with an option to round up the number of pool tokens.

Depending on the use case (deposit or borrow), `tokenToShares` is passed different arguments, hence the "general" formula is the same, but the result varies.

Finally, if the number of shares is greater than zero, the `deposit` updates the current total supply of "pool" tokens and transfers the loan tokens to the pool from the depositor.

Now, lets get back to `tokenToShares`. In the `deposit` scenario, if we use the parameters that it is passing to the function, the equation would look like this:

$$shares = \dfrac{amount * (currentTotalDebt + loanTokenBalance)}{currentTotalSupply}$$

However, at the very first deposit the `supplied` is zero, because there is no debt and no loan token balance, hence it will return the `tokenAmount` without any further calculations.

$$supplied = (currentTotalDebt + loanTokenBalance)$$

The issue arises from the line of code below, which can actually break the invariant above and brick the contract if a malicious actor deposits tokens to the pool indirectly by interacting with the loan token. Unexpected balance is a well known issue that you can read more about [here](https://swcregistry.io/docs/SWC-132)

`uint _loanTokenBalance = LOAN_TOKEN.balanceOf(address(this));`

So, it would work like this:
1. New pool is created, the pool's loan, debt, and total supply equal 0.
2. Alice is generous and *donates* 1 wei of loan token to the pool address.
3. Now, Bob is excited to use Surge's new pool to make some cash and wants to deposit 10 * 1e18 of loan token.
4. No matter what amount of tokens he's trying to deposit, the deposit function will revert, because shares will never > 0, because the invariant `if(currentTotalDebt + loanTokenBalance) == 0) return _tokenAmount` is broken.

$$shares = \dfrac{(10 * 1e18) * (0 + 1)}{0}$$

Finally, here's a proof of concept how would that work:

```solidity

   function testBrickDeposit() external {
        uint amount = 1000 ether;
        MockERC20 collateralToken = new MockERC20(amount, 18);
        MockERC20 loanToken = new MockERC20(amount * 2, 18);
        Pool pool = factory.deploySurgePool(IERC20(address(collateralToken)), IERC20(address(loanToken)), 1e18, 0.5e18, 1e15, 1e15, 0.1e18, 0.4e18, 0.6e18);

        address alice = makeAddr("alice");
        vm.startPrank(alice);
        loanToken.approve(address(pool), type(uint).max);
        loanToken.mint(1 ether);
        loanToken.transfer(address(pool), 1 wei);
        vm.stopPrank();

        address bob = makeAddr("bob");
        vm.startPrank(bob);
        loanToken.approve(address(pool), type(uint).max);
        loanToken.mint(1 ether);

        pool.deposit(1 ether);
    }

```


## Impact

The impact here is that a new pool can be completely bricked and rendered useless by depositing a very small amount of ether (1 wei), meaning that a malicious actor can brick all the new pools deployed by the factory with very low efforts.

Setting severity to be high as the attack can be carried out by anyone willing to spend a few dollars. 


## Code Snippet

https://github.com/Surge-fi/surge-protocol-v1/blob/b7cb1dc2a2dcb4bf22c765a4222d7520843187c6/src/Pool.sol#L308

https://github.com/Surge-fi/surge-protocol-v1/blob/b7cb1dc2a2dcb4bf22c765a4222d7520843187c6/src/Pool.sol#LL199-L204C6


## Tool used

Manual Review

## Recommendation

The low hanging fruit is to automatically transfer 1 wei to any new pools, but that can be easily front run if somebody is persistent in his attempts to take down your pools.

Another possibility is to transfer 1 wei while deploying the contract, which will be safe in terms of front running, but can open another can of worms.

My strongest recommendation is to adapt your pool accounting logic so that it cannot be influenced indirectly by third parties.