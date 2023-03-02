rvierdiiev

medium

# Attacker can use reorg attack in order to steal depositors funds

## Summary
Attacker can use reorg attack in order to steal depositors funds.
## Vulnerability Detail
Surge protocol is going to be deployed on different chains. 
> DEPLOYMENT: Mainnet, Optimism, Arbitrum, Fantom, Avalanche, Polygon, BNB Chain and other EVM chains

Some of the chains (Polygon, Optimism, Arbitrum) to which the Factory will be deployed are suspicious of the reorg attack. Some of these reorgs takes long time which is enough to deploy pool and deposit into it.

`Factory.deploySurgePool` is creating Pool contract using `create` method, and address of new Pool depends only on nonce of Factory.
Because of that next situation is possible.
1.Honest user creates Pool with his own params that will have impact on interest rate and liquidation ratio. 
2.Then it uses Pool.deposit in order to provide funds that can be borrowed.
3.Attacker notices reorg and he calls `Factory.deploySurgePool` with bigger amount of gas(to deploy this contract before honest user). Also he provides own params [to the function](https://github.com/sherlock-audit/2023-02-surge/blob/main/surge-protocol-v1/src/Factory.sol#L164-L171). For example he provides malicious token as collateral.
4.Then tx of honest user are executed and it will call deposit on malicious Pool, created by attacker.
5.Attacker lends all tokens of owner for free.

Link to [similar issue on c4](https://github.com/code-423n4/2023-01-rabbithole-findings/issues/661).
## Impact
Depositor loses funds.
## Code Snippet
https://github.com/sherlock-audit/2023-02-surge/blob/main/surge-protocol-v1/src/Factory.sol#L162-L192
## Tool used

Manual Review

## Recommendation
Use `create2` with salt that will include all important params.