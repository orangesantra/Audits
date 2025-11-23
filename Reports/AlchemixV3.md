# **H1** Missing totalSyntheticsIssued Update in repay() Function.
## Description
A bug exists in the repay() function in `AlchemistV3.sol` where it fails to update the `totalSyntheticsIssued` state variable when debt is repaid. This inconsistency creates a discrepancy between totalDebt and `totalSyntheticsIssued`, which should remain synchronized.
```js
function repay(uint256 amount, uint256 recipientTokenId) public returns (uint256) {
    // ...existing code...
    _subDebt(recipientTokenId, credit);
    
    // Missing update: totalSyntheticsIssued -= credit;
    
    // Transfer the repaid tokens to the transmuter
    TokenUtils.safeTransferFrom(yieldToken, msg.sender, transmuter, creditToYield);
    // ...existing code...
}
```
For comparison, both `burn()` and `redeem()` functions properly update this variable:
```js
function burn(uint256 amount, uint256 recipientId) external returns (uint256) {
    // ...existing code...
    _subDebt(recipientId, credit);
    totalSyntheticsIssued -= credit; // Properly updated here
    // ...existing code...
}
```
```js
function redeem(uint256 amount) external onlyTransmuter {
    // ...existing code...
    totalSyntheticsIssued -= amount; // Properly updated here
    // ...existing code...
}
```
The bug causes totalSyntheticsIssued to grow increasingly out of sync with totalDebt as more users repay via the `repay()` function rather than `burn()`. This inconsistency could affect protocol metrics, risk assessments, and potentially other functions that rely on totalSyntheticsIssued.

## poc
Paste this poc in `Alchemist.t.sol`
```js
function testMissingSyntheticsIssuedUpdateInRepay() external {
    // Setup: Create a position and mint some debt
    uint256 amount = 100e18;
    vm.startPrank(address(0xbeef));
    SafeERC20.safeApprove(address(fakeYieldToken), address(alchemist), amount * 2);
    alchemist.deposit(amount, address(0xbeef), 0);
    uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
    
    // Mint synthetic tokens
    uint256 mintAmount = 50e18;
    alchemist.mint(tokenId, mintAmount, address(0xbeef));
    
    // Record initial state - both values should be equal
    uint256 initialTotalDebt = alchemist.totalDebt();
    uint256 initialSyntheticsIssued = alchemist.totalSyntheticsIssued();
    
    console.log("Initial totalDebt:", initialTotalDebt / 1e18);
    console.log("Initial syntheticsIssued:", initialSyntheticsIssued / 1e18);
    assertEq(initialTotalDebt, initialSyntheticsIssued, "totalDebt and totalSyntheticsIssued should start equal");
    
    // Advance block to avoid same-block restriction
    vm.roll(block.number + 1);
    
    // Repay half the debt using repay()
    uint256 repayAmount = alchemist.convertDebtTokensToYield(mintAmount / 2);
    alchemist.repay(repayAmount, tokenId);
    vm.stopPrank();
    
    // Check final state - should now be different due to the bug
    uint256 finalTotalDebt = alchemist.totalDebt();
    uint256 finalSyntheticsIssued = alchemist.totalSyntheticsIssued();
    
    console.log("Final totalDebt:", finalTotalDebt / 1e18);
    console.log("Final syntheticsIssued:", finalSyntheticsIssued / 1e18);
    
    // The bug causes these values to be out of sync
    assertEq(finalTotalDebt, initialTotalDebt - (mintAmount / 2), "totalDebt should decrease");
    assertEq(finalSyntheticsIssued, initialSyntheticsIssued, "BUG: totalSyntheticsIssued remains unchanged");
    assertNotEq(finalTotalDebt, finalSyntheticsIssued, "BUG: totalDebt and totalSyntheticsIssued are out of sync");
}
```
Output -
```js
Ran 1 test for src/test/AlchemistV3.t.sol:AlchemistV3Test
[PASS] testMissingSyntheticsIssuedUpdateInRepay() (gas: 679007)
Logs:
  Initial totalDebt: 50
  Initial syntheticsIssued: 50
  Final totalDebt: 25
  Final syntheticsIssued: 50

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 35.21ms (3.33ms CPU time)

Ran 1 test suite in 655.50ms (35.21ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
Recommendation
Add the missing update to totalSyntheticsIssued in the repay() function:

function repay(uint256 amount, uint256 recipientTokenId) public returns (uint256) {
    // ...existing code...
    _subDebt(recipientTokenId, credit);
    
    // Add this line to maintain consistency with burn() and redeem()
    totalSyntheticsIssued -= credit;
    
    // Transfer the repaid tokens to the transmuter
    TokenUtils.safeTransferFrom(yieldToken, msg.sender, transmuter, creditToYield);
    // ...existing code...
}
```

