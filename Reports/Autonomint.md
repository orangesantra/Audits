# **H 1** - User can put any arbitrary usdaPrice and usdtPrice while calling CDS.sol::redeemUSDT function.
- https://github.com/sherlock-audit/2024-11-autonomint-judging/issues/771

## Summary

`CDS.sol::redeemUSDT` is used to swap usda for usdt, takes the amount to swap and price of usda/ usdt as parameters. but the redeemer can use any arbitrary value of usdaPrice and usdtPrice, making the trade in his profit or drain out amount more than expected.

also the value of usdaPrice and usdtPrice isn't checked anywhere in `CDS.sol::redeemUSDT` and `CDSLib.sol::redeemUSDT` functions.

## Root Cause

passing usdaPrice and usdtPrice as parameters.

https://github.com/sherlock-audit/2024-11-autonomint/blob/main/Blockchain/Blockchian/contracts/Core_logic/CDS.sol#L506

## Internal pre-conditions
No response

## External pre-conditions
No response

## Attack Path
User will call CDS.sol::redeemUSDT passing usdaPrice very high (than actual marketPrice) and usdtPrice very low (than actual marketPrice).
```js
In CDSLib.sol::redeemUSDT the calculation will be as follow -
    uint128 usdtAmount = ((usdaPrice * usdaAmount) / usdtPrice);
    interfaces.treasury.approveTokens(IBorrowing.AssetName.TUSDT, address(interfaces.cds), usdtAmount);
    // Transfer usdt to user
    bool success = interfaces.usdt.transferFrom(
        address(interfaces.treasury),
        msg.sender,
        usdtAmount
    );
```
User will end up getting larger amount tha expected or if he very bad he can drain out all usdt from the treasory.

## Impact
User can take out funds more than expected.
USDT in treasory can be fully drained out.

## PoC
No response

## Mitigation
avoid passing usdaPrice and usdtPrice as parameters, even it's passed check it's legitimacy whether it's in market value or not.

# **H 2** - CDS.sol::updateDownsideProtected() can be called by anyone, with anyvalue changeing the state variable downsideProtected.
- https://github.com/sherlock-audit/2024-11-autonomint-judging/issues/847

## Summary
Anyone can call CDS.sol::updateDownsideProtected() and modify the state of blockchain, by altering downsideProtected.

## Root Cause
Not checking who's the caller of this function or implanting any kind of modifier to verify the caller.

https://github.com/sherlock-audit/2024-11-autonomint/blob/main/Blockchain/Blockchian/contracts/Core_logic/CDS.sol#L829

## Internal pre-conditions
No response

## External pre-conditions
No response

## Attack Path
downsideProtected is being used in _updateCurrentTotalCdsDepositedAmount() and _updateCurrentTotalCdsDepositedAmount() is being called in both withdraw() and deposit() functions.
```js
    function _updateCurrentTotalCdsDepositedAmount() private {
        if (downsideProtected > 0) {
            totalCdsDepositedAmount -= downsideProtected;
            totalCdsDepositedAmountWithOptionFees -= downsideProtected;
            downsideProtected = 0;
        }
    }
```
when `_updateCurrentTotalCdsDepositedAmount()` is executed it updates 2 another state variables -
totalCdsDepositedAmount and totalCdsDepositedAmountWithOptionFees

and also these 2 varibles are being used in deposit() and withdraw()function.

let's consider in `deposit()` function totalCdsDepositedAmount being updated as -
`totalCdsDepositedAmount = result.totalCdsDepositedAmount;`

But later on in `withdraw()` function `totalCdsDepositedAmount` being used for is manimpulated one, as a malicious user has called

```js
updateDownsideProtected. -
        withdrawResult = CDSLib.withdrawUser(
            WithdrawUserParams(
                cdsDepositDetails,
                omniChainData,
                optionFees,
                optionsFeesToGetFromOtherChain,
                returnAmount,
                withdrawResult.ethAmount,
                withdrawResult.usdaToTransfer,
                weETH_ExchangeRate,
                rsETH_ExchangeRate,
                fee.nativeFee
            ),
            Interfaces(
                treasury,
                globalVariables,
                usda,
                usdt,
                borrowing,
                CDSInterface(address(this))
            ),
            totalCdsDepositedAmount, // manipulated
            totalCdsDepositedAmountWithOptionFees,
            omniChainCDSLiqIndexToInfo
        );
```
## Impact
Can lead to loss of user funds to to manipulated values of totalCdsDepositedAmount and totalCdsDepositedAmountWithOptionFees

