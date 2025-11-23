# **H1** LendingP2P.sol::repayLoan() can be called by any one without the borrower consent.

## Description
Repository - hyperlend-p2p

`LendingP2P.sol::repayLoan()` should only be called by borrower of loan, not anyone. this can lead to repayment without borrower's will. which is totally unacceptable, because the borrower wants to pay the debt other day day but another user pays before borrower's wish, It can be verified in test file `4_repay.js` by pasting the below PoC. we can see that loan can be repaid by attacker.

## Proof of Concept
```js
it("can be repayed by an attacker without borrower consent", async function () {

        [,,,attacker] = await ethers.getSigners();

        const encodedLoan = encodeLoan(loan)
        await loanContract.connect(borrower).requestLoan(encodedLoan);

        await mockAsset.connect(borrower).transfer(lender.address, loan.assetAmount)
        await mockAsset.connect(borrower).transfer(attacker.address, loan.assetAmount)

        let balancesBefore = await recordBalances()

        expect(balancesBefore.borrower.asset).to.equal("999999980000000000000000000");
        expect(balancesBefore.borrower.collateral).to.equal("1000000000000000000000000000");
        expect(balancesBefore.lender.asset).to.equal(loan.assetAmount);
        expect(balancesBefore.lender.collateral).to.equal("0");

        await mockCollateral.connect(borrower).approve(loanContract.target, loan.collateralAmount)
        await mockAsset.connect(lender).approve(loanContract.target, loan.assetAmount)

        await loanContract.connect(lender).fillRequest(0);

        let balancesAfter = await recordBalances()

        expect(balancesAfter.borrower.asset).to.equal(balancesBefore.borrower.asset + loan.assetAmount);
        expect(balancesAfter.borrower.collateral).to.equal(balancesBefore.borrower.collateral - loan.collateralAmount);

        expect(balancesAfter.lender.asset).to.equal(balancesBefore.lender.asset - loan.assetAmount);
        expect(balancesAfter.lender.collateral).to.equal("0");

        expect(balancesAfter.contract.asset).to.equal("0");
        expect(balancesAfter.contract.collateral).to.equal(loan.collateralAmount);

        const storedLoan = await loanContract.loans(0);

        expect(storedLoan.borrower).to.equal(loan.borrower);
        expect(storedLoan.lender).to.equal(lender.address);
        expect(storedLoan.asset).to.equal(loan.asset);
        expect(storedLoan.collateral).to.equal(loan.collateral);

        expect(storedLoan.assetAmount).to.equal(loan.assetAmount);
        expect(storedLoan.repaymentAmount).to.equal(loan.repaymentAmount);
        expect(storedLoan.collateralAmount).to.equal(loan.collateralAmount);

        expect(storedLoan.startTimestamp).to.not.equal(0);
        expect(storedLoan.duration).to.equal(loan.duration);

        expect(storedLoan.liquidation.isLiquidatable).to.equal(loan.liquidation.isLiquidatable);
        expect(storedLoan.liquidation.liquidationThreshold).to.equal(loan.liquidation.liquidationThreshold);
        expect(storedLoan.liquidation.assetOracle).to.equal(loan.liquidation.assetOracle);
        expect(storedLoan.liquidation.collateralOracle).to.equal(loan.liquidation.collateralOracle);

        expect(storedLoan.status).to.equal(2);

        let fee = (loan.repaymentAmount - loan.assetAmount) * ethers.toBigInt(2000) / ethers.toBigInt(10000);

        //repay loan
        await mockAsset.connect(borrower).approve(loanContract.target, loan.repaymentAmount)
   @--> await expect(loanContract.connect(attacker).repayLoan(0))
            .to.emit(loanContract, "LoanRepaid")
            .withArgs(0, borrower.address, lender.address)
            .to.emit(loanContract, "ProtocolRevenue")
            .withArgs(0, mockAsset.target, fee)

        let balancesAfterRepay = await recordBalances()

        expect(balancesAfterRepay.lender.asset).to.equal(balancesBefore.lender.asset - loan.assetAmount + loan.repaymentAmount - fee);
        expect(balancesAfterRepay.lender.collateral).to.equal("0");

        expect(balancesAfterRepay.contract.asset).to.equal("0");
        expect(balancesAfterRepay.contract.collateral).to.equal("0");

        expect(balancesAfterRepay.deployer.asset).to.equal(fee);

        const storedLoanAfterRepay = await loanContract.loans(0);
        expect(storedLoanAfterRepay.status).to.equal(3);
    });
```
## Recommendation
Putting this check -
```js
+   require(msg.sender == _loan.borrower);
```

# **L1** LendingP2P.sol::requestLoan() can be called without checking weather the collateral amount is zero or not.

## Description
Repository - hyperlend-p2p