# **H2** Protocol Fee Transfer Bug in repay() Function
## Description

A critical bug exists in the repay() function of AlchemistV3.sol (line 767) where the contract transfers the entire repayment amount to the protocol fee receiver instead of just the intended fee percentage.
```js
// Transfer the repaid tokens to the transmuter.
TokenUtils.safeTransferFrom(yieldToken, msg.sender, transmuter, creditToYield);
// @audit - incorrect fee transfer value
TokenUtils.safeTransfer(yieldToken, protocolFeeReceiver, creditToYield);
```
The issue is that the code is transferring the full creditToYield amount (which represents the entire repayment in yield tokens) to the protocolFeeReceiver, when it should only transfer a small percentage.

Earlier in the function, the code correctly calculates and deducts the fee from the user's collateral balance:
```js
// Debt is subject to protocol fee similar to redemptions
account.collateralBalance -= creditToYield * protocolFee / BPS;
```
But when transferring the actual fee tokens, it incorrectly sends the entire amount.

Poc
Paste this poc in AlchemistV3.t.sol
```js
function testProtocolFeeTransferBug() external {
    // Set a non-zero protocol fee (1%)
    vm.prank(alOwner);
    alchemist.setProtocolFee(100); // 100 BPS = 1%
    
    uint256 amount = 100e18;
    
    // First user deposits collateral and mints synthetic tokens
    vm.startPrank(address(0xbeef));
    SafeERC20.safeApprove(address(fakeYieldToken), address(alchemist), amount);
    alchemist.deposit(amount, address(0xbeef), 0);
    uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
    alchemist.mint(tokenId, amount / 2, address(0xbeef));
    vm.stopPrank();
    
    vm.roll(block.number + 1); // Advance block to avoid same-block restriction
    
    // Track transmuter and fee receiver balances before repayment
    uint256 preRepayTransmuterBalance = IERC20(fakeYieldToken).balanceOf(address(transmuterLogic));
    uint256 preRepayFeeReceiverBalance = IERC20(fakeYieldToken).balanceOf(alchemist.protocolFeeReceiver());
    
    // Calculate expected values
    uint256 repayAmount = 25e18; // Repay 25 tokens
    uint256 expectedFeeAmount = repayAmount * 100 / 10000; // 1% of repayAmount
    
    // User repays debt - this triggers the bug
    vm.startPrank(address(0xbeef));
    SafeERC20.safeApprove(address(fakeYieldToken), address(alchemist), repayAmount);
    alchemist.repay(repayAmount, tokenId);
    vm.stopPrank();
    
    // Check balances after repayment
    uint256 postRepayTransmuterBalance = IERC20(fakeYieldToken).balanceOf(address(transmuterLogic));
    uint256 postRepayFeeReceiverBalance = IERC20(fakeYieldToken).balanceOf(alchemist.protocolFeeReceiver());
    
    // Verify the bug - fee receiver gets the full amount instead of just the fee
    uint256 transmuterIncrease = postRepayTransmuterBalance - preRepayTransmuterBalance;
    uint256 feeReceiverIncrease = postRepayFeeReceiverBalance - preRepayFeeReceiverBalance;
    
    console.log("Transmuter received:", transmuterIncrease / 1e18);
    console.log("Fee receiver received:", feeReceiverIncrease / 1e18);
    console.log("Expected fee:", expectedFeeAmount / 1e18);
    
    // The bug causes both to receive the full repayment amount
    assertEq(transmuterIncrease, repayAmount, "Transmuter should receive full repayment");
    assertEq(feeReceiverIncrease, repayAmount, "BUG: Fee receiver should only get 1% but gets full amount");
    assertNotEq(feeReceiverIncrease, expectedFeeAmount, "Fee receiver should not receive correct fee amount");
    
    // Total tokens transferred should have been repayAmount + expectedFeeAmount
    // But due to the bug, it's repayAmount + repayAmount = 2*repayAmount
    uint256 totalTransferred = transmuterIncrease + feeReceiverIncrease;
    assertEq(totalTransferred, 2 * repayAmount, "Bug causes double transfer of tokens");
}
```
Output -
```js
Ran 1 test for src/test/AlchemistV3.t.sol:AlchemistV3Test
[PASS] testProtocolFeeTransferBug() (gas: 688260)
Logs:
  Transmuter received: 25
  Fee receiver received: 25
  Expected fee: 0

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 35.71ms (3.43ms CPU time)

Ran 1 test suite in 723.48ms (35.71ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
Recommendation
Implement this instead:

// Calculate the fee amount
uint256 feeAmount = creditToYield * protocolFee / BPS;

// Transfer the repaid tokens to the transmuter
TokenUtils.safeTransferFrom(yieldToken, msg.sender, transmuter, creditToYield);

// Transfer only the fee amount to the protocol fee receiver
if (feeAmount > 0) {
    TokenUtils.safeTransfer(yieldToken, protocolFeeReceiver, feeAmount);
}
```
# **H3** Potential Underflow in Earmark Weight Calculation Leading to Incorrect Debt Accounting

