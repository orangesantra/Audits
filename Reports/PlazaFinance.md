# **H1** - In Pool.sol::getRedeemAmount() there is nothing for marketRate of levETH.
- https://github.com/sherlock-audit/2024-12-plaza-finance-judging/issues/133

## Summary
The marketRate value being passed in `getRedeemAmount()` is only of BONDeth, there is nothing for marketRate of levETH.

https://github.com/sherlock-audit/2024-12-plaza-finance/blob/14a962c52a8f4731bbe4655a2f6d0d85e144c7c2/plaza-evm/src/Pool.sol#L519

function flow

User wants to redeem levETH, so he chooses the token tyoe as levETH.
so the function execution will be -user -> redeem -> _redeem -> simulateRedeem -> getRedeemAmount.
if the redeemRate is higher than the param marketRate and marketRate is non-zero then below condition will execute -
```js
    if (marketRate != 0 && marketRate < redeemRate) {
      redeemRate = marketRate;
    }
```
now as the marketRate is of bondETH, not levETH. the new redeemRate of levETH will be set for marketRate of BONDeth. which will be unexpected.

## Root Cause
No marketRate of levETH is passed in the param.

## Impact
User will get incorrect Redeem amount value, either causing fund loss to user itself or protocol.

## Mitigation
Passing the marketRate of levETH in the getRedeemAmount function.

# **H2** - Incorrect logic for calculation of shares in BondToken::getIndexedUserAmount.
- https://github.com/sherlock-audit/2024-12-plaza-finance-judging/issues/133

## Summary
The function getIndexedUserAmount() calculates shares for each iteration as -
https://github.com/sherlock-audit/2024-12-plaza-finance/blob/14a962c52a8f4731bbe4655a2f6d0d85e144c7c2/plaza-evm/src/BondToken.sol#L195

`shares += (balance * globalPool.previousPoolAmounts[i].sharesPerToken).toBaseUnit(SHARES_DECIMALS);`

The issue -

for this calculation the balance is considered constant for each period or each iteration.
But there could be possibility that balance of user at different periods is different.
In that scenario, the above formula is incorrect as it considers balance of user at every period constant.

## Root Cause
Consider that balance of user is constant throughout each period.

## Impact
Incorrect shares calculation, getIndexedUserAmount() is called in Distributor.sol::claim() function, if incorrect shares is being fetched user will claim unexpected amount, leading to loss of funds to either protocol or user itself.

## Mitigation
Implemenation something like below will be more accurate, i.e. storing and fecthing from mapping of user and period -

`shares += (balanceofUserAtEachPeriod[user][i] * globalPool.previousPoolAmounts[i].sharesPerToken).toBaseUnit(SHARES_DECIMALS);`

# **M1** - Precision loss in calculation of redeemRate in Pool.sol::getRedeemAmount function.
- https://github.com/sherlock-audit/2024-12-plaza-finance-judging/issues/94

## Summary
In getRedeemAmount function the redeemRate at one instance is calculated as -
https://github.com/sherlock-audit/2024-12-plaza-finance/blob/14a962c52a8f4731bbe4655a2f6d0d85e144c7c2/plaza-evm/src/Pool.sol#L514
`redeemRate = ((tvl - (bondSupply * BOND_TARGET_PRICE)) / assetSupply) * PRECISION;`

It's clearly divison before multiplication. which is classic case of precision error.

If numerator is less than denominator, then the result will be always zero.
and if redeemRate 0, due to above calculation the redeemAMount is also 0. due to following calculation.
`return ((depositAmount * redeemRate).fromBaseUnit(oracleDecimals) / ethPrice) / PRECISION;`

## Root Cause
Divison before multiplication

## Impact
Incorrect calculation of redeemRate and redeemAmount, hence user will not able to get correct amount of wstETH, that's intended.

## Mitigation
Using this instead -
`redeemRate = ((tvl - (bondSupply * BOND_TARGET_PRICE)) * PRECISION) / assetSupply;`