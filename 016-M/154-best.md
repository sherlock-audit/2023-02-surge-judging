ast3ros

high

# # [H-02] Approve and transferFrom functions of Pool tokens are subject to front-run attack.

## Summary

`Approve` and `transferFrom` functions of Pool tokens are subject to front-run attack because the `approve` method overwrites the current allowance regardless of whether the spender already used it or not. In case the spender spent the amonut, the `approve` function will approve a new amount.

## Vulnerability Detail

The `approve` method overwrites the current allowance regardless of whether the spender already used it or not. It allows the spender to front-run and spend the amount before the new allowance is set.

Scenario:

- Alice allows Bob to transfer N of Alice's tokens (N>0)  by calling the `pool.approve` method, passing the Bob's address and N as the method arguments
- After some time, Alice decides to change from N to M (M>0) the number of Alice's tokens Bob is allowed to transfer, so she calls the `pool.approve` method again, this time passing the Bob's address and M as the method arguments
- Bob notices the Alice's second transaction before it was mined and quickly sends another transaction that calls the `pool.transferFrom` method to transfer N Alice's tokens somewhere
- If the Bob's transaction will be executed before the Alice's transaction, then Bob will successfully transfer N Alice's tokens and will gain an ability to transfer another M tokens
Before Alice noticed that something went wrong, Bob calls the `pool.transferFrom` method again, this time to transfer M Alice's tokens.
- So, an Alice's attempt to change the Bob's allowance from N to M (N>0 and M>0) made it possible for Bob to transfer N+M of Alice's tokens, while Alice never wanted to allow so many of her tokens to be transferred by Bob.

## Impact

It can result in losing pool tokens of users when he approve pool tokens to any malicious account.

## Code Snippet

https://github.com/sherlock-audit/2023-02-surge/blob/main/surge-protocol-v1/src/Pool.sol#L284
https://github.com/sherlock-audit/2023-02-surge/blob/main/surge-protocol-v1/src/Pool.sol#L299

## Tool used

Manual Review

## Recommendation

Use `increaseAllowance` and `decreaseAllowance` instead of approve as OpenZeppelin ERC20 implementation. Please see details here:

https://forum.openzeppelin.com/t/explain-the-practical-use-of-increaseallowance-and-decreaseallowance-functions-on-erc20/15103/4