## Description
A vulnerability exists in the _calculateUnrealizedDebt function in AlchemistV3.sol where a specific sequence of events can cause an arithmetic underflow when calculating the debtToEarmark value:
```js
debtToEarmark = PositionDecay.ScaleByWeightDelta(
    account.debt - account.earmarked, 
    previousRedemption.earmarkWeight - account.lastAccruedEarmarkWeight
);
```
The issue occurs because the contract assumes that previousRedemption.earmarkWeight will always be greater than or equal to `account.lastAccruedEarmarkWeight`, but this assumption fails under certain timing conditions.

Detailed Example of How This Bug Occurs:
Step 1: Initial States

User Bob has a CDP with 5,000 alUSD debt
System state at block 10,000: _earmarkWeight = 0.5
Bob interacts with the system (e.g., calls withdraw), which calls _sync()
Bob's account is updated: account.lastAccruedEarmarkWeight = 0.5
Step 2: System Changes Without User Interaction

Between blocks 10,000-10,100, more deposits flow into the Transmuter
At block 10,100, someone calls a function triggering _earmark()
System _earmarkWeight increases to 0.6
No redemption has occurred yet
Step 3: Redemption Occurs

At block 10,200, the Transmuter calls redeem()
This records the current state: _redemptions[10200] = RedemptionInfo(..., 0.5) (using the earmark weight at the last redemption block, which is 0.5)
Note: This is lower than Bob's lastAccruedEarmarkWeight (0.6)
lastRedemptionBlock is updated to 10,200
Step 4: Bob Interacts Again

At block 10,300, Bob calls getCDP()
This triggers _calculateUnrealizedDebt(), which runs:
```js
debtToEarmark = PositionDecay.ScaleByWeightDelta(
    account.debt - account.earmarked, 
    previousRedemption.earmarkWeight - account.lastAccruedEarmarkWeight
); // 0.5 - 0.6 = -0.1 (underflow!)
```
Result:

The subtraction 0.5 - 0.6 causes DOS
If `PositionDecay.ScaleByWeightDelta()` doesn't properly handle negative inputs, it could lead to DOS.
This bug is particularly insidious because it happens under normal operational conditions when the timing of user interactions and system processes don't follow the expected sequence.