The `requestLoan()` can be called even if the collateral amount is 0. There is no check to ensure that collateral amount must not be equal to 0. This will cause waste of storage allocation and gas, as it updates two state variables loans[loanLength] and loanLength += 1;.
```js
function requestLoan(bytes memory _encodedLoan) external nonReentrant {
        Loan memory loan = abi.decode(_encodedLoan, (Loan));

        require(loan.borrower == msg.sender, "borrower != msg.sender");
        require(loan.repaymentAmount > loan.assetAmount, "amount <= repayment");
        require(loan.asset != loan.collateral, "asset == collateral");
        require(loan.liquidation.liquidationThreshold <= 10000, "liq threshold > max bps");

        //since users can use any address (even non-standard contracts), verify that the decimals function exists
        require(IERC20Metadata(loan.asset).decimals() >= 0, "invalid decimals");
        require(IERC20Metadata(loan.collateral).decimals() >= 0, "invalid decimals");

        if (loan.liquidation.isLiquidatable){
            uint8 assetOracleDecimals = AggregatorInterface(loan.liquidation.assetOracle).decimals();
            uint8 collateralOracleDecimals = AggregatorInterface(loan.liquidation.collateralOracle).decimals();
            require(assetOracleDecimals == collateralOracleDecimals, "oracle decimals mismatch");
        }

        loan.createdTimestamp = uint64(block.timestamp);
        loan.startTimestamp = 0;
        loan.status = Status.Pending;

        loans[loanLength] = loan;
        loanLength += 1;

        emit LoanRequested(loanLength - 1, msg.sender);
    }
```
## Recommendation
use a require check to ensure that collateral amount is not zero.

+    require(loan.collateralAmount > 0, "Collateral amount is 0");

# **L2** In LendingP2P.sol::repayLoan() the protocolFee will be zero, if _loan.repaymentAmount - _loan.assetAmount < 5

## Description
Repository - hyperlend-p2p

`LendingP2P.sol::repayLoan()` if the protocolFee will be zero, that is possible in case _loan.repaymentAmount - _loan.assetAmount < 5 then it cause following problems -

As,
```js
uint256 protocolFee = (_loan.repaymentAmount - _loan.assetAmount) * PROTOCOL_FEE / 10000;
uint256 protocolFee = (_loan.repaymentAmount - _loan.assetAmount) * 2000 / 10000;
uint256 protocolFee = amountlessthan(10000) / 10000;
uint256 protocolFee = 0
protocol fee = 0, which means protocol will end up having nothing.
```
some ERC20s, revert for amount being 0 in transfer or transferFrom function, leading to DOS.
## Recommendation
In requesting loan function - LendingP2P.sol::requestLoan(), apply the check,

# **L3** In LendeingP2P.sol liquidator gets less collateral amount ,if LIQUIDATOR_BONUS_BPS, PROTOCOL_LIQUIDATION_FEE

## Description
**Repository - hyperlend-p2p**

By default values are -

LIQUIDATOR_BONUS_BPS = 100 PROTOCOL_LIQUIDATION_FEE = 20

suppose a loan is active and also in liquidation state. so as per calculation in -
```js
function _liquidate(uint256 loanId) internal {
        Loan memory _loan = loans[loanId];
        
        uint256 liquidatorBonus = _loan.collateralAmount * LIQUIDATOR_BONUS_BPS / 10000;
        uint256 protocolFee = _loan.collateralAmount * PROTOCOL_LIQUIDATION_FEE / 10000;
        uint256 lenderAmount = _loan.collateralAmount - liquidatorBonus - protocolFee;

        loans[loanId].status = Status.Liquidated;
        
        IERC20(_loan.collateral).safeTransfer(_loan.lender, lenderAmount);
        IERC20(_loan.collateral).safeTransfer(msg.sender, liquidatorBonus);
        IERC20(_loan.collateral).safeTransfer(feeCollector, protocolFee);

        emit LoanLiquidated(loanId);
        emit ProtocolRevenue(loanId, _loan.collateral, protocolFee);
    }
```
Let collateralAmount is 100 ethers.

Apply above formula we get -

liquidatorBonus = 100100/10000 = 1 ethers protocolFee = 10020/10000 = 0.2 ethers lenderAmount = 100 - 1 - 0.2 = 98.8 ethers

Now suppose admin increases the values as -

LIQUIDATOR_BONUS_BPS = 900 PROTOCOL_LIQUIDATION_FEE = 400

New calculation, keeping collateral amount same(100 ethers).

newliquidatorBonus = 100900/10000 = 9 ethers newprotocolFee = 100400/10000 = 4 ethers newlenderAmount = 100 - 9 - 4 = 87 ethers

Loss of lender's collateral amount = lenderAmount - newlenderAmount = 98.8 - 87 = 11.8 ethers.

Tagging this issue as medium because unintentional loss of 11.8 ethers is a big deal, as lender wasn't aware that the new changes can be applied to ongoing or active loan status.

