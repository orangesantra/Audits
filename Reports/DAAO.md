# **H1**- Daao.sol::finalizeFundraising can be called by anyone passing any lowerTick and upperTick value
## Summary
In Daao.sol::finalizeFundraising there isn't any access control modifer, like owner or protocol admin, it means this can be called by anyone, They can put any arbitrary lowertick and uppertick for pool, that's not expected.

## Recommendation (optional)
put access modifier in finalizefundraising function.

# **H2**- Not validating the caller of CLPoolRouter.sol::uniswapV3SwapCallback.
## Summary
It is expected that uniswapV3SwapCallback should be triggered by AddLiquidity manager (not in current protocol), though the sender is decoded, but it's not validated wheather it's legit or not.

If the sender is not legit, he can transfer any token with any amount to msg.sender (swapper/ trader).
```js
function uniswapV3SwapCallback(
        int256 amount0Delta,
        int256 amount1Delta,
        bytes calldata data
    ) external override {
        address sender = abi.decode(data, (address));
        if (amount0Delta > 0) {
            IERC20Minimal(ICLPool(msg.sender).token0()).transferFrom(
                sender,
                msg.sender,
                uint256(amount0Delta)
            );
        } else if (amount1Delta > 0) {
            IERC20Minimal(ICLPool(msg.sender).token1()).transferFrom(
                sender,
                msg.sender,
                uint256(amount1Delta)
            );
        }
    }
```
## Impact Explanation
Swapper can be tricked with illegal sender.

# **M1**- In Daao.sol::contribute()non-whitlisted user will never be able tocontribute.

## Summary
It is expected that non whitlisted users able to contribute if maxWhitelistAmount = 0 and maxPublicContributionAmount > 0, but even these 2 conditions hit, non-whitlist users will never be able to contribute due to require(userTier != WhitelistTier.None, "Not whitelisted");-
```js
function contribute() public payable nonReentrant {
        require(!goalReached, "Goal already reached");
        require(block.timestamp < fundraisingDeadline, "Deadline hit");
        require(msg.value > 0, "Contribution must be greater than 0");

        // Must be whitelisted
        WhitelistTier userTier = userTiers[msg.sender];

@->     require(userTier != WhitelistTier.None, "Not whitelisted");
        // Contribution must boolow teir limit
        uint256 userLimit = tierLimits[userTier];
        require(
            contributions[msg.sender] + msg.value <= userLimit,
            "Exceeding tier limit"
        );
        // $doubt - what if maxwhitelistAmount and maxPublicContributionAmount are both >0, then else condition will never hit.
        if (maxWhitelistAmount > 0) {
            require(
                contributions[msg.sender] + msg.value <= maxWhitelistAmount,
                "Exceeding maxWhitelistAmount"
            );
        } else if (maxPublicContributionAmount > 0) {
            require(
                contributions[msg.sender] + msg.value <=
                    maxPublicContributionAmount,
                "Exceeding maxPublicContributionAmount"
            );
        }
```
## Impact Explanation
Non whitlisted users will never be able to contribute.

## Recommendation (optional)
do the check require(userTier != WhitelistTier.None, "Not whitelisted"); only for whitelisted user.

# **L1** No slippage protection in minting params.

## Summary
for minting params the, minamount0 and minamount1 both set to 0, which lead to minitng of pool position with 0 liquidity, in case market is very low for daao token and MODE token.
```js
INonfungiblePositionManager.MintParams
        memory params = INonfungiblePositionManager.MintParams(
            token0,
            token1,
            TICKING_SPACE,
            initialTick,
            upperTick,
            amountToken0ForLP
            amountToken1ForLP,
            0,
            0,
            address(this),
            block.timestamp,
            sqrtPriceX96
        );
```
## Impact Explanation
This can lead to creation and locking of NFT which represents a pool with 0 liquidity, if this case arises then liquidity providers will not earn any incentive, on locking his asset to Lockfactory.

## Recommendation (optional)
Add non-zero amount0 and amount1

# **I1** An attacker can drain out all the tokenOut balance of CLPoolRouter.sol contract