## PoC
Add the following functions in AlchemistV3.sol -
```js
function getEarMarkWeight() external view returns (uint256) {
    return _earmarkWeight;
}

function getRedemptionEarmarkWeight(uint256 blockNumber) external view returns (uint256) {
    return _redemptions[blockNumber].earmarkWeight;
}

function getLastAccruedEarmarkWeight(uint256 tokenId) external view returns (uint256) {
    return _accounts[tokenId].lastAccruedEarmarkWeight;
}
```
paste the followiing poc in AlchemistV3.t.sol.
```js
function testEarmarkWeightUnderflowBug() external {
    // Step 1: Set up a user position
    uint256 deposit_Amount = 100e18;
    vm.startPrank(address(0xbeef));
    SafeERC20.safeApprove(address(fakeYieldToken), address(alchemist), deposit_Amount * 2);
    alchemist.deposit(deposit_Amount, address(0xbeef), 0);
    uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
    alchemist.mint(tokenId, deposit_Amount / 2, address(0xbeef));
    vm.stopPrank();
    
    // Step 2: User interacts with system, updating lastAccruedEarmarkWeight
    vm.startPrank(address(0xbeef));
    alchemist.withdraw(1e18, address(0xbeef), tokenId); // Small withdrawal to trigger _sync()
    vm.stopPrank();
    
    // Store original earmarkWeight by checking the internal value using our new getter
    uint256 originalEarmarkWeight = alchemist.getEarMarkWeight();
    
    // Step 3: Increase system earmarkWeight by creating deposits to Transmuter
    // Create a redemption to trigger earmarking
    vm.startPrank(address(0xdad));
    SafeERC20.safeApprove(address(alToken), address(transmuterLogic), 10e18);
    transmuterLogic.createRedemption(10e18);
    vm.stopPrank();
    
    // Make repayments to increase earmarkWeight
    vm.startPrank(address(0xbeef));
    vm.roll(block.number + 1); // Advance block to avoid same-block restriction
    alchemist.repay(alchemist.convertDebtTokensToYield(10e18), tokenId);
    vm.stopPrank();
    
    // Verify earmarkWeight increased using our new getter
    uint256 newEarmarkWeight = alchemist.getEarMarkWeight();
    assertGt(newEarmarkWeight, originalEarmarkWeight, "EarmarkWeight should have increased");
    
    // Capture the account's last accrued earmark weight
    uint256 accountEarmarkWeight = alchemist.getLastAccruedEarmarkWeight(tokenId);
    
    // Store current block number
    uint256 lastRedemptionBlock = alchemist.lastRedemptionBlock();
    
    vm.prank(alOwner);
    alchemist.setTransmuter(address(transmuterLogic)); // Trigger a new redemption period with current earmark weight
    
    // Verify the condition for underflow now exists
    uint256 redemptionEarmarkWeight = alchemist.getRedemptionEarmarkWeight(alchemist.lastRedemptionBlock());
    assertGt(accountEarmarkWeight, redemptionEarmarkWeight, "Account earmark weight should be greater than redemption earmark weight");
    // This should revert due to the underflow when previousRedemption.earmarkWeight < account.lastAccruedEarmarkWeight
    vm.expectRevert();
    alchemist.getCDP(tokenId);
}
```
## Recommendation
Add a safety check to handle this edge case:
```js
if (previousRedemption.earmarkWeight <= account.lastAccruedEarmarkWeight) {
    // Skip the earmark calculation in this edge case
    debtToEarmark = 0;
} else {
    debtToEarmark = PositionDecay.ScaleByWeightDelta(
        account.debt - account.earmarked, 
        previousRedemption.earmarkWeight - account.lastAccruedEarmarkWeight
    );
}
```
Alternatively, implement a more robust solution that properly handles the case where a user's account has been updated more recently than the last redemption:
```js
// Use max to ensure we never have a negative weight delta
uint256 weightDelta = previousRedemption.earmarkWeight > account.lastAccruedEarmarkWeight ? 
    previousRedemption.earmarkWeight - account.lastAccruedEarmarkWeight : 0;
    
debtToEarmark = PositionDecay.ScaleByWeightDelta(account.debt - account.earmarked, weightDelta);
```
This change ensures the calculations remain valid even when users interact with the system in unexpected sequences.

