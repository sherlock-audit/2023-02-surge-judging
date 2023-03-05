0Kage

high

# Liquidations would freeze when collateral price falls faster than drop in collateral ratio, ie. longer `collateralRatio fall duration` can freeze liquidations and lead to total loss for depositors

## Summary
As per [medium article by Nour Haridy](https://medium.com/surge-fi/introduction-to-surge-protocol-overview-34cc828d7c50), `Collateral ratio fall duration` is the time needed for collateral ratio to move from max to zero. A longer duration would mean a smaller change change in collateral ratio over any 1 hour duration. In a scenario where price of collateral token viz-a-viz loan token is dropping fast, this slow change in collateral ratio can lead to a freeze in liquidations.

If liquidations are frozen, bad debt builds up in the pool & causes a total loss to depositors. (See POC)

## Vulnerability Detail
Rate of change of collateral ratio during surge is controlled by state variables `COLLATERAL_RATIO_FALL_DURATION`, `MAX_COLLATERAL_RATIO_MANTISSA`, both of which are immutable. Over any time (`timeDelta`), drop in collateral ratio while in surge is defined as:

```solidity
    uint timeDelta = _now - _lastAccrueInterestTime;
    uint change = timeDelta * _maxCollateralRatioMantissa / _collateralRatioFallDuration;
```

Notice that for a given pool, change over a given `timeDelta` is always fixed (assuming in surge) over the life of that pool. For pools with longer `collateralRatioFallDuration`, collateral ratio falls slower compared to pools with shorter `collateralRatioFallDuration`.

When utilisation is above surge, collateral ratio continuously keeps falling over time. As per protocol design, any user borrowing at a specific collateral ratio gets liquidated if pool collateral ratio goes below the user collateral ratio. 

However this is **ONLY** plausible if liquidation is profitable for liquidators. 

Liquidators are repaying debt in `loan tokens` and receiving `collateral tokens` determined by the current collateral ratio. If price of collateral token viz-a-viz loan token falls faster than the rate of change of collateral ratio, a liquidator will be forced to liquidate at a loss. By the time pool allows liquidation at a given collateral ratio, price would already be much lower (refer POC below)

## Impact
Let's look at a POC to understand this better

- Let's say a  `DAI (loan token) - USDC (collateral token)` pool has 10 million DAI deposits
- Max collateral ratio is 0.9, surge threshold - 80%
- Collateral ratio fall duration = 9 days (0.1 drop per day)
- Bob, Marley have borrowed 5 million & 3 million from pool at collateral ratios of 0.8 & 0.6 respectively

- Current utilisation at T = 0 is 80% (8 / 10)
- At T = 1 days, assume USDC starts to de-peg. Price of DAI-USDC moves to 1.1 (1 DAI = 1.1 USDC)
- Some depositors withdraw 1 million, utilisation > 80%, surge is ON

- During the next 24 hrs, USDC de-pegs further (1 DAI = 1.333 USDC or 1 USDC = 0.75 DAI)
- At T = 2 days, surge is still on & collateral ratio now reaches 0.8 (0.9 - 0.1)
- At this level, Bob should get liquidated. A liquidator has to pay 5 million DAI & gets 6.25 million USDC (5/0.8) from pool
- However, market price dictates that liquidator can swap 5 million DAI for  6.66 million USDC (5 / 0.75) on Uniswap
- No liquidator would want to execute above txn for a net loss of 400k USDC (6.66m-6.25m)
- As USDC drops in value further & faster, more and more bad debt piles up in the pool.
- Borrowers with higher collateral ratios will anyway default - collateral tokens they deposited in pool are worth much less in the market than the loan tokens they have to repay


Effectively, we reach a state where borrowers don't repay, liquidators don't liquidate such borrowers - leading to a total loss to depositors.

Unlike a traditional lending pool where debt gets dynamically liquidated at prevailing market prices, current model piles up bad debt because the reflexiveness of the platform (ie. the rate of change of collateral ratio) is fixed upfront & cannot keep up with dynamic market changes. 

## Code Snippet
https://github.com/Surge-fi/surge-protocol-v1/blob/b7cb1dc2a2dcb4bf22c765a4222d7520843187c6/src/Pool.sol#L252

## Tool used

Manual Review

## Recommendation

This seems to be fundamental design issue. Completely ignoring prices would mean that protocol cannot increase/decrease its rate of change of collateral ratio to keep up with the market. One possible solution would have been to have a dynamic collateral fall duration that can be changed by pool deployer but that opens another set of attack vectors.

Can't think of a possible remedy that doesn't involve price oracles (involving price oracles takes us back to square one)