## Summary
```js
function getSwapResult(
        address pool,
        bool zeroForOne,
        int256 amountSpecified,
        uint160 sqrtPriceLimitX96
    )
        external
        returns (
            int256 amount0Delta,
            int256 amount1Delta,
            uint160 nextSqrtRatio
        )
    {
 
        address tokenIn = zeroForOne
            ? ICLPool(pool).token0()
            : ICLPool(pool).token1();
        address tokenOut = zeroForOne
            ? ICLPool(pool).token1()
            : ICLPool(pool).token0();
        uint256 amount = uint256(
            amountSpecified > 0 ? amountSpecified : -amountSpecified
        );

        _handleApproval(tokenIn, amount);

        IERC20Minimal(tokenIn).transferFrom(msg.sender, pool, amount);

        (amount0Delta, amount1Delta) = ICLPool(pool).swap(
            address(this),
            zeroForOne,
            amountSpecified,
            sqrtPriceLimitX96,
            abi.encode(msg.sender)
        );

        (nextSqrtRatio, , , , , ) = ICLPool(pool).slot0();
        uint256 outputAmount;
        if (zeroForOne) {
            require(amount1Delta < 0, "Invalid amount1Delta");
            outputAmount = uint256(-amount1Delta);
        } else {
            require(amount0Delta < 0, "Invalid amount0Delta");
            outputAmount = uint256(-amount0Delta);
        }

        if (outputAmount > 0) {
            uint256 poolBalance = IERC20Minimal(tokenOut).balanceOf(
                address(this)
            );
            require(
                poolBalance >= outputAmount,
                "Insufficient pool balance for output"
            );
@->         IERC20Minimal(tokenOut).transfer(msg.sender, outputAmount);
        }

        emit SwapExecuted(
            pool,
            msg.sender,
            zeroForOne,
            amountSpecified,
            sqrtPriceLimitX96,
            amount0Delta,
            amount1Delta,
            outputAmount
        );
    }
```
address of pool is not checked wheather it's a legit pool or malicious one, also the tokenOut is send from this contract, which can be very dangerous.

## Finding Description
Suppose zeroToOne is true.
Now suppose an attacker creates an pool with an maleicious tokenX and token1.
He transfers that pool address in the getSwapResult function.
so tokenIn = tokenX and tokenOut = token1.
Now tokenX will be transfered to the pool via.
IERC20Minimal(tokenIn).transferFrom(msg.sender, pool, amount);
There is futher other operation, but the point of consideration is this part -
```js
if (outputAmount > 0) {
            uint256 poolBalance = IERC20Minimal(tokenOut).balanceOf(
                address(this)
            );
            require(
                poolBalance >= outputAmount,
                "Insufficient pool balance for output"
            );
@->         IERC20Minimal(tokenOut).transfer(msg.sender, outputAmount);
        }
```
If outputAmount is 0, and also `poolBalance >= outputAmount` it means token1 that is present in the contract will be tranfered to attacker.
## Impact Explanation
User can put fake tokens (tokenX), and get legit tokens (token1) in back.

# **I2** wrong approval in CLPoolRouter.sol::_handleApproval.

## Summary
```js
function _handleApproval(address token, uint256 amount) internal {
        IERC20Minimal tokenContract = IERC20Minimal(token);
        uint256 currentAllowance = tokenContract.allowance(
            msg.sender,
            address(this)
        );
        // $mynotes - the whole idea is if the current allowance (permission to this contract to spend is
        // less tha amount needed, then first approve to zero then approve to new amount).
        // $doubt - isn't the contract approving itself ?
        if (currentAllowance < amount) {
            if (currentAllowance > 0) {
                
                tokenContract.approve(address(this), 0);
            }

            require(
                tokenContract.approve(address(this), amount),
                "Approval failed"
            );

            emit ApprovalHandled(token, msg.sender, amount);
        }
    }
```
In the above snippet there is code line (below), the intention behind is to allow users to approve the CLPoolRouter.sol, for funds to spend, on there behalf.
```js
if (currentAllowance > 0) {
       
        tokenContract.approve(address(this), 0);
    }

    require(
        tokenContract.approve(address(this), amount),
        "Approval failed"
    );
```
But the problem here is that this contract is approving itself, which is incorrect as msg.sender from tokenContract.approve() perspective will be CLPoolRouter.sol not the caller of getSwapResult().

## Impact Explanation
Wrong approval, can lead to DOS and fund loss to users.

## Recommendation (optional)
implement a functionality so that user is approving the tokencontract instead of contract itself.

# **I3** Non whitelister won't be able to conteibute if maxWhitelistAMount and maxPublicContribution amount both greater than 0.
## Summary
It is possible that maxWhitelistAMount and maxPublicContribution amount both greater than 0, in thatt case else if condition will not execute. hense leading to DOS for non-whitelisted members.
```js
if (maxWhitelistAmount > 0) {
            require(
                contributions[msg.sender] + msg.value <= maxWhitelistAmount,
                "Exceeding maxWhitelistAmount"
            );
        } else if (maxPublicContributionAmount > 0) {
            require(
                contributions[msg.sender] + msg.value <=
                    maxPublicContributionAmount,
                "Exceeding maxPublicContributionAmount"
            );
        }
```
## Impact Explanation
DOS for non whitelisted members in case, both `maxWhitelistAMount` and `maxPublicContribution` both non zero.