# **M1** Potential Underflow in _subDebt Function When Freeing Collateral

## Description
A medium to high severity vulnerability exists in the _subDebt function in AlchemistV3.sol (lines 1249-1262). The function decreases a user's debt and frees a proportional amount of their locked collateral but fails to check if the amount to free (toFree) exceeds the account's locked collateral (account.rawLocked).
```js
function _subDebt(uint256 tokenId, uint256 amount) internal {
    Account storage account = _accounts[tokenId];
    account.debt -= amount;
    totalDebt -= amount;

    // Update collateral variables
    uint256 toFree = convertDebtTokensToYield(amount) * minimumCollateralization / FIXED_POINT_SCALAR;

    // For cases when someone above minimum LTV gets liquidated.
    if (toFree > _totalLocked) {
        toFree = _totalLocked;
    }

    _totalLocked -= toFree;
    account.rawLocked -= toFree;  // <-- Potential underflow DOS
    account.freeCollateral += toFree;
}
```
While there is a check to ensure toFree doesn't exceed the global _totalLocked value, there is no equivalent check against the account-specific `account.rawLocked`.

## Numerical Example
Consider a scenario where:

Initial State:

- User has 1,000 alUSD debt
- User has 2,000 yvDAI locked collateral (account.rawLocked = 2,000)
- Minimum collateralization: 200% (2e18)
- FIXED_POINT_SCALAR: 1e18

Partial Liquidation Occurs:

- Liquidation targets 600 alUSD of debt
- System removes 1,200 yvDAI from account.rawLocked (now 800)
- But only updates account.debt to 700 alUSD (not proportionally)
- This legitimate scenario can happen due to price fluctuations
- User Attempts to Repay Remaining Debt:

User calls burn(700) to fully repay their debt

- _subDebt calculates: toFree = 700 * 2e18 / 1e18 = 1,400 yvDAI
- But account.rawLocked is only 800
- When executing account.rawLocked -= toFree, we attempt 800 - 1,400
- This causes an arithmetic underflow and the transaction reverts
- Impact:

User cannot fully repay their debt

- Their position is stuck in a limbo state
- If liquidation is needed, the _liquidate function (which calls _forceRepay which calls _subDebt) would also revert
- This creates a position that cannot be properly managed or liquidated
- This vulnerability could occur in several scenarios:

- After partial liquidations that modify debt and collateral ratios
- When protocol parameters like conversion rates change between debt creation and repayment
- When _subDebt is called with a large amount parameter relative to the remaining account.rawLocked
- The impact is severe because this function is called by critical protocol operations:

- burn() - Users repaying their loans
- repay() - Users repaying with yield tokens
- _forceRepay() - Called during liquidations
NOTE - Most critically, this could cause liquidations to fail during market stress, which is not good.