## PoC
No response

## Mitigation
Restrict the visibilty of CDS.sol::updateDownsideProtected() or check who's the caller of this function.

# **H 3** - A user won't be able to get his redeemed amount by calling borrowing.sol::redeemYields().
- https://github.com/sherlock-audit/2024-11-autonomint-judging/issues/921

## Summary
Code snippet of Borrowlib.sol::redeemYeild.

https://github.com/sherlock-audit/2024-11-autonomint/blob/main/Blockchain/Blockchian/contracts/lib/BorrowLib.sol#L978

```js
    function redeemYields(
        address user,
        uint128 aBondAmount,
        address usdaAddress,
        address abondAddress,
        address treasuryAddress,
        address borrow
    ) public returns (uint256) {
        // check abond amount is non zewro
        if (aBondAmount == 0) revert IBorrowing.Borrow_NeedsMoreThanZero();

        IABONDToken abond = IABONDToken(abondAddress);
        // get user abond state
        State memory userState = abond.userStates(user);
        // check user have enough abond
        if (aBondAmount > userState.aBondBalance) revert IBorrowing.Borrow_InsufficientBalance();

        ITreasury treasury = ITreasury(treasuryAddress);
        // calculate abond usda ratio
        uint128 usdaToAbondRatio = uint128((treasury.abondUSDaPool() * RATE_PRECISION) / abond.totalSupply());
        uint256 usdaToBurn = (usdaToAbondRatio * aBondAmount) / RATE_PRECISION;
        // update abondUsdaPool in treasury
        treasury.updateAbondUSDaPool(usdaToBurn, false);

        // calculate abond usda ratio from liquidation
        // @audit - reflection of prvious audit.
        uint128 usdaToAbondRatioLiq = uint128((treasury.usdaGainedFromLiquidation() * RATE_PRECISION) /abond.totalSupply());
        uint256 usdaToTransfer = (usdaToAbondRatioLiq * aBondAmount) / RATE_PRECISION;
        // update usdaGainedFromLiquidation in treasury
        treasury.updateUSDaGainedFromLiquidation(usdaToTransfer, false);

        // Burn the usda from treasury
        treasury.approveTokens(
            IBorrowing.AssetName.USDa,
            borrow,
            (usdaToBurn + usdaToTransfer)
        );

        IUSDa usda = IUSDa(usdaAddress);
        // burn the usda
        bool burned = usda.contractBurnFrom(address(treasury), usdaToBurn);
        if (!burned) revert IBorrowing.Borrow_BurnFailed();

        // @audit - due to this check which will always fail as usdaTotransfer will be as,treasury.usdaGainedFromLiquidation()
        // will always be 0, as usdagainedfromliquidation is not updated for positive value throughout the protocol.
        if (usdaToTransfer > 0) {
            // transfer usda to user
            bool transferred = usda.contractTransferFrom(
                address(treasury),
                user,
                usdaToTransfer
            );
            if (!transferred) revert IBorrowing.Borrow_TransferFailed();
        }
        // withdraw eth from ext protocol
        uint256 withdrawAmount = treasury.withdrawFromExternalProtocol(user,aBondAmount);

        //Burn the abond from user
        bool success = abond.burnFromUser(msg.sender, aBondAmount);
        if (!success) revert IBorrowing.Borrow_BurnFailed();

        return withdrawAmount;
    }
```
function flow -

User calls redeemYields, which will eventually call `BorrowLib.redeemYields.`

Inside `BorrowLib.sol::redeemYields` there is a code line which calculates the usdaToAbondRatioLiq-

`uint128 usdaToAbondRatioLiq = uint128((treasury.usdaGainedFromLiquidation() * RATE_PRECISION) /abond.totalSupply());`

`treasury.usdaGainedFromLiquidation()` is done to get the value of usdaGainedFromLiquidation state variable from treasury contract.

usdaGainedFromLiquidation can only be updated via. `Treasury.sol::updateUSDaGainedFromLiquidation()`
```js
    function updateUSDaGainedFromLiquidation(
        uint256 amount,
        bool operation
    ) external onlyCoreContracts {
        if (operation) {
            usdaGainedFromLiquidation += amount;
        } else {
            usdaGainedFromLiquidation -= amount;
        }
    }
```
The only way value of `usdaGainedFromLiquidation` could be greater than 0, if updateUSDaGainedFromLiquidation has been called in core contracts with bool operation as true. something like `treasury.updateUSDaGainedFromLiquidation(amount, true).`

