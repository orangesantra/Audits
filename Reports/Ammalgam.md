## Perverse Incentives in Ammalgam's Depletion Bonus Mechanism

## Summary
Ammalgam's depletion bonus mechanism contains a design flaw where smaller deposits receive proportionally larger bonuses than larger deposits, creating perverse incentives that encourage inefficient user behavior and potentially harmful exploitation of the protocol.

## Finding Description
The issue exists in the depletionReserveAdjustmentWhenAssetIsAdded function which calculates bonus credits for users who deposit assets into depleted pools:
```js
function depletionReserveAdjustmentWhenAssetIsAdded(
    uint256 amountAssets,
    uint256 reserveAssets,
    uint256 _missingAssets
) private pure returns (uint256 adjustReserves_) {
    if (reserveAssets * BUFFER < _missingAssets * MAG2) {
        if (
            amountAssets > _missingAssets
                || (reserveAssets - amountAssets) * BUFFER >= (_missingAssets - amountAssets) * MAG2
        ) {
            adjustReserves_ = reserveAssets - _missingAssets;
        } else {
            adjustReserves_ = amountAssets;
        }
    }
}
```
The function creates an economic "cliff" based on whether a deposit restores pool health:

- Small deposits (that don't restore pool health) receive a 100% bonus: adjustReserves_ = amountAssets
- Large deposits (that do restore health) receive a fixed bonus: adjustReserves_ = reserveAssets - _missingAssets
This creates a scenario where a user depositing a small amount can receive a much higher percentage bonus than someone depositing a larger amount.

## Impact Explanation
This design flaw has significant economic implications:

- Inefficient Capital Allocation: Users are incentivized to make multiple small deposits instead of one large deposit, increasing gas costs and protocol overhead.
For example, in a severely depleted pool:

- User A deposits 10 ETH → receives 10 ETH bonus (100%)
- User B deposits 30 ETH → receives 4 ETH bonus (13.3%)
- User C makes 3 separate 10 ETH deposits → receives 30 ETH in bonuses (100% each time)
This clearly shows how a rational user would choose to split their deposits to maximize bonuses, which is economically inefficient for the protocol.

- Example
Example 1: Severely Depleted ETH Pool (96% Depleted)
Starting conditions:

ETH reserve: 100 ETH
Missing ETH: 96 ETH (96% depleted)
User deposits: 10 ETH
Calculation:

Check if severely depleted:

reserveAssets * BUFFER < _missingAssets * MAG2
100 * 95 < 96 * 100
9,500 < 9,600
✅ Pool is severely depleted

Check if deposit restores pool health:

amountAssets > _missingAssets || (reserveAssets - amountAssets) * BUFFER >= (_missingAssets - amountAssets) * MAG2
10 > 96 || (100 - 10) * 95 >= (96 - 10) * 100
false || 8,550 >= 8,600
false || false
❌ Small deposit doesn't restore pool health

- Therefore:

adjustReserves_ = amountAssets = 10
Result: User deposits 10 ETH but receives credit for 20 ETH (10 ETH deposit + 10 ETH bonus)

Example 2: Same Pool With Larger Deposit
Starting conditions:

ETH reserve: 100 ETH
Missing ETH: 96 ETH (96% depleted)
User deposits: 30 ETH
Calculation:

Pool is still severely depleted (same as above)

Check if deposit restores pool health:

30 > 96 || (100 - 30) * 95 >= (96 - 30) * 100
false || 6,650 >= 6,600
false || true
✅ Large deposit partially restores pool health

Therefore:

adjustReserves_ = reserveAssets - _missingAssets = 100 - 96 = 4
Result: User deposits 30 ETH but receives credit for 34 ETH (30 ETH deposit + 4 ETH bonus)