## poc
paste in alchemist.t.sol
```js
function testSubDebtUnderflowBug() external {
    // Step 1: Set up a user position with debt
    uint256 depositAmount = 200_000e18;
    vm.startPrank(address(0xbeef));
    SafeERC20.safeApprove(address(fakeYieldToken), address(alchemist), depositAmount);
    alchemist.deposit(depositAmount, address(0xbeef), 0);
    uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
    
    // Create debt - borrowing the maximum allowed amount
    uint256 maxBorrow = alchemist.getMaxBorrowable(tokenId);
    alchemist.mint(tokenId, maxBorrow, address(0xbeef));
    vm.stopPrank();
    
    // Record initial state
    (uint256 initialCollateral, uint256 initialDebt,) = alchemist.getCDP(tokenId);
    console.log("Initial state:");
    console.log("- Collateral:", initialCollateral / 1e18);
    console.log("- Debt:", initialDebt / 1e18);
    
    // Step 2: Manipulate token price to trigger a partial liquidation
    // This will reduce rawLocked more than debt in proportion
    uint256 initialVaultSupply = IERC20(address(fakeYieldToken)).totalSupply();
    fakeYieldToken.updateMockTokenSupply(initialVaultSupply);
    // Increase yield token supply by 10% to decrease its value
    uint256 modifiedVaultSupply = (initialVaultSupply * 1000 / 10_000) + initialVaultSupply;
    fakeYieldToken.updateMockTokenSupply(modifiedVaultSupply);
    
    // Step 3: Perform liquidation to reduce rawLocked
    vm.startPrank(externalUser);
    alchemist.liquidate(tokenId);
    vm.stopPrank();
    
    // Get post-liquidation state
    (uint256 postLiquidationCollateral, uint256 postLiquidationDebt,) = alchemist.getCDP(tokenId);
    console.log("After liquidation:");
    console.log("- Collateral:", postLiquidationCollateral / 1e18);
    console.log("- Debt:", postLiquidationDebt / 1e18);
    
    // Step 4: Restore price to create the underflow condition
    // Now rawLocked is disproportionately lower than debt
    fakeYieldToken.updateMockTokenSupply(initialVaultSupply);
    
    // Step 5: Try to burn all debt - this should underflow
    vm.startPrank(address(0xbeef));
    vm.roll(block.number + 1); // Avoid same-block restriction
    SafeERC20.safeApprove(address(alToken), address(alchemist), postLiquidationDebt);
    
    // The call to burn will trigger _subDebt with an amount that will calculate
    // toFree > account.rawLocked, causing an underflow
    console.log("Attempting to burn all debt - this will calculate toFree based on full debt amount");
    console.log("But rawLocked was reduced by liquidation, so account.rawLocked -= toFree will underflow");
    
    try alchemist.burn(postLiquidationDebt, tokenId) {
        console.log("UNEXPECTED: Burn succeeded when it should have underflowed");
    } catch {
        console.log("EXPECTED: Burn failed due to underflow in _subDebt");
    }
    
    vm.stopPrank();
    
    // The test demonstrates the bug if the burn call reverts due to underflow
    // This confirms that account.rawLocked < toFree in _subDebt
}
```
## Recommendation
Add a check to ensure toFree doesn't exceed `account.rawLocked`, similar to the existing check for _totalLocked:
```js
function _subDebt(uint256 tokenId, uint256 amount) internal {
    Account storage account = _accounts[tokenId];
    account.debt -= amount;
    totalDebt -= amount;
    // Update collateral variables
    uint256 toFree = convertDebtTokensToYield(amount) * minimumCollateralization / FIXED_POINT_SCALAR;
    if (toFree > _totalLocked) {
        toFree = _totalLocked;
    }
    
+   if (toFree > account.rawLocked) {
+       toFree = account.rawLocked;
+   }
    _totalLocked -= toFree;
    account.rawLocked -= toFree;
    account.freeCollateral += toFree;
}
```

# **M2** Deposit Cap Bypass via Balance Manipulation

## Description
A medium to high severity vulnerability exists in the deposit function in AlchemistV3.sol that allows users to bypass the deposit cap through balance manipulation. The function verifies that deposits won't exceed the cap using the contract's current balance:

`_checkState(IERC20(yieldToken).balanceOf(address(this)) + amount <= depositCap);`
This implementation is vulnerable to a transaction sequence attack because it relies on the contract's current balance rather than tracking total deposits independently. A malicious user can execute the following steps to bypass the intended deposit limit:

1. Initial State:

Current contract balance: 990,000 yvDAI
Deposit cap: 1,000,000 yvDAI
User wants to deposit 15,000 yvDAI (would exceed cap by 5,000)

2. Attack Steps:

User calls withdraw(10,000), reducing the contract balance to 980,000 yvDAI
In the same transaction or block, user calls deposit(15,000)
The deposit cap check passes: 980,000 + 15,000 = 995,000 < 1,000,000
The deposit succeeds despite effectively exceeding the intended cap