But if we analyse through whole codeBase we will find that `updateUSDaGainedFromLiquidation()` haven't been called with true parameter, it means the state variable `usdaGainedFromLiquidation` is always 0.

It means `treasury.usdaGainedFromLiquidation()` will be 0 in the code line -

`uint128 usdaToAbondRatioLiq = uint128((treasury.usdaGainedFromLiquidation() * RATE_PRECISION) /abond.totalSupply());`

which means - usdaToAbondRatioLiq = 0 and usdaToTransfer = 0 as -
uint256 usdaToTransfer = (usdaToAbondRatioLiq * aBondAmount) / RATE_PRECISION;
And since usdaToTransfer = 0. The condition if (usdaToTransfer > 0) will never hit. hence user will not get any usda.

Also abond of function caller will be burned.

## Also to note -
treasury.updateUSDaGainedFromLiquidation(usdaToTransfer, false); will not cause any revert issue as 0 is being substarcted from 0.

## Root Cause
treasury.updateUSDaGainedFromLiquidation() not been called for true operation in entire codeBase.
leading to value of usdaGainedFromLiquidation never be greater than 0.

## Internal pre-conditions
No response

## External pre-conditions
No response

## Attack Path
No response

## Impact
The redeem function caller's ABOND will be burned, without user being getting any USDA.

## PoC
No response

## Mitigation
In deposit of `borrowing.sol` function add `treasury.updateUSDaGainedFromLiquidation(amount, true);`.

# **M 1** - when borrowing.sol::depositTokens is called, stale lastCumulativeRate is being passed to BorrowLib.deposit() function.

- https://github.com/sherlock-audit/2024-11-autonomint-judging/issues/820

Medium

when borrowing.sol::depositTokens is called, stale lastCumulativeRate is being passed to BorrowLib.deposit() function.
## Summary
When a user calls borrowing.sol::depositTokens it will call BorrowLib.deposit() with lastCumulativeRate as one of it's parameter, but since the lastCumulativeRate has been updated when the contract was deployed (@1). If a user calls borrowing.sol::depositTokens after very long time, the lastCumulativeRate being passed as parameter for BorrowLib.deposit() will be very old. This will lead to wrong totalNormalizedAmount calculation (@2).

@1
DeployBorrowing.s.sol

    contractsA.borrow.calculateCumulativeRate();
    contractsB.borrow.calculateCumulativeRate();
@2
borrowing.sol::depositTokens

    totalNormalizedAmount = BorrowLib.deposit(
        BorrowLibDeposit_Params(
            LTV,
            APR,
            lastCumulativeRate,
            totalNormalizedAmount,
            exchangeRate,
            ethPrice,
            lastEthprice
        ),
Also the same issue exist if the duration between 2 calls to borrowing.sol::depositTokens is very large.

## Root Cause
Use of stale lastCumulativeRate.

https://github.com/sherlock-audit/2024-11-autonomint/blob/main/Blockchain/Blockchian/contracts/Core_logic/borrowing.sol#L241

## Internal pre-conditions
No response

## External pre-conditions
No response

## Attack Path
No response

## Impact
totalNormalizedAmount is a state variable in borrowing.sol , wrong value assignment can lead to -

totalNormalizedAmount is being used in _withdraw() function of borrowing.sol, this can lead incorrect
withdrawal amount, code snippet @3. potential loss of user funds.
@3
borrowing.sol::_withdraw

    BorrowWithdraw_Result memory result = BorrowLib.withdraw(
        depositDetail,
        BorrowWithdraw_Params(
            toAddress,
            index,
            ethPrice,
            exchangeRate,
            withdrawTime,
            lastCumulativeRate,
            totalNormalizedAmount,
            bondRatio,
            collateralRemainingInWithdraw,
            collateralValueRemainingInWithdraw
        ),
        Interfaces(treasury, globalVariables, usda, abond, cds, options)
    );
## PoC
No response

## Mitigation
In borrowing.sol::depositToken call calculateCumulativeRate(); before BorrowLib.deposit()-
```js
    totalNormalizedAmount = BorrowLib.deposit(
        BorrowLibDeposit_Params(
            LTV,
            APR,
            lastCumulativeRate,
            totalNormalizedAmount,
            exchangeRate,
            ethPrice,
            lastEthprice
        ),
```
By calling calculateCumulativeRate(); early; lastCumulativeRate will be updated and the param being passed to BorrowLib.deposit() will be latest.