# **I4** Daao.sol::finalizeFundraising will always revert due to ownable access modifier in token contract.

## Summary
```js
function finalizeFundraising(int24 initialTick, int24 upperTick) external {
        require(goalReached, "Fundraising goal not reached");
        require(!fundraisingFinalized, "DAO tokens already minted");
        require(daoToken != address(0), "Token not set");
        emit DebugLog("Starting finalizeFundraising");
        DaaoToken token = DaaoToken(daoToken);
        daoToken = address(token);

        // Mint and distribute tokens to all contributors
        for (uint256 i = 0; i < contributors.length; i++) {
            address contributor = contributors[i];
            uint256 contribution = contributions[contributor];
            uint256 tokensToMint = (contribution * SUPPLY_TO_FUNDRAISERS) / totalRaised;

            emit MintDetails(contributor, tokensToMint);
@->         token.mint(contributor, tokensToMint);
        }

        // ADD THE NEW CODE RIGHT HERE, AFTER TOKEN DISTRIBUTION BUT BEFORE POOL CREATION
        uint256 totalCollected = address(this).balance;
        uint256 amountForLP = (totalCollected * LP_PERCENTAGE) / 100;
        uint256 amountForTreasury = totalCollected - amountForLP;
        // Transfer treasury amount to owner (We are left with LP amount)
        (bool success, ) = owner().call{value: amountForTreasury}("");
        require(success, "Treasury transfer failed");

        emit FundraisingFinalized(true);
        fundraisingFinalized = true;

        int24 iprice = -13862;
        uint160 sqrtPriceX96 = TickMath.getSqrtRatioAtTick(iprice);
        emit DebugLog("Calculated sqrtPriceX96");

        uint256 amountToken0ForLP;
        uint256 amountToken1ForLP;

        if (daoToken < MODE) {
            token0 = daoToken;
            token1 = MODE;
            // 4:1 mapping (Token Ratio in pool) == for 1 WETH, 4 of our token will be paired
            amountToken0ForLP = 4 * amountForLP;
            amountToken1ForLP = amountForLP;
        } else {
            token0 = MODE;
            token1 = daoToken;
            // 4:1 mapping (Token Ratio in pool) == for 1 WETH, 4 of our token will be paired
            amountToken0ForLP = amountForLP;
            amountToken1ForLP = 4 * amountForLP;
        }

@->     token.mint(address(this), 4 * amountForLP);

        INonfungiblePositionManager.MintParams
            memory params = INonfungiblePositionManager.MintParams(
                token0,
                token1,
                TICKING_SPACE,
                initialTick,
                upperTick,
                amountToken0ForLP, // desired to send to pool
                amountToken1ForLP, // desired to send to pool
                0, // minimum amount to send to pool.
                0, // minimum amount to send to pool.
                address(this), // recient of nft
                block.timestamp,
                sqrtPriceX96
            );

@->     token.renounceOwnership();

        IERC20(token0).approve(address(POSITION_MANAGER), amountToken0ForLP);
        IERC20(token1).approve(address(POSITION_MANAGER), amountToken1ForLP);
        emit DebugLog("Minted additional tokens for LP");

        (uint256 tokenId, , , ) = POSITION_MANAGER.mint(params);
        emit LPTokenMinted(tokenId);

        // Deploy the liquidity locker
        address lockerAddress = liquidityLockerFactory.deploy(
            address(POSITION_MANAGER),
            owner(),
            uint64(fundExpiry),
            tokenId,
            lpFeesCut,
            address(this)
        );
        emit LockerDeployed(lockerAddress);
        POSITION_MANAGER.safeTransferFrom(
            address(this),
            lockerAddress,
            tokenId
        );
        emit TokenTransferredToLocker(tokenId, lockerAddress);

        // Initialize the locker
        ILocker(lockerAddress).initializer(tokenId);
        emit LockerInitialized(tokenId);

        liquidityLocker = lockerAddress;
        emit DebugLog("Finalize fundraising complete");
    }
```
The `DaaoToken.sol` is Ownable and using onlyOwner modifier, which means mint function is only called by deployer of daaotoken, but since `Daao.sol` is't deploying daaotoken, which means `Daao.sol` isn't the owner of this contract and calling mint function from dao will lead to reverting of finalizeFundraising everytime, also same goes with `token.renounceOwnership();` it can only be called by deployer or owner of `DaaoToken.sol`.
```js
function mint(address _to, uint256 _amount) external onlyOwner {
        _mint(_to, _amount);
    }
function renounceOwnership() public virtual onlyOwner {
        _transferOwnership(address(0));
    }
```
## Impact Explanation
finalizeFundraising reverting everytime

## Recommendation (optional)
Implement a functionality so that `Daao.sol` can deploy/create daaotoken.