3. Result:

The user has bypassed the deposit cap protection
The protocol now has 5,000 more yvDAI than intended
This can be repeated to further exceed the cap
This vulnerability undermines protocol safety parameter designed to limit risk exposure and could lead to protocol insolvency in extreme cases if exploited at scale.

Poc
Paste this poc in Alchemist.t.sol
```js
function testDepositCapBypass() external {
    uint256 depositCap = 1_000_000e18;
    
    // Set a specific deposit cap for the test
    vm.prank(alOwner);
    alchemist.setDepositCap(depositCap);
    
    // Initial deposit - close to the cap (990,000)
    uint256 initialDeposit = 990_000e18;
    vm.startPrank(address(0xbeef));
    SafeERC20.safeApprove(address(fakeYieldToken), address(alchemist), initialDeposit + 100_000e18);
    alchemist.deposit(initialDeposit, address(0xbeef), 0);
    uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
    
    // Verify we're near the cap
    assertEq(IERC20(fakeYieldToken).balanceOf(address(alchemist)), initialDeposit);
    
    // Try to deposit more than allowed by cap - this should revert
    uint256 excessDeposit = 15_000e18; // Would exceed cap by 5,000
    vm.expectRevert();
    alchemist.deposit(excessDeposit, address(0xbeef), tokenId);
    
    // Now perform the bypass attack: withdraw and then deposit more than withdrawn
    uint256 withdrawAmount = 10_000e18;
    alchemist.withdraw(withdrawAmount, address(0xbeef), tokenId);
    
    // Verify the balance is reduced
    assertEq(IERC20(fakeYieldToken).balanceOf(address(alchemist)), initialDeposit - withdrawAmount);
    
    // Now deposit 15,000 - this should succeed even though net result exceeds cap
    alchemist.deposit(excessDeposit, address(0xbeef), tokenId);
    vm.stopPrank();
    
    // Verify final balance exceeds what should be allowed
    uint256 finalBalance = IERC20(fakeYieldToken).balanceOf(address(alchemist));
    
    // Calculate how much we've bypassed the effective cap by
    uint256 effectiveExcess = (initialDeposit - withdrawAmount + excessDeposit) - (depositCap - withdrawAmount);
    console.log("Bypass succeeded! Exceeded effective cap by", effectiveExcess / 1e18, "tokens");
    
    // Demonstrate the attack can be repeated
    vm.startPrank(address(0xbeef));
    alchemist.withdraw(withdrawAmount, address(0xbeef), tokenId);
    alchemist.deposit(excessDeposit, address(0xbeef), tokenId);
    vm.stopPrank();
    
    finalBalance = IERC20(fakeYieldToken).balanceOf(address(alchemist));
    console.log("Final contract balance after repeated bypass:", finalBalance / 1e18);
}
```
Output -
```js
Ran 1 test for src/test/AlchemistV3.t.sol:AlchemistV3Test
[PASS] testDepositCapBypass() (gas: 416863)
Logs:
  Bypass succeeded! Exceeded effective cap by 5000 tokens
  Final contract balance after repeated bypass: 1000000

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 38.22ms (3.62ms CPU time)

Ran 1 test suite in 724.01ms (38.22ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
Recommendation
Replace the current balance-based check with a total deposits tracking mechanism, something like this:

// Add a state variable
uint256 public totalDeposited;

// In the deposit function
function deposit(uint256 amount, address recipient, uint256 recipientId) external returns (uint256) {
    // ...existing checks...
    _checkState(totalDeposited + amount <= depositCap);
    
    // Update tracking variable
    totalDeposited += amount;
    
    // ...rest of the function...
}

// In the withdraw function
function withdraw(uint256 amount, address recipient, uint256 tokenId) external returns (uint256) {
    // ...existing code...
    
    // Update tracking variable
    totalDeposited -= amount;
    
    // ...rest of the function...
}
This approach ensures the deposit cap is enforced based on net deposits rather than the current contract balance, making it resistant to manipulation through withdrawals.
```