This can be verified by pasting the below code snippet in test file 5_liquidate.js.

## Proof of Concept
```js
it("should liquidate a valid liquidatable loan, and lender gets less collateral amount as expected", async function () {
        const encodedLoan = encodeLoan(loan)
        await loanContract.connect(borrower).requestLoan(encodedLoan);

        await mockAsset.connect(borrower).transfer(lender.address, loan.assetAmount)

        let balancesBefore = await recordBalances()

        expect(balancesBefore.borrower.asset).to.equal("999999990000000000000000000");
        expect(balancesBefore.borrower.collateral).to.equal("1000000000000000000000000000");
        expect(balancesBefore.lender.asset).to.equal(loan.assetAmount);
        expect(balancesBefore.lender.collateral).to.equal("0");

        await mockCollateral.connect(borrower).approve(loanContract.target, loan.collateralAmount)
        await mockAsset.connect(lender).approve(loanContract.target, loan.assetAmount)

        await loanContract.connect(lender).fillRequest(0);

        let balancesAfter = await recordBalances()

        expect(balancesAfter.borrower.asset).to.equal(balancesBefore.borrower.asset + loan.assetAmount);
        expect(balancesAfter.borrower.collateral).to.equal(balancesBefore.borrower.collateral - loan.collateralAmount);

        expect(balancesAfter.lender.asset).to.equal(balancesBefore.lender.asset - loan.assetAmount);
        expect(balancesAfter.lender.collateral).to.equal("0");

        expect(balancesAfter.contract.asset).to.equal("0");
        expect(balancesAfter.contract.collateral).to.equal(loan.collateralAmount);

        const storedLoan = await loanContract.loans(0);

        expect(storedLoan.borrower).to.equal(loan.borrower);
        expect(storedLoan.lender).to.equal(lender.address);
        expect(storedLoan.asset).to.equal(loan.asset);
        expect(storedLoan.collateral).to.equal(loan.collateral);

        expect(storedLoan.assetAmount).to.equal(loan.assetAmount);
        expect(storedLoan.repaymentAmount).to.equal(loan.repaymentAmount);
        expect(storedLoan.collateralAmount).to.equal(loan.collateralAmount);

        expect(storedLoan.startTimestamp).to.not.equal(0);
        expect(storedLoan.duration).to.equal(loan.duration);

        expect(storedLoan.liquidation.isLiquidatable).to.equal(loan.liquidation.isLiquidatable);
        expect(storedLoan.liquidation.liquidationThreshold).to.equal(loan.liquidation.liquidationThreshold);
        expect(storedLoan.liquidation.assetOracle).to.equal(loan.liquidation.assetOracle);
        expect(storedLoan.liquidation.collateralOracle).to.equal(loan.liquidation.collateralOracle);

        expect(storedLoan.status).to.equal(2);

        //make loan liquidatable
        await aggregatorAsset.connect(deployer).setAnswer(250000000000);
     
        // liquidate loan
        await expect(loanContract.connect(liquidator).liquidateLoan(0))
            .to.emit(loanContract, "LoanLiquidated")
            .withArgs(0)

        let balancesAfterLiquidate = await recordBalances()

        let liquidatorBonus = loan.collateralAmount * ethers.toBigInt(100) / ethers.toBigInt(10000);
        let protocolFee = loan.collateralAmount * ethers.toBigInt(20) / ethers.toBigInt(10000);
        let lenderAmount = loan.collateralAmount - liquidatorBonus - protocolFee;
        let temp_loan_collateral_amount = loan.collateralAmount;

        let lb = await loanContract.LIQUIDATOR_BONUS_BPS();
        let pf = await loanContract.PROTOCOL_LIQUIDATION_FEE();
        let la = temp_loan_collateral_amount - lb - pf;

        console.log("ADMIN UPDATES THE LIQUIDATOR_BONUS_BPS and PROTOCOL_FEE_BPS");

        await loanContract.connect(deployer).setLiquidationConfig(900, 400);

        let newliquidatorBonus = await loanContract.LIQUIDATOR_BONUS_BPS();
        let newprotocolFee = await loanContract.PROTOCOL_LIQUIDATION_FEE();
        let newlenderAmount = temp_loan_collateral_amount - newliquidatorBonus - newprotocolFee;

        console.log("Lender's loss", la - newlenderAmount);

        expect(balancesAfterLiquidate.borrower.collateral).to.equal(balancesBefore.borrower.collateral - loan.collateralAmount);
        expect(balancesAfterLiquidate.contract.collateral).to.equal("0");

        console.log("lender new collatral balance", newlenderAmount);
        console.log("lender expected collatral balance", la);

        const storedLoanAfterLiquidate = await loanContract.loans(0);
        expect(storedLoanAfterLiquidate.status).to.equal(4);
    });
```