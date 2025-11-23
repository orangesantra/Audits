# **M1** Improper Handling of Mixed Rewards and Slashing During Unstaking

The `_unstake` function incorrectly handles scenarios where both rewards and slashing occur during the unstaking period. When rewards are distributed followed by slashing (where net effect is still positive), stakers receive undeserved rewards that should have been returned to the prover, violating the protocol's stated intention that "rewards earned during the unstaking period do not go to the staker."

## Finding Description
In the `_unstake` function, the logic for determining how much iPROVE a staker should receive is oversimplified:
```js
if (iPROVEReceived > _iPROVESnapshot) {
    // Rewards were earned during unstaking. Return the excess to the prover.
    uint256 excess = iPROVEReceived - _iPROVESnapshot;
    IERC20(iProve).safeTransfer(_prover, excess);
    iPROVE = _iPROVESnapshot;
} else {
    // Either no change or slashing occurred. Staker gets what's available.
    iPROVE = iPROVEReceived;
}
```
The issue is in the else branch, which assumes that if iPROVEReceived <= _iPROVESnapshot, it's purely due to slashing or no change. However, this doesn't account for scenarios where:

Rewards are distributed during unstaking (increasing value)
Slashing occurs after rewards (decreasing value)
The net result is still above the original snapshot value when accounting for the rewards that should be excluded
In such cases, stakers receive more than they should because they benefit from rewards earned during the unstaking period.

Consider the following case -
Day 1: Alice stakes 1000 PROVE
Day 10: Alice requests unstake of 1000 stPROVE
iPROVESnapshot = 1000 (recorded at request time)
Day 12: Protocol dispenses rewards (+10%, total value becomes 1100)
Day 15: Prover1 gets slashed by 15% (1100 → 935)
Day 17: Alice completes unstaking and receives 935 PROVE
Problem ?
Alice should NOT get 935 PROVE. Here's the correct calculation:

What Should Happen (Protocol Intent)
Alice's original stake: 1000 PROVE
Rewards during unstaking: +100 PROVE (but she shouldn't get these)
Slashing during unstaking: -165 PROVE (15% of 1100)
Alice should get: 1000 - 165 = 835 PROVE
Alice gets 935 PROVE (the current value)
This means she effectively received: 935 - 835 = 100 PROVE extra
She got some of the rewards she shouldn't have!
The current logic in _unstake only compares:
```js
if (iPROVEReceived > _iPROVESnapshot) {
    // Return excess to prover
} else {
    // Give all iPROVEReceived to staker - THIS IS WRONG!
}
```
This logic assumes that if iPROVEReceived < _iPROVESnapshot, it's purely due to slashing. But it could be:

Net positive: Rewards > Slashing (Alice gets undeserved gains)
Net negative: Slashing > Rewards (Alice bears appropriate losses)
Impact Explanation
High Impact: This bug allows stakers to extract value from the protocol that they shouldn't receive according to the design specifications.

The financial impact compounds over time as Multiple stakers can exploit this during volatile periods with frequent rewards and slashing

## Likelihood Explanation
Medium

## PoC
Paste this function in SuccinctStaking.unstake.t.sol.
```js
function test_Bug_MixedRewardsAndSlashingDuringUnstakePeriod() public {
    uint256 stakeAmount = 1000e18; // 1000 PROVE tokens
    uint256 rewardAmount = 100e18;  // 10% reward
    uint256 slashPercentage = 15;   // 15% slash of inflated value
    
    // Alice stakes 1000 PROVE
    _permitAndStake(STAKER_1, STAKER_1_PK, ALICE_PROVER, stakeAmount);
    
    // Verify initial state
    assertEq(SuccinctStaking(STAKING).staked(STAKER_1), stakeAmount);
    assertEq(IERC20(STAKING).balanceOf(STAKER_1), stakeAmount);
    
    // Alice requests unstake of 1000 stPROVE (snapshot = 1000 iPROVE)
    vm.prank(STAKER_1);
    SuccinctStaking(STAKING).requestUnstake(stakeAmount);
    
    // Get the iPROVE snapshot from unstake request
    uint256 iPROVESnapshot = SuccinctStaking(STAKING).unstakeRequests(STAKER_1)[0].iPROVESnapshot;
    assertEq(iPROVESnapshot, stakeAmount); // Should be 1000
    
    // During unstaking period, rewards are distributed (+10%)
    MockVApp(VAPP).processFulfillment(ALICE_PROVER, rewardAmount);
    
    // Calculate and withdraw rewards to increase prover's iPROVE balance
    (uint256 protocolFee, uint256 stakerReward, uint256 ownerReward) = 
        _calculateFullRewardSplit(rewardAmount);
    _withdrawFromVApp(FEE_VAULT, protocolFee);
    _withdrawFromVApp(ALICE, ownerReward);
    _withdrawFromVApp(ALICE_PROVER, stakerReward);
    
    // Verify the prover's assets increased due to rewards
    uint256 proverAssetsAfterReward = IERC4626(ALICE_PROVER).totalAssets();
    assertApproxEqAbs(proverAssetsAfterReward, stakeAmount + stakerReward, 1);
    
    // Slashing occurs (15% of the inflated value)
    // 15% of ~1100 = ~165, bringing total to ~935
    uint256 slashAmount = (proverAssetsAfterReward * slashPercentage) / 100;
    
    MockVApp(VAPP).processSlash(ALICE_PROVER, slashAmount);
    
    // Complete the slash
    skip(SLASH_PERIOD);
    vm.prank(OWNER);
    SuccinctStaking(STAKING).finishSlash(ALICE_PROVER, 0);
    
    // Complete unstake after the unstaking period
    skip(UNSTAKE_PERIOD);
    
    uint256 proveBalanceBefore = IERC20(PROVE).balanceOf(STAKER_1);
    
    vm.prank(STAKER_1);
    uint256 proveReceived = SuccinctStaking(STAKING).finishUnstake(STAKER_1);
    
    uint256 proveBalanceAfter = IERC20(PROVE).balanceOf(STAKER_1);
    assertEq(proveBalanceAfter - proveBalanceBefore, proveReceived);
    
    // What Alice should receive vs what she actually receives
    uint256 expectedCorrectAmount = stakeAmount - ((slashAmount * stakeAmount) / (stakeAmount + stakerReward));
    
    // What Alice ACTUALLY receives (buggy implementation):
    // She gets the current value after rewards and slashing: ~935 PROVE
    
    console.log("Alice's original stake:", stakeAmount);
    console.log("Rewards earned during unstaking:", stakerReward);
    console.log("Slash amount (on inflated value):", slashAmount);
    console.log("What Alice should receive:", expectedCorrectAmount);
    console.log("What Alice actually received:", proveReceived);
    console.log("Undeserved gain:", proveReceived > expectedCorrectAmount ? proveReceived - expectedCorrectAmount : 0);
    
    assertGt(proveReceived, expectedCorrectAmount, "BUG: Alice received more than she should have");
    
    // The excess she received represents rewards she shouldn't have gotten
    uint256 undeservedGain = proveReceived - expectedCorrectAmount;
    
    uint256 expectedUndeservedGain = stakerReward - ((slashAmount * stakerReward) / (stakeAmount + stakerReward));
    assertApproxEqAbs(undeservedGain, expectedUndeservedGain, 10, 
        "Undeserved gain should equal rewards minus proportional slash");

    assertTrue(undeservedGain > 0, "Alice received undeserved rewards during unstaking period");
}
```
Output -
```js
Ran 1 test for test/SuccinctStaking.unstake.t.sol:SuccinctStakingUnstakeTests
[PASS] test_Bug_MixedRewardsAndSlashingDuringUnstakePeriod() (gas: 886132)
Logs:
  Alice's original stake: 1000000000000000000000
  Rewards earned during unstaking: 9970000000000000000
  Slash amount (on inflated value): 151495500000000000000
  What Alice should receive: 850000000000000000000
  What Alice actually received: 858474500000000000000
  Undeserved gain: 8474500000000000000

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 10.72ms (2.40ms CPU time)      

Ran 1 test suite in 17.76ms (10.72ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
Recommendation
Implement proper tracking of rewards during the unstaking period and adjust the calculation accordingly:

function _unstake(address _staker, address _prover, uint256 _stPROVE, uint256 _iPROVESnapshot)
    internal
    returns (uint256 PROVE)
{
    // ... existing code ...
    
    uint256 iPROVEReceived = IERC4626(_prover).redeem(_stPROVE, address(this), address(this));
    
    uint256 iPROVE;
    if (iPROVEReceived > _iPROVESnapshot) {
        // Pure rewards case - return excess to prover
        uint256 excess = iPROVEReceived - _iPROVESnapshot;
        IERC20(iProve).safeTransfer(_prover, excess);
        iPROVE = _iPROVESnapshot;
    } else {
        // Calculate what the value should be excluding rewards earned during unstaking
        // This requires tracking the total rewards distributed to this prover during 
        // the unstaking period and adjusting accordingly
        uint256 baseValueExcludingUnstakingRewards = _calculateBaseValue(_staker, _prover, _iPROVESnapshot);
        
        if (iPROVEReceived > baseValueExcludingUnstakingRewards) {
            // Mixed case: net positive but includes undeserved rewards
            iPROVE = baseValueExcludingUnstakingRewards;
            uint256 excess = iPROVEReceived - baseValueExcludingUnstakingRewards;
            IERC20(iProve).safeTransfer(_prover, excess);
        } else {
            // Pure slashing or slashing > rewards
            iPROVE = iPROVEReceived;
        }
    }
    
    // ... rest of function ...
}
```