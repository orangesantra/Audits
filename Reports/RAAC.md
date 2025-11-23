# Core Contracts - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. A user can be frontrunned, while claiming reward from gauge.](#H-01)
    - ### [H-02. A user can re-update his userBoost storage struct, with different values by calling `delegateBoost()` function.](#H-02)
    - ### [H-03. No incentive or enforcement for users to call `FeeCollector.sol::collectFee()` function.](#H-03)
    - ### [H-04. Performance share is not distributed to veRAAC holders in `GaugeController.sol::distributeRevenue()`.](#H-04)
    - ### [H-05. Incorrect logic for gauge weight update.](#H-05)
    - ### [H-06. Any user can claim reward from gauge, even though user not staked anything.](#H-06)
    - ### [H-07. Scaling done twice, in transfer and transferFrom function in `RToken.sol`.](#H-07)
    - ### [H-08. An user can loose funds, if he calls `veRACC.sol::lock()` more than once during lock duration.](#H-08)
    - ### [H-09. User will always get 0 reward on calling `FeeCollector.sol::claimRewards`.](#H-09)
    - ### [H-10. User fund can be stuck if totalDistributed increases at very slow amount and user VotingPower increases very fast.](#H-10)
- ## Medium Risk Findings
    - ### [M-01. Average weight will always be 0, whenever period update occurs via `BaseGuage::updatePeriod()` function.](#M-01)
    - ### [M-02. re-updating `lastUpdateTime` in `BaseGauge.sol::notifyRewardAmount()` could be problamatic.](#M-02)
    - ### [M-03. Anyone can call `GauageController.sol::distributeRewards()` function.](#M-03)
    - ### [M-04. On boost delegation removal, updating to poolBoost storage struct isn't happening.](#M-04)
    - ### [M-05. Pool boost working supply not updated correctly, in `BoostController.sol::updateUserBoost`.](#M-05)
    - ### [M-06. An user can unethically liquidated even his healthFactor is in safe zone.](#M-06)
    - ### [M-07. Token validation isn't done in `Treasory.sol::deposit()`.](#M-07)
    - ### [M-08. Staleness of NFT price is not checked.](#M-08)
- ## Low Risk Findings
    - ### [L-01. Extra minting of raacTokens to stability pool; if `lastUpdateBlock` is reset to less value than earlier, by calling setLastUpdateBlock, in `RAACMinter.sol`.](#L-01)
    - ### [L-02. Griefing attack via `LendingPool::updateState()` function.](#L-02)
    - ### [L-03. Boost delegation to same address cannot be repeated again.](#L-03)
    - ### [L-04. crvUSD depositors not getting their accrued interest after withdrawel from LendingPool.](#L-04)
    - ### [L-05. There is no function in Controller to call `setWeeklyEmission()`.](#L-05)
    - ### [L-06. zero slippage in `LendingPool.sol::_withdrawFromVault()` can lead to DOS.](#L-06)
    - ### [L-07. An attacker can call `veRAACToken.sol::recordVote()` on behalf of voter, leading to unintended vote to a proposal.](#L-07)
    - ### [L-08. Incorrect initialization of minBoost or maxBoost in `BaseGuage` contract's constructor.](#L-08)
    - ### [L-09. `FeeCollector.sol::updateFeeType` will not work for all fee types.](#L-09)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: Regnum Aurum Acquisition Corp

### Dates: Feb 3rd, 2025 - Feb 24th, 2025

[See more contest details here](https://codehawks.cyfrin.io/c/2025-02-raac)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 14
- Medium: 18
- Low: 9


# High Risk Findings

## <a id='H-01'></a>H-01. A user can be frontrunned, while claiming reward from gauge.            



## Summary
Contract - `BaseGauge.sol`

**function_flow**
when user claims his reward.

`getReward()` -> `updateReward()` -> `_updateReward()` -> `earned()` -> `getRewardPerToken()`

```js
    function getReward() external virtual nonReentrant whenNotPaused updateReward(msg.sender) {
        if (block.timestamp - lastClaimTime[msg.sender] < MIN_CLAIM_INTERVAL) {
            revert ClaimTooFrequent();
        }
        
        lastClaimTime[msg.sender] = block.timestamp;
        UserState storage state = userStates[msg.sender];
        uint256 reward = state.rewards; // $mynotes - the reward decided at the moment user deposited his staking token. 
        
        if (reward > 0) {
            state.rewards = 0;
            
            uint256 balance = rewardToken.balanceOf(address(this));
            if (reward > balance) {
                revert InsufficientBalance();
            }
            
            rewardToken.safeTransfer(msg.sender, reward);
            emit RewardPaid(msg.sender, reward);
        }
    }
```
```js
    modifier updateReward(address account) {
        _updateReward(account);
        _;
    }
```
```js
    function _updateReward(address account) internal {
        rewardPerTokenStored = getRewardPerToken();
        lastUpdateTime = lastTimeRewardApplicable();

        if (account != address(0)) {
            UserState storage state = userStates[account];
            state.rewards = earned(account);

            state.rewardPerTokenPaid = rewardPerTokenStored;
            state.lastUpdateTime = block.timestamp;
            emit RewardUpdated(account, state.rewards);
        }
    }
```

```js
    function earned(address account) public view returns (uint256) {
        return (getUserWeight(account) * 
        // here - start again
            (getRewardPerToken() - userStates[account].rewardPerTokenPaid) / 1e18
        ) + userStates[account].rewards;
    }
```

```js
    function getRewardPerToken() public view returns (uint256) {
        if (totalSupply() == 0) {
            return rewardPerTokenStored;
        }

        return rewardPerTokenStored + (
@->         (lastTimeRewardApplicable() - lastUpdateTime) * rewardRate * 1e18 / totalSupply()
        );
    }
```

## Vulnerability Details

1. the return value in `getRewardPerToken()` is -

```js
    return rewardPerTokenStored + (
        (lastTimeRewardApplicable() - lastUpdateTime) * rewardRate * 1e18 / totalSupply()
    );
```
2. This can be problematic.

3. Suppose a malicious user leveraging flashloan, deposits very high amount of staking tokens via `stake()` function, just before a claimer is claiming his reward. (basically frontrunning)

```js
    function stake(uint256 amount) external nonReentrant updateReward(msg.sender) {
        if (amount == 0) revert InvalidAmount();
@->     _totalSupply += amount;
        _balances[msg.sender] += amount;
        stakingToken.safeTransferFrom(msg.sender, address(this), amount);
        emit Staked(msg.sender, amount);
    }
```
4. This will increase `_totalSupply` to very large value, means `totalSupply()` will return very large value as :-
```js
    function totalSupply() public view virtual returns (uint256) {
        return _totalSupply;
    }
```
5. subsequently, `(lastTimeRewardApplicable() - lastUpdateTime) * rewardRate * 1e18 / totalSupply()` will become very less.

6. Leading to loss of reward claim value to claimer.

7. One above operations done, attacker can withdraw staking tokens by calling `withdraw` function, replaying the flashloan amount.

## Impact

-  Reward claimer is getting very less reward.
-  Hence loss of funds to claimer.

## Tools Used

Manual

## Recommendations

Implement a time delay mechanism between `stake` and `withdraw` function. if tha attacker wants to perform this attack it will fail, bacause of time delay between stake and withdraw function.

## <a id='H-02'></a>H-02. A user can re-update his userBoost storage struct, with different values by calling `delegateBoost()` function.            



## Summary

**Contract** - BoostController.sol

1. Whenever `updateUserBoost(address user, address pool)` function is called, the userBoost storage of user updates let's say userA.

```js
    function updateUserBoost(address user, address pool) external override nonReentrant whenNotPaused {
        if (paused()) revert EmergencyPaused();
        if (user == address(0)) revert InvalidPool();
        if (!supportedPools[pool]) revert PoolNotSupported();
        
        UserBoost storage userBoost = userBoosts[user][pool];
        PoolBoost storage poolBoost = poolBoosts[pool];
        
        uint256 oldBoost = userBoost.amount;
        // Calculate new boost based on current veToken balance

        uint256 newBoost = _calculateBoost(user, pool, 10000); // Base amount
        
        userBoost.amount = newBoost;
        userBoost.lastUpdateTime = block.timestamp;
        
        // Update pool totals safely
        if (newBoost >= oldBoost) {
            poolBoost.totalBoost = poolBoost.totalBoost + (newBoost - oldBoost);
        } else {
            poolBoost.totalBoost = poolBoost.totalBoost - (oldBoost - newBoost);
        }

        poolBoost.workingSupply = newBoost; // Set working supply directly to new boost
        poolBoost.lastUpdateTime = block.timestamp;
        
        emit BoostUpdated(user, pool, newBoost);
        emit PoolBoostUpdated(pool, poolBoost.totalBoost, poolBoost.workingSupply);
    }
```

1. Now, same userA can call `delegateBoost(address to, uint256 amount, uint256 duration)`; passing `to` address as address of
   `pool`.
2. By doing this user will able to update his userBoost storage value to whatever he wants.
3. The main thing, user can re-update is userBoost amount indicated as `@-1-`.
4. `userBoost.amount` is used at many instances; like `getWorkingBalance()` and `getBoostMultiplier()`.
5. By inflating the `userBoost.amount` user can earn more than expected, causing loss to protocol.

`BoostController.sol::delegateBoost()` ->

```js
    function delegateBoost(
        address to,
        uint256 amount,
        uint256 duration
    ) external override nonReentrant {
        if (paused()) revert EmergencyPaused();
        if (to == address(0)) revert InvalidPool();
        if (amount == 0) revert InvalidBoostAmount();
        if (duration < MIN_DELEGATION_DURATION || duration > MAX_DELEGATION_DURATION) 
            revert InvalidDelegationDuration();
        
        uint256 userBalance = IERC20(address(veToken)).balanceOf(msg.sender);
        if (userBalance < amount) revert InsufficientVeBalance();

        UserBoost storage delegation = userBoosts[msg.sender][to];

        if (delegation.amount > 0) revert BoostAlreadyDelegated();
        
@-1-    delegation.amount = amount;
        delegation.expiry = block.timestamp + duration;
        delegation.delegatedTo = to;
        delegation.lastUpdateTime = block.timestamp;
        
        emit BoostDelegated(msg.sender, to, amount, duration);
    }
```

## Vulnerability Details

## Impact

User can claim higher reward (more than necessity), leading to loss of protocol.

## Tools Used

Manual

## Recommendations

implement a check, In `delegateBoost()` function that to address cannot be of registered pool.

## <a id='H-03'></a>H-03. No incentive or enforcement for users to call `FeeCollector.sol::collectFee()` function.            



## Summary
[Link](https://github.com/Cyfrin/2025-02-raac/blob/89ccb062e2b175374d40d824263a4c0b601bcb7f/contracts/core/collectors/FeeCollector.sol#L162)

- The purpose of `collectFee()` function is to collect fee from users and store in `FeeCollector.sol`. 
- The `FeeCollector.sol::collectFee()` is external function and isn't interacting with any of protocol's other contract.
- so this function is intended to be called by users seperatily (not assciated with any deposit or withdraw) operation.
- Users will not pay the fee, if they are not forced to do that.

## Vulnerability Details

Flaw in current architechture, 

## Impact

`FeeCollector.sol` will lack suffiecint RAAC tokens, required for other operations.

## Tools Used

Manual

## Recommendations
Reconsider the architecture.

## <a id='H-04'></a>H-04. Performance share is not distributed to veRAAC holders in `GaugeController.sol::distributeRevenue()`.            



## Summary

Contract - **GaugeController.sol**

The performanceShare to be distributed to all veRAAC holder. but it's not happening in current implementation of `distributeRevenue()` function.

```js
    /**
     * @notice Distributes revenue between veToken holders and gauges
     * @dev Only callable by emergency admin
     * @param gaugeType Type of gauge for distribution
     * @param amount Amount to distribute
     */
    function distributeRevenue(
        GaugeType gaugeType,
        uint256 amount
    ) external onlyRole(EMERGENCY_ADMIN) whenNotPaused {
        if (amount == 0) revert InvalidAmount();

        uint256 veRAACShare = amount * 80 / 100; // 80% to veRAAC holders
        uint256 performanceShare = amount * 20 / 100; // 20% performance fee
        
        revenueShares[gaugeType] += veRAACShare;
        _distributeToGauges(gaugeType, veRAACShare);
        
        emit RevenueDistributed(gaugeType, amount, veRAACShare, performanceShare);
    }
```

## Vulnerability Details

veRAAC holder not receiving performance shares.

## Impact

* Bad reputation to protocol as it's not doing what it supposed to do.
* Loss of funds to users (as they are not receiving performance share).

## Tools Used

Manual

## Recommendations

Distribute performance share to all veRAAC holders.

## <a id='H-05'></a>H-05. Incorrect logic for gauge weight update.            



## Summary

Contract - **GaugeController.sol**

In current implementation user can vote on particular gauge, and can increase it's weight, via calling `vote()` function.

```js
    function vote(address gauge, uint256 weight) external override whenNotPaused {
        if (!isGauge(gauge)) revert GaugeNotFound();
        if (weight > WEIGHT_PRECISION) revert InvalidWeight();
        
        uint256 votingPower = veRAACToken.balanceOf(msg.sender);
        if (votingPower == 0) revert NoVotingPower();

        uint256 oldWeight = userGaugeVotes[msg.sender][gauge];
        userGaugeVotes[msg.sender][gauge] = weight;
        
        _updateGaugeWeight(gauge, oldWeight, weight, votingPower);
        
        emit WeightUpdated(gauge, oldWeight, weight);
    }
```
**_updateGaugeWeight()**

```js
        function _updateGaugeWeight(
            address gauge,
            uint256 oldWeight,
            uint256 newWeight,
            uint256 votingPower
        ) internal {
            Gauge storage g = gauges[gauge];
            
            uint256 oldGaugeWeight = g.weight;
@->         uint256 newGaugeWeight = oldGaugeWeight - (oldWeight * votingPower / WEIGHT_PRECISION)
                + (newWeight * votingPower / WEIGHT_PRECISION);
                
            g.weight = newGaugeWeight;
            g.lastUpdateTime = block.timestamp;
        }
```

Incorrect logic part -
```js
(oldWeight * votingPower / WEIGHT_PRECISION)
```
currently, multiplication of oldWeight of user and current veRACC holding of user is performed, but it's may be possible that veRAAC holding user earlier (during oldWeight time), is less compared to current holding; or veRAAC holding of user earlier (during oldWeight time) is higher than current holder.

So the point is that, the multiplication should be done between **user's old weight** and **user's old voting power/ veRAAC holding**; instead of **user's old weight** and **user's current voting power**.

## Vulnerability Details
Same as above
## Impact

Incorrect updating of gauge weight or `g.weight`, with incorrect value.

## Tools Used
Manual
## Recommendations
Create new state variable struct for user, to track the voting status. something like -

```js
struct UserVoteStatus{
    uint256 last_User_Weight;
    uint256 last_RAAC_holding;
    uint256 last_timestamp;
}
// userAddress -> block.timestamp -> user vote status.

mapping(address => mapping(uint256 => UserVoteStatus))
```
Implement something like this in `vote()` function instead of -

```js
(oldWeight * votingPower / WEIGHT_PRECISION)
```
## <a id='H-06'></a>H-06. Any user can claim reward from gauge, even though user not staked anything.            



## Summary

An user can get reward, by calling `getReward()` function.

Contract - BaseGauge.sol

```js
    function getReward() external virtual nonReentrant whenNotPaused updateReward(msg.sender) {
        if (block.timestamp - lastClaimTime[msg.sender] < MIN_CLAIM_INTERVAL) {
            revert ClaimTooFrequent();
        }
        
        lastClaimTime[msg.sender] = block.timestamp;
        UserState storage state = userStates[msg.sender];
@->     uint256 reward = state.rewards; 
        
        if (reward > 0) {
            state.rewards = 0;
            
            uint256 balance = rewardToken.balanceOf(address(this));
            if (reward > balance) {
                revert InsufficientBalance();
            }
            
            rewardToken.safeTransfer(msg.sender, reward);
            emit RewardPaid(msg.sender, reward);
        }
    }
```

**function flow**

1 .`getReward()` -> 2 .`updateReward(msg.sender)` -> 3 .`_updateReward(account)`

```js
    function _updateReward(address account) internal {
        rewardPerTokenStored = getRewardPerToken();
        lastUpdateTime = lastTimeRewardApplicable();

        if (account != address(0)) {
            UserState storage state = userStates[account];
            state.rewards = earned(account);

            state.rewardPerTokenPaid = rewardPerTokenStored;
            state.lastUpdateTime = block.timestamp;
            emit RewardUpdated(account, state.rewards);
        }
    }
```

4 .`earned()` -> **NON-ZERO**

```js
    function earned(address account) public view returns (uint256) {
        return (getUserWeight(account) * 
        // here - start again
            (getRewardPerToken() - userStates[account].rewardPerTokenPaid) / 1e18
        ) + userStates[account].rewards;
    }
```

5 . `getRewardPerToken()` ->
**NON-ZERO** -

* as the return value (rewardPerTokenStored) isn't user dependent.
* `(lastTimeRewardApplicable() - lastUpdateTime) * rewardRate * 1e18 / totalSupply()` always non-zero.

```js
    function getRewardPerToken() public view returns (uint256) {
        if (totalSupply() == 0) {
            return rewardPerTokenStored;
        }
        return rewardPerTokenStored + (
            (lastTimeRewardApplicable() - lastUpdateTime) * rewardRate * 1e18 / totalSupply()
        );
    }

```

6 .`getUserWeight()` ->
**NON-ZERO** -

* As `_applyBoost()` is non-zero. see point 8.

```js
    function getUserWeight(address account) public view virtual returns (uint256) {
        uint256 baseWeight = _getBaseWeight(account);
        return _applyBoost(account, baseWeight);
    }
```

7 . `_getBaseWeight(account)` ->
**NON-ZERO** -

* This will be non-zero value as it's returning gauge weight of Gauge contract, not user.

```js
    function _getBaseWeight(address account) internal view virtual returns (uint256) {
        return IGaugeController(controller).getGaugeWeight(address(this)); 
    }
```

8 . `_applyBoost(account, baseWeight);` -> **NON-ZERO** The only way to return 0 is if veRAAC balance of account is 0.

NOTE: It's possible that veRAAC balance of user is non-zero, but he haven't staked any veRAAC in guage contract.

```js
    function _applyBoost(address account, uint256 baseWeight) internal view virtual returns (uint256) {
        if (baseWeight == 0) return 0;
        
        IERC20 veToken = IERC20(IGaugeController(controller).veRAACToken());
        uint256 veBalance = veToken.balanceOf(account);
        uint256 totalVeSupply = veToken.totalSupply();

        // Create BoostParameters struct from boostState
        BoostCalculator.BoostParameters memory params = BoostCalculator.BoostParameters({
            maxBoost: boostState.maxBoost,
            minBoost: boostState.minBoost,
            boostWindow: boostState.boostWindow,
            totalWeight: boostState.totalWeight,
            totalVotingPower: boostState.totalVotingPower,
            votingPower: boostState.votingPower
        });

        uint256 boost = BoostCalculator.calculateBoost(
            veBalance,
            totalVeSupply,
            params
        );
        
        // $mynotes - weight is in 1e18 format so it's correct.
        return (baseWeight * boost) / 1e18;
    }
```

since, `earned()` is non - zero, so `state.rewards` is also non-zero, which means non-zero reward is being tranfered to user.

## Vulnerability Details

It means any user who have veRAAC balance, but not staked in gauge can still earn rewards, without staking.

## Impact

* User who haven't staked veRAAC are also claiming the reward, which is unethical for user who deposited in Gauge contract.
* This could possibly drain out whole RAAC balance of Gauge contract.

## Tools Used

Manual

## Recommendations

Implement a check, to ensure only stakers can claim reward.

## <a id='H-07'></a>H-07. Scaling done twice, in transfer and transferFrom function in `RToken.sol`.            



## Summary

Contract - **RToken.sol**

**function-flow** -

`transferFrom()` -> `super.transferFrom()` -> `_update` -> `super._update`

```js
    function transferFrom(address sender, address recipient, uint256 amount) public override(ERC20, IERC20) returns (bool) {
        uint256 scaledAmount = amount.rayDiv(_liquidityIndex);
        return super.transferFrom(sender, recipient, scaledAmount);
    }
```

```js
    function _update(address from, address to, uint256 amount) internal override {
        // Scale amount by normalized income for all operations (mint, burn, transfer)
        // @audit - scaledAmount has been scaled 2 times, once in the transferFrom function and once here.
        // so it will cause loss and from sender and protocol both, beacuse of extra transfer
        uint256 scaledAmount = amount.rayDiv(ILendingPool(_reservePool).getNormalizedIncome());
        super._update(from, to, scaledAmount);
    }
```

Similar kind of issue in `calculateDustAmount()` function as well,

1. scaling is done in `totalSupply()` function.
2. then again scaling is done via -

```js
uint256 totalRealBalance = currentTotalSupply.rayMul(ILendingPool(_reservePool).getNormalizedIncome());
```

## Vulnerability Details

* The scaling of amount is already doen in parent caller function.
* The scaling is done again in child function.
* so, scaling is done 2 times.

## Impact

Twice scaling will lead to prision issue. and incorrect accounting.

## Tools Used

Manual

## Recommendations

Remove one extra scaling.

## <a id='H-08'></a>H-08. An user can loose funds, if he calls `veRACC.sol::lock()` more than once during lock duration.            



## Summary

Contract - **`veRACC.sol`**

`veRACC.sol::lock()` is as follow -

```js
    function lock(uint256 amount, uint256 duration) external nonReentrant whenNotPaused {
        if (amount == 0) revert InvalidAmount();
        if (amount > MAX_LOCK_AMOUNT) revert AmountExceedsLimit();
        if (totalSupply() + amount > MAX_TOTAL_SUPPLY) revert TotalSupplyLimitExceeded();
        if (duration < MIN_LOCK_DURATION || duration > MAX_LOCK_DURATION) 
            revert InvalidLockDuration();

        // Do the transfer first - this will revert with ERC20InsufficientBalance if user doesn't have enough tokens
        raacToken.safeTransferFrom(msg.sender, address(this), amount);
        
        // Calculate unlock time
        uint256 unlockTime = block.timestamp + duration;
        
        // Create lock position
@->     _lockState.createLock(msg.sender, amount, duration);
        _updateBoostState(msg.sender, amount);

        // Calculate initial voting power
        (int128 bias, int128 slope) = _votingState.calculateAndUpdatePower(
            msg.sender,
            amount,
            unlockTime
        );

        // Update checkpoints
        uint256 newPower = uint256(uint128(bias));
        _checkpointState.writeCheckpoint(msg.sender, newPower);

        // Mint veTokens
        _mint(msg.sender, newPower);

        emit LockCreated(msg.sender, amount, unlockTime);
    }
```

`LockManager.sol::createLock()` ->

```js
    function createLock(
        LockState storage state,
        address user,
        uint256 amount,
        uint256 duration
    ) internal returns (uint256 end) {
        // Validation logic remains the same
        if (state.minLockDuration != 0 && state.maxLockDuration != 0) {
            if (duration < state.minLockDuration || duration > state.maxLockDuration) 
                revert InvalidLockDuration();
        }

        if (amount == 0) revert InvalidLockAmount();

        end = block.timestamp + duration;
        
        state.locks[user] = Lock({
            amount: amount,
            end: end,
            exists: true
        });

        state.totalLocked += amount;

        emit LockCreated(user, amount, end);
        return end;
    }
```

1. When `lock()` function is called by user, a lock is created  like -

```js
    state.locks[user] = Lock({
        amount: amount, // amount = A1
        end: end,
        exists: true
    });
```

1. Now, suppose same innocent user calls `lock()` function again (within current lock duration phase), then again `LockManager.sol::createLock()` will be executed, and `state.locks[user]` will be updated with new value.

2. For first call, the amount was let's say **`A1`** and for second call **`A2`**.

3. which means user's lock state amount has been updated from **`A1`** to **`A2`**.

4. Now when lock period is over; user will call `veRAAC.sol::withdraw()` ->

`veRAAC.sol::withdraw()` ->

```js
    function withdraw() external nonReentrant {
        LockManager.Lock memory userLock = _lockState.locks[msg.sender];
        
        if (userLock.amount == 0) revert LockNotFound();
        if (block.timestamp < userLock.end) revert LockNotExpired();

@-1 >   uint256 amount = userLock.amount;
        uint256 currentPower = balanceOf(msg.sender);

        // Clear lock data
        delete _lockState.locks[msg.sender];
        delete _votingState.points[msg.sender];

        // Update checkpoints
        _checkpointState.writeCheckpoint(msg.sender, 0);

        // Burn veTokens and transfer RAAC
        _burn(msg.sender, currentPower);
@-2 >    raacToken.safeTransfer(msg.sender, amount);
        
        emit Withdrawn(msg.sender, amount);
    }
```

1. it will fetch user's locked amount marked as `@-1 >` and then transfer raccs to user marked as `@-2`.

2. Means lock amount **`A2`** will be transferred to user.

3. But remember, user has transfer **`A1 + A2`** amount of tokens into the system but only receiving **`A2`** amounts.

4. which means user has lost **`A1`** amount of raacs.

## Vulnerability Details

Same as above.

## Impact

* User funds are lost, the raac tokens are lost.

## Tools Used

Manual

## Recommendations

* Put a check in lock function, that user can't lock `raacs` again for existing lock **calling** **`lock()`**.
* `increaseLock()` for this purpose, use this.
* But even if they mistankly called **lock()** again for existing lock, make below improvement in `LockManager.sol::createLock()` ->

```diff
    function createLock(
        LockState storage state,
        address user,
        uint256 amount,
        uint256 duration
    ) internal returns (uint256 end) {
        
        // Code
        .
        .
        .

        state.locks[user] = Lock({
-           amount: amount,
+          amount: state.locks[user].amount + amount,
            end: end,
            exists: true
        });
        .
        .
        .
        // Code
    }

```

## <a id='H-09'></a>H-09. User will always get 0 reward on calling `FeeCollector.sol::claimRewards`.            



## Summary

The `claimRewards()` function is as follow -

```js
    function claimRewards(address user) external override nonReentrant whenNotPaused returns (uint256) {
        if (user == address(0)) revert InvalidAddress();
        
        uint256 pendingReward = _calculatePendingRewards(user);
        if (pendingReward == 0) revert InsufficientBalance();
        
        // Reset user rewards before transfer
@->     userRewards[user] = totalDistributed;
        
        // Transfer rewards
        raacToken.safeTransfer(user, pendingReward);
        
        emit RewardClaimed(user, pendingReward);
        return pendingReward;
    }
```

**\_calculatePendingRewards(user)** ->

```js
    function _calculatePendingRewards(address user) internal view returns (uint256) {
        uint256 userVotingPower = veRAACToken.getVotingPower(user);
        if (userVotingPower == 0) return 0;

        uint256 totalVotingPower = veRAACToken.getTotalVotingPower();
        if (totalVotingPower == 0) return 0;
   
        uint256 share = (totalDistributed * userVotingPower) / totalVotingPower;
        return share > userRewards[user] ? share - userRewards[user] : 0;
    }
```

1. `userRewards[user]` is wrongly updated to `totalDistributed` instead it should not updated to `pendingReward`
   this will cause user reward to always get stuck as -
   totalDistributed will always be greater than (totalDistributed \* userVotingPower) / totalVotingPower;.
2. so, `userRewards[user] > share` will be the case everytime; in other words else condition i.e. share = 0 will always
   executed.

## Vulnerability Details

1. Whenever user will call `claimRewards()` he will recive 0 reward.
2. The reason is incorrect updation of `userRewards[user]` to `totalDistributed` instead of `pendingReward`
3. `totalDistributed` has totally different purpose, and it's used to track historical distribution of reward to all users.

## Impact

User will always recive 0 rewards.

## Tools Used

Eye

## Recommendations

```diff
-   userRewards[user] = totalDistributed;
+   userRewards[user] = pendingReward;
```

## <a id='H-10'></a>H-10. User fund can be stuck if totalDistributed increases at very slow amount and user VotingPower increases very fast.            



`FeeCollector.sol::claimRewards()` ->

```js
    function claimRewards(address user) external override nonReentrant whenNotPaused returns (uint256) {
        if (user == address(0)) revert InvalidAddress();
        
        uint256 pendingReward = _calculatePendingRewards(user);
        if (pendingReward == 0) revert InsufficientBalance();
        
        userRewards[user] = totalDistributed;
        
        // Transfer rewards
        raacToken.safeTransfer(user, pendingReward);
        
        emit RewardClaimed(user, pendingReward);
        return pendingReward;
    }
```

code snippet for **FeeCollector.sol::\_calculatePendingRewards** -

```js
    function _calculatePendingRewards(address user) internal view returns (uint256) {
        uint256 userVotingPower = veRAACToken.getVotingPower(user);
        if (userVotingPower == 0) return 0;

        uint256 totalVotingPower = veRAACToken.getTotalVotingPower();
        if (totalVotingPower == 0) return 0;

        uint256 share = (totalDistributed * userVotingPower) / totalVotingPower;
        return share > userRewards[user] ? share - userRewards[user] : 0;
    }
```

## Vulnerability Details

1. for first call to `claimRewards()`; let's say totalDistributed = 100, userVotingPower = 100, totalVotingPower = 100.
   share\_1 = 100\*100/100 = 100, userRewards\[user] = 100.

2. for second call to `claimRewards()`; let's say totalDistributed = 200, userVotingPower = 150, totalVotingPower = 400.
   share\_2 = 200\*150/400 = 75

since; **share\_2 (latest share) < share\_1 (old share)**, the else condition will be executed which means user will get 0
reward. even though user is eligible to get his reward.

## Impact

whenever someone will call `claimRewards()` -> `_calculatePendingRewards()`, the user will not be able to get the reward due to revertion (if above scenario arises).

## Tools Used

Manual

## Recommendations

Reconsider architecture

    
# Medium Risk Findings

## <a id='M-01'></a>M-01. Average weight will always be 0, whenever period update occurs via `BaseGuage::updatePeriod()` function.            



## Summary

The `BaseGuage::updatePeriod()` function is as follow -

```js
    /**
     * @notice Updates the period and calculates new weights
     */
    function updatePeriod() external override onlyController {
        uint256 currentTime = block.timestamp;
        uint256 periodEnd = periodState.periodStartTime + getPeriodDuration();
        
        if (currentTime < periodEnd) {
            revert PeriodNotElapsed();
        }

        uint256 periodDuration = getPeriodDuration();
        // Calculate average weight for the ending period
@->     uint256 avgWeight = periodState.votingPeriod.calculateAverage(periodEnd);
        
        // Calculate the start of the next period (ensure it's in the future)
        uint256 nextPeriodStart = ((currentTime / periodDuration) + 2) * periodDuration;
        
        // Reset period state
        periodState.distributed = 0;
        periodState.periodStartTime = nextPeriodStart;

        // Create new voting period
        TimeWeightedAverage.createPeriod(
            periodState.votingPeriod,
            nextPeriodStart,
            periodDuration,
            avgWeight,
            WEIGHT_PRECISION
        );
    }
```

`calculateAverage()` function is as follow -

```js
    function calculateAverage(
        Period storage self,
        uint256 timestamp
    ) internal view returns (uint256) {
        if (timestamp <= self.startTime) return self.value;
        
        uint256 endTime = timestamp > self.endTime ? self.endTime : timestamp;
        // @audit - this will always be 0.
        uint256 totalWeightedSum = self.weightedSum;
        
        if (endTime > self.lastUpdateTime) {
            uint256 duration = endTime - self.lastUpdateTime;
            // self.value = 0, determined in constructor.
            uint256 timeWeightedValue = self.value * duration;
            if (duration > 0 && timeWeightedValue / duration != self.value) revert ValueOverflow();
            totalWeightedSum += timeWeightedValue;
        }
        
        return totalWeightedSum / (endTime - self.startTime); // returns 0
    }
```

1. self.value = 0, determined in constructor.
2. so, `timeWeightedValue` will also be 0. means return value will also be 0.
3. so `avgWeight` will be zero, as-

```js
uint256 avgWeight = periodState.votingPeriod.calculateAverage(periodEnd);
```

1. It means when new period will be created via `TimeWeightedAverage.createPeriod`, -

```js
        TimeWeightedAverage.createPeriod(
            periodState.votingPeriod,
            nextPeriodStart,
            periodDuration,
            avgWeight, // 0
            WEIGHT_PRECISION
        );
```

1. If means whenever new period P1, P2, P3... is created, the avgWeight accounting of previous periods is not tracked.

2. Which is very important aspect of protocol, to track the time-weighted average.

3. But with current implementation it's not happening, i.e. average weighted value is always 0.

4. **NOTE** - Developers may be assuming that they can set the weight of current period by calling `setInitialWeight()` function, but they won't be able to -

```js
    function setInitialWeight(uint256 weight) external onlyController {
        uint256 periodDuration = getPeriodDuration();
        uint256 currentTime = block.timestamp;
        uint256 nextPeriodStart = ((currentTime / periodDuration) + 2) * periodDuration;
        
        TimeWeightedAverage.createPeriod(
            periodState.votingPeriod, 
            nextPeriodStart,
            periodDuration,
            weight,
            10000 // WEIGHT_PRECISION
        );

        periodState.periodStartTime = nextPeriodStart;
    }
```

1. The reason why they won't be able to beacuse, there's check inside `TimeWeightedAverage.createPeriod` function -

```js
        if (self.startTime != 0 && startTime < self.startTime + self.totalDuration) {
            revert PeriodNotElapsed();
        }
```

1. In simple words, controller can't update the weight of current period, more accuratly contraoller can't change any data of current ongoing period.

2. .value can only be updated after end of a period/ or during start of next period.

## Vulnerability Details

## Impact

* averageWeight of period is not tracked or it always returns 0, which is not the expected behavior.
*

## Tools Used

Manual

## Recommendations

* In constructor, don't initialize initial value with 0, do it with any not-zero number.
* The main cause of this issue is this, which is triggering as chain-reaction of avgWeight for all periods to 0.

## <a id='M-02'></a>M-02. re-updating `lastUpdateTime` in `BaseGauge.sol::notifyRewardAmount()` could be problamatic.            



## Summary

`notifyRewardAmount` is as follow -

```js
    function notifyRewardAmount(uint256 amount) external override onlyController updateReward(address(0)) {
        if (amount > periodState.emission) revert RewardCapExceeded();

        rewardRate = notifyReward(periodState, amount, periodState.emission, getPeriodDuration());
        periodState.distributed += amount;
        
        // $mynotes - balance of specific guage contract.
        uint256 balance = rewardToken.balanceOf(address(this));
        if (rewardRate * getPeriodDuration() > balance) {
            revert InsufficientRewardBalance();
        }

@ -1 >  lastUpdateTime = block.timestamp;
        emit RewardNotified(amount);
    }
```

firstly it will call modifier `updateReward()` ->

```js
    modifier updateReward(address account) {
        _updateReward(account);
        _;
    }
```

```js
    function _updateReward(address account) internal {
        rewardPerTokenStored = getRewardPerToken();
@ ->    lastUpdateTime = lastTimeRewardApplicable();

        if (account != address(0)) {
            UserState storage state = userStates[account];
            state.rewards = earned(account);

            state.rewardPerTokenPaid = rewardPerTokenStored;
            state.lastUpdateTime = block.timestamp;
            emit RewardUpdated(account, state.rewards);
        }
    }
```

`lastTimeRewardApplicable()` ->

```js
    function lastTimeRewardApplicable() public view returns (uint256) {
        return block.timestamp < periodFinish() ? block.timestamp : periodFinish();
    }
```

`periodFinish()` ->

```js
    function periodFinish() public view returns (uint256) {
        return lastUpdateTime + getPeriodDuration();
    }
```

`getPeriodDuration()` ->

```js
    function getPeriodDuration() public view virtual returns (uint256) {
        return 7 days; // Default period duration, can be overridden by child contracts
    }
```

1. So, if `block.timestamp` is greater than `periodFinish()`, then lastUpdateTime = `periodFinish()`.
2. But code line marked as `@ -1 >` is reupdating lastUpdateTime to `block.timestamp`.

```js
lastUpdateTime = block.timestamp;
```

1. Which is incorrect, it should only be value that's being updated in modifier.

## Vulnerability Details

Same as above.

## Impact

`lastUpdateTime` is associated with reward claim on gauge contracts, it's incorrect value could either lead to delay of reward or early reward claim, not good for user and protcol repectively.

## Tools Used

Manual

## Recommendations

remove `lastUpdateTime = block.timestamp;` from `notifyRewardAmount()` functions.

## <a id='M-03'></a>M-03. Anyone can call `GauageController.sol::distributeRewards()` function.            



## Summary

In `GauageController.sol` anyone can call `distributeRewards()` function, increamenting the reward amount of a gauge, the actual behaviour should be, it's only be called by admin.

```js
    function distributeRewards(
        address gauge
    ) external override nonReentrant whenNotPaused {
        if (!isGauge(gauge)) revert GaugeNotFound();
        if (!gauges[gauge].isActive) revert GaugeNotActive();
        
        uint256 reward = _calculateReward(gauge);
        if (reward == 0) return;
        
        IGauge(gauge).notifyRewardAmount(reward);
        emit RewardDistributed(gauge, msg.sender, reward);
    }
```
## Vulnerability Details
Missing access modifier in `disitributeRewards()` function.
## Impact

- Guage reward allocation can be arbitraly increase by anyone.
- `notifyRewardAmount()` function in BaseGauge.sol, doesn't implement any check how frequent it should be called.
- Which means whenever this function is called new reward will be allocated to gauge, without admin consent.

## Tools Used

Manual

## Recommendations

Implement access modifier in `distributeRewards()` function.

## <a id='M-04'></a>M-04. On boost delegation removal, updating to poolBoost storage struct isn't happening.            



## Summary

**Contract** - BoostController.sol

Code snippet of `delegateBoost()` function.

```js
    function delegateBoost(
        address to,
        uint256 amount,
        uint256 duration
    ) external override nonReentrant {
        if (paused()) revert EmergencyPaused();
        if (to == address(0)) revert InvalidPool();
        if (amount == 0) revert InvalidBoostAmount();
        if (duration < MIN_DELEGATION_DURATION || duration > MAX_DELEGATION_DURATION) 
            revert InvalidDelegationDuration();
        
        uint256 userBalance = IERC20(address(veToken)).balanceOf(msg.sender);
        if (userBalance < amount) revert InsufficientVeBalance();

        UserBoost storage delegation = userBoosts[msg.sender][to];

        if (delegation.amount > 0) revert BoostAlreadyDelegated();
        
        delegation.amount = amount;
        delegation.expiry = block.timestamp + duration;
        delegation.delegatedTo = to;
        delegation.lastUpdateTime = block.timestamp;

        // @audit - PoolBoost storage poolBoost = poolBoosts[to]; not being used here, which is being used in remove delegation operat
        // in removeBoostDelegation function, there will not any state change happen as poolBoost data will be empty object there.    
        emit BoostDelegated(msg.sender, to, amount, duration);
    }
```
Code snippet of `removeBoostDelegation()` function.

```js
    function removeBoostDelegation(address from) external override nonReentrant {
        UserBoost storage delegation = userBoosts[from][msg.sender];
        if (delegation.delegatedTo != msg.sender) revert DelegationNotFound();
        if (delegation.expiry > block.timestamp) revert InvalidDelegationDuration();
        
        // Update pool boost totals before removing delegation
        PoolBoost storage poolBoost = poolBoosts[msg.sender];
        // @audit - reflection of previous audit, no if statement will execute.
        if (poolBoost.totalBoost >= delegation.amount) {
            poolBoost.totalBoost -= delegation.amount;
        }
        if (poolBoost.workingSupply >= delegation.amount) {
            poolBoost.workingSupply -= delegation.amount;
        }
        poolBoost.lastUpdateTime = block.timestamp;
        
        emit DelegationRemoved(from, msg.sender, delegation.amount);
        delete userBoosts[from][msg.sender];
    }
```

## Vulnerability Details

1. When `removeBoostDelegation()` is called, it's expected to update `poolBoost.totalBoost` and `poolBoost.workingSupply`.
2. If `poolBoost.totalBoost >= delegation.amount` and `poolBoost.workingSupply >= delegation.amount`, then poolBoost state
will update.
3. But problem here is that `poolBoost` will be empty object all the time.
4. Why? Because in `delegateBoost()` function, poolBoost is not set for `to` address. something like -
```js
PoolBoost storage poolBoost = poolBoosts[to];
```
5. as `to` in `delegateBoost()` is equivalent to `msg.sender` in `removeBoostDelegation()`.
6. In `PoolBoost storage poolBoost = poolBoosts[msg.sender];`; poolBoost will always be an empty object.
7. so the operation (below) will never execute.
```js
    if (poolBoost.totalBoost >= delegation.amount) {
        poolBoost.totalBoost -= delegation.amount;
    }
    if (poolBoost.workingSupply >= delegation.amount) {
        poolBoost.workingSupply -= delegation.amount;
    }
```

## Impact

1. `poolBoost` state not being udpated can lead to multiple issue, as it's used many places, throughout the proctocol.
2. Also it's causing the function to not work properly.

## Tools Used

Manual

## Recommendations

In `delegateBoost()` function update poolBoost for `to` address. something like -

```diff
    function delegateBoost(
        address to,
        uint256 amount,
        uint256 duration
    ) external override nonReentrant {
        // Code....
+       PoolBoost storage poolBoost = poolBoosts[to];
+       poolBoost.totalBoost += delegation.amount;
+       poolBoost.workingSupply += delegation.amount;
        // Code....

    }
```
## <a id='M-05'></a>M-05. Pool boost working supply not updated correctly, in `BoostController.sol::updateUserBoost`.            



**Contract** - BoostController.sol

**Code snippet**

```js
    function updateUserBoost(address user, address pool) external override nonReentrant whenNotPaused {
        if (paused()) revert EmergencyPaused();
        if (user == address(0)) revert InvalidPool();
        if (!supportedPools[pool]) revert PoolNotSupported();
        
        UserBoost storage userBoost = userBoosts[user][pool];
        PoolBoost storage poolBoost = poolBoosts[pool];
        
        uint256 oldBoost = userBoost.amount;
        // Calculate new boost based on current veToken balance
        uint256 newBoost = _calculateBoost(user, pool, 10000); // Base amount
        
        userBoost.amount = newBoost;
        userBoost.lastUpdateTime = block.timestamp;
        
        // Update pool totals safely
        if (newBoost >= oldBoost) {
            poolBoost.totalBoost = poolBoost.totalBoost + (newBoost - oldBoost);
        } else {
            poolBoost.totalBoost = poolBoost.totalBoost - (oldBoost - newBoost);
        }

@ ->    poolBoost.workingSupply = newBoost; // Set working supply directly to new boost
        poolBoost.lastUpdateTime = block.timestamp;
        
        emit BoostUpdated(user, pool, newBoost);
        emit PoolBoostUpdated(pool, poolBoost.totalBoost, poolBoost.workingSupply);
    }
```

The purpose of `poolBoost.workingSupply` is to track total boosts of all users in action, it's not designed for to track for
single user, in the current implementation the poolBoost working supply is being updated to `newBoost` (boost of single user).

## Vulnerability Details

1. `poolBoost.workingSupply` is supposed to take care of all user's boost.
2. In current implementation `poolBoost.workingSupply` is being updated with `newBoost` of a single user.

## Impact

* `poolBoost.workingSupply` isn't behaving as it should behave.
* The expected behaviour should be summation and substraction of `newBoost`, on `poolBoost.workingSupply`; like this -

```js
        if (newBoost >= oldBoost) {
            poolBoost.workingSupply = poolBoost.workingSupply + (newBoost - oldBoost);
        } else {
            poolBoost.workingSupply = poolBoost.workingSupply - (oldBoost - newBoost);
        }
```

## Tools Used

Manual

## Recommendations

Implement this way instead -

```diff
-       poolBoost.workingSupply = newBoost;

+       if (newBoost >= oldBoost) {
+           poolBoost.workingSupply = poolBoost.workingSupply + (newBoost - oldBoost);
+       } else {
+           poolBoost.workingSupply = poolBoost.workingSupply - (oldBoost - newBoost);
+       }

```

## <a id='M-06'></a>M-06. An user can unethically liquidated even his healthFactor is in safe zone.            



## Summary

In `LendingPool.sol` - the liquidation initiation takes place as follow -

```js
    function initiateLiquidation(address userAddress) external nonReentrant whenNotPaused {
        if (isUnderLiquidation[userAddress]) revert UserAlreadyUnderLiquidation();

        // update state
        ReserveLibrary.updateReserveState(reserve, rateData);

        UserData storage user = userData[userAddress];
        
        // position.
        uint256 healthFactor = calculateHealthFactor(userAddress);
        
        if (healthFactor >= healthFactorLiquidationThreshold) revert HealthFactorTooLow();

        isUnderLiquidation[userAddress] = true;
        liquidationStartTime[userAddress] = block.timestamp;

        emit LiquidationInitiated(msg.sender, userAddress);
    }
```

finalization via -

```js
    function finalizeLiquidation(address userAddress) external nonReentrant onlyStabilityPool {
        if (!isUnderLiquidation[userAddress]) revert NotUnderLiquidation();

        // update state
        ReserveLibrary.updateReserveState(reserve, rateData);

        if (block.timestamp <= liquidationStartTime[userAddress] + liquidationGracePeriod) {
            revert GracePeriodNotExpired();
        }

        UserData storage user = userData[userAddress];

        uint256 userDebt = user.scaledDebtBalance.rayMul(reserve.usageIndex);

        
        isUnderLiquidation[userAddress] = false;
        liquidationStartTime[userAddress] = 0;
         // Transfer NFTs to Stability Pool
        for (uint256 i = 0; i < user.nftTokenIds.length; i++) {
            uint256 tokenId = user.nftTokenIds[i];
            user.depositedNFTs[tokenId] = false;
            raacNFT.transferFrom(address(this), stabilityPool, tokenId);
        }
        delete user.nftTokenIds;

        // Burn DebtTokens from the user
        (uint256 amountScaled, uint256 newTotalSupply, uint256 amountBurned, uint256 balanceIncrease) = IDebtToken(reserve.reserveDebtTokenAddress).burn(userAddress, userDebt, reserve.usageIndex);

        // Transfer reserve assets from Stability Pool to cover the debt
        IERC20(reserve.reserveAssetAddress).safeTransferFrom(msg.sender, reserve.reserveRTokenAddress, amountScaled);

        // Update user's scaled debt balance
        user.scaledDebtBalance -= amountBurned;
        reserve.totalUsage = newTotalSupply;

        // Update liquidity and interest rates
        ReserveLibrary.updateInterestRatesAndLiquidity(reserve, rateData, amountScaled, 0);


        emit LiquidationFinalized(stabilityPool, userAddress, userDebt, getUserCollateralValue(userAddress));
    }
```

## Vulnerability Details

* suppose `healthFactor` is less than `healthFactorLiquidationThreshold`, and someone initaites the liquidation.
* now owner sets a new `healthFactorLiquidationThreshold` (by calling `setParameter()` function), which is less than healthFactor.
* In this case liquidation shouldn't happen, but it will take place, hence unethical liquidation of user's position.
* in other words, owner updates healthFactorLiquidationthreshold during grace period.
* And we can see that inside `finalizeLiquidation()` the health factor of user isn't checked again.

## Impact

Unethical liquidation of user, even healthfactor is greater than healthFactorLiquidationThreshold.

## Tools Used

Manual

## Recommendations

Re-initialize liquidation initiation process, in case admin updates health factor threshold, during grace period of a liquidation

## <a id='M-07'></a>M-07. Token validation isn't done in `Treasory.sol::deposit()`.            



## Summary

Code snippet of `Treasory.sol::deposit()` -

```js
    function deposit(address token, uint256 amount) external override nonReentrant {
        if (token == address(0)) revert InvalidAddress();
        if (amount == 0) revert InvalidAmount();
        
        IERC20(token).transferFrom(msg.sender, address(this), amount);
        _balances[token] += amount;
        _totalValue += amount;
        
        emit Deposited(token, amount);
    }
```

* token address isn't verified,
* Attacker can put any fake token and hit `deposit()` function which will inflate `_totalValue` very high.
* This will lead to disperncy between actual balance of treasory and internal accounting mapping `_balances[token]`.
* This can lead to incorrect amount of token related actions like depositing and withdrawing.

## Vulnerability Details

Same as above

## Impact

Non synchronous value between actual balance of treasury contract and internal accounting, can lead many flaws in token inflow/ outflow operations.

## Tools Used

Manual

## Recommendations

Implement a require statement to verify if token is legit or not.

## <a id='M-08'></a>M-08. Staleness of NFT price is not checked.            



## Summary

Contract - `LendingPool.sol`

```js
    function getNFTPrice(uint256 tokenId) public view returns (uint256) {
        (uint256 price, uint256 lastUpdateTimestamp) = priceOracle.getLatestPrice(tokenId);
        // @audit - it's not checking staleness of price, by leveraging lastUpdateTimestamp.
        if (price == 0) revert InvalidNFTPrice();
        return price;
    }
```

It's fetching `lastUpdateTimestamp` but not cheaking w\.r.t heartbeat of oracle.

## Vulnerability Details

It can lead to stale price return, if the feed not updated frequently.

## Impact

Wrong NFT price fetching from oracle.

## Tools Used

Manual

## Recommendations

verify `lastUpdateTimestamp` w\.r.t heartbeat of oracle feed.



# Low Risk Findings

## <a id='L-01'></a>L-01. Extra minting of raacTokens to stability pool; if `lastUpdateBlock` is reset to less value than earlier, by calling setLastUpdateBlock, in `RAACMinter.sol`.            



## Summary

The `tick()` function in `RAACMinter.sol` does following :-

\| Triggers the minting process and updates the emission rate if the interval has passed

```js
function tick() external nonReentrant whenNotPaused {
        if (emissionUpdateInterval == 0 || block.timestamp >= lastEmissionUpdateTimestamp + emissionUpdateInterval) {
            updateEmissionRate();
        }

        uint256 currentBlock = block.number;
        uint256 blocksSinceLastUpdate = currentBlock - lastUpdateBlock;
        if (blocksSinceLastUpdate > 0) {
            uint256 amountToMint = emissionRate * blocksSinceLastUpdate;

            if (amountToMint > 0) {
                excessTokens += amountToMint;
                lastUpdateBlock = currentBlock;
                raacToken.mint(address(stabilityPool), amountToMint);
                emit RAACMinted(amountToMint);
            } 
        } 
    }
```

The problem is that if `lastUpdateBlock` is reset to less value as it was originally, by calling `setLastUpdateBlock()` by admin. extra minting of tokens to stability pool will occur, due to block/ block number overlap. Let's analyse.

```Solidity
                                                In this duration 
                                                tokens minted twice.
                                                 |            |
        t1                        t2             |            t3                  t4
        |--------------------------|-------------|------------|--------------------|
    block.number[t1]          block.number[t2]   |     block.number[t3]        block.number[t4] 
                                                 |
                                            block.number[<t3]
    
 tick() function can be called at t1, t2, t3, t4.

 where, t1, t2, t3, t4 are block.timestamps representing lastEmissionUpdateTimestamp.

 At time t3.
 uint256 blocksSinceLastUpdate = currentBlock - lastUpdateBlock;
 The blockSinceLastUpdate = block.number[t3] - block.number[t2] = X

 At time t4. [Typical case]
 uint256 blocksSinceLastUpdate = currentBlock - lastUpdateBlock;
 The blockSinceLastUpdate_1 = block.number[t4] - block.number[t3] = Y

 Suppose admin updates the lastUpdateBlock value less then block.number[t3], let's call 
 it block.number[<t3], by executing setLastUpdateBlock(<t3) or emergencyShutdown() function.

 Now at time t4, [Buggy case] 
 the blockSinceLastUpdate_2 = block.number[t4] - block.number[<t3] = Z
 clearly, Z > Y

 Tokens has been minted for X.
 Now tokens will be minted from block.number[<t3] - block.number[t4].

 The problem is that raacTokens are minted again for block duration between block.number[<t3] and block.number[t3].
 So extra tokens is being minted to stability pool.
```

## Vulnerability Details

Similar to summary section.

## Impact

* Extra tokens is being minted to stability pool.
* Inside stability pool contract there is function `calculateRaacRewards()` to calculate raac rewards,  -

```js
    function calculateRaacRewards(address user) public view returns (uint256) {
        uint256 userDeposit = userDeposits[user];
        uint256 totalDeposits = deToken.totalSupply();

        uint256 totalRewards = raacToken.balanceOf(address(this));
        if (totalDeposits < 1e6) return 0;

@->     return (totalRewards * userDeposit) / totalDeposits;
    }
```

* as raac reward is directly proportional to raac balance of stability pool, means user will get extra raac rewards than intended.

## Tools Used

Manual

## Recommendations

Implement a functionality such that if there is block number overlap, or reward minted 2 ice for certain duration, then burn those extra minted raac tokens.

## <a id='L-02'></a>L-02. Griefing attack via `LendingPool::updateState()` function.            



## Summary

Contract - `LendingPool.sol`.

The `updateState()` function is -

```js
    function updateState() external {
        ReserveLibrary.updateReserveState(reserve, rateData);
    }
``` 

`ReserveLibrary.updateReserveState()` ->

```js
    function updateReserveState(ReserveData storage reserve,ReserveRateData storage rateData) internal {
        updateReserveInterests(reserve, rateData);
    }
```

`ReserveLibrary.updateReserveInterests()` ->

```js
    function updateReserveInterests(ReserveData storage reserve,ReserveRateData storage rateData) internal {
        uint256 timeDelta = block.timestamp - uint256(reserve.lastUpdateTimestamp);
        if (timeDelta < 1) {
@->           return;
        }

        uint256 oldLiquidityIndex = reserve.liquidityIndex;
        if (oldLiquidityIndex < 1) revert LiquidityIndexIsZero();

        // Update liquidity index using linear interest
        reserve.liquidityIndex = calculateLiquidityIndex(
            rateData.currentLiquidityRate,
            timeDelta,
            reserve.liquidityIndex
        );

        // Update usage index (debt index) using compounded interest
        reserve.usageIndex = calculateUsageIndex(
            rateData.currentUsageRate,
            timeDelta,
            reserve.usageIndex
        );

        // Update the last update timestamp
@->    reserve.lastUpdateTimestamp = uint40(block.timestamp);
        
        emit ReserveInterestsUpdated(reserve.liquidityIndex, reserve.usageIndex);
    }
```

The problem is that `updateReserveInterests` is also being called inside `deposit()`, `withdraw()` and `updateInterestRatesAndLiquidity()` functions of `LendingPool.sol`. 

## Vulnerability Details

1. An attacker sees transactions of `deposit()`, `withdraw()` or `updateInterestRatesAndLiquidity()` in mempool.
2. He will frontrun those txs, by paying extra gas fee calling `updateState()` -

```js
    function updateState() external {
        ReserveLibrary.updateReserveState(reserve, rateData);
    }
```
3. This will update set lastUpdateTimestamp to current block.timestamp `reserve.lastUpdateTimestamp = uint40(block.timestamp);` .

4. Now when the any of transaction `deposit()`, `withdraw()` or `updateInterestRatesAndLiquidity()` will be executed, they will return without performing any operation because `timeDelta` will be 0 -

```js
        uint256 timeDelta = block.timestamp - uint256(reserve.lastUpdateTimestamp);
        if (timeDelta < 1) {
           return;
        }
```
5. Attacker can perform this attack operation everytime, causing unintended behaviors.

## Impact

- Greifing attack for functions `deposit()`, `withdraw()` or `updateInterestRatesAndLiquidity()`.
- Leading to unexpected behavior, and also potentially can cause loss of funds to both protocol and user.

## Tools Used

Manual

## Recommendations

Put access modifier on `updateState()` function, for admin or owner.
## <a id='L-03'></a>L-03. Boost delegation to same address cannot be repeated again.            



## Summary

**Contract** - BoostController.sol

1. There is function `removeBoostDelegation()` which is used to remove the boostdelgation from user.
2. After removal of delegation, if user wishes to delegate again to same address, he won't be able to do that.
3. Because of following check -

```js
if (delegation.amount > 0) revert BoostAlreadyDelegated();
```

1. this problem arises because in `removeBoostDelegation()`, `delegation.amount` amount not resetting to 0.

`BoostController.sol::delegateBoost()` ->

```js
    function delegateBoost(
        address to,
        uint256 amount,
        uint256 duration
    ) external override nonReentrant {
        if (paused()) revert EmergencyPaused();
        if (to == address(0)) revert InvalidPool();
        if (amount == 0) revert InvalidBoostAmount();
        if (duration < MIN_DELEGATION_DURATION || duration > MAX_DELEGATION_DURATION) 
            revert InvalidDelegationDuration();
        
        uint256 userBalance = IERC20(address(veToken)).balanceOf(msg.sender);
        if (userBalance < amount) revert InsufficientVeBalance();

        UserBoost storage delegation = userBoosts[msg.sender][to];

@->     if (delegation.amount > 0) revert BoostAlreadyDelegated();
        
        delegation.amount = amount;
        delegation.expiry = block.timestamp + duration;
        delegation.delegatedTo = to;
        delegation.lastUpdateTime = block.timestamp;
        
        emit BoostDelegated(msg.sender, to, amount, duration);
    }
```

`BoostController.sol::removeBoostDelegation()` ->

```js
    function removeBoostDelegation(address from) external override nonReentrant {
        UserBoost storage delegation = userBoosts[from][msg.sender];
        if (delegation.delegatedTo != msg.sender) revert DelegationNotFound();
        if (delegation.expiry > block.timestamp) revert InvalidDelegationDuration();
        
        PoolBoost storage poolBoost = poolBoosts[msg.sender];
        if (poolBoost.totalBoost >= delegation.amount) {
            poolBoost.totalBoost -= delegation.amount;
        }
        if (poolBoost.workingSupply >= delegation.amount) {
            poolBoost.workingSupply -= delegation.amount;
        }
        poolBoost.lastUpdateTime = block.timestamp;
        
        emit DelegationRemoved(from, msg.sender, delegation.amount);
        delete userBoosts[from][msg.sender];
    }
```

As we can see, inside `removeBoostDelegation` the `delegation.amount` amount is not re-setting to 0

## Vulnerability Details

Same as above.

## Impact

User will never be able to delegate his boost again to same address, twice. Hence breaking the code functionality.

## Tools Used

Maanual

## Recommendations

in `removeBoostDelegation()` set `delegation.amount = 0`.

## <a id='L-04'></a>L-04. crvUSD depositors not getting their accrued interest after withdrawel from LendingPool.            



## Summary

Doc statement -

\| By doing so, user will receives a RToken that represents such deposit + any accrued interest. Contrary to the DebtToken,
\| those RToken are transferable.

The withdraw function in `LendingPool.sol` is as follow -

```js
    function withdraw(uint256 amount) external nonReentrant whenNotPaused onlyValidAmount(amount) {
        if (withdrawalsPaused) revert WithdrawalsArePaused();

        // Update the reserve state before the withdrawal
        ReserveLibrary.updateReserveState(reserve, rateData);

        // Ensure sufficient liquidity is available
        _ensureLiquidity(amount);

        // Perform the withdrawal through ReserveLibrary
        (uint256 amountWithdrawn, uint256 amountScaled, uint256 amountUnderlying) = ReserveLibrary.withdraw(
            reserve,   // ReserveData storage
            rateData,  // ReserveRateData storage
            amount,    // Amount to withdraw
            msg.sender // Recipient
        );

        // Rebalance liquidity after withdrawal
        _rebalanceLiquidity();

        emit Withdraw(msg.sender, amountWithdrawn);
    }
```

**`ReserveLibrary.withdraw()`** ->

```js
    function withdraw(
        ReserveData storage reserve,
        ReserveRateData storage rateData,
        uint256 amount,
        address recipient
    ) internal returns (uint256 amountWithdrawn, uint256 amountScaled, uint256 amountUnderlying) {
        if (amount < 1) revert InvalidAmount();

        // Update the reserve interests
        updateReserveInterests(reserve, rateData);

        // Burn RToken from the recipient - will send underlying asset to the recipient
        (uint256 burnedScaledAmount, uint256 newTotalSupply, uint256 amountUnderlying) = IRToken(reserve.reserveRTokenAddress).burn(
            recipient,              // from
            recipient,              // receiverOfUnderlying
            amount,                 // amount
            reserve.liquidityIndex  // index
        );
        amountWithdrawn = burnedScaledAmount;
        
        // Update the total liquidity and interest rates
        updateInterestRatesAndLiquidity(reserve, rateData, 0, amountUnderlying);

        emit Withdraw(recipient, amountUnderlying, burnedScaledAmount);

        return (amountUnderlying, burnedScaledAmount, amountUnderlying);
    }
```

**`IRToken.burn()`** ->

```js
    function burn(
        address from,
        address receiverOfUnderlying,
        uint256 amount,
        uint256 index
    ) external override onlyReservePool returns (uint256, uint256, uint256) {
        if (amount == 0) {
            return (0, totalSupply(), 0);
        }

        uint256 userBalance = balanceOf(from);  

        _userState[from].index = index.toUint128();

        if(amount > userBalance){
            amount = userBalance;
        }

        uint256 amountScaled = amount.rayMul(index);

        _userState[from].index = index.toUint128();

        _burn(from, amount.toUint128());

        if (receiverOfUnderlying != address(this)) {
@->          IERC20(_assetAddress).safeTransfer(receiverOfUnderlying, amount);
        }

        emit Burn(from, receiverOfUnderlying, amount, index);

        return (amount, totalSupply(), amount);
    }
```

## Vulnerability Details

As we can see -

1. when withdrawel of crvUSD happens, the receiver get only the crvUSD amount he deposited = burned RTokens.
2. user not getting any accured interest.
3. Apart from that devs may be assuming that the user can withdraw the accured intrest through any other function (via pull mechanism), but there isn't any function like that, neither there is any state varibale that tracks how much interest a perticular user has earned.

## Impact

* User not getting his accured interest, the incentive behind depositing crvUSD in lending pool.

## Tools Used

Manual

## Recommendations

Implement `reserve.liquidityIndex` on withdrawel amount, as it deals with interets accured of deposited crvUSD in pool.

## <a id='L-05'></a>L-05. There is no function in Controller to call `setWeeklyEmission()`.            



## Summary

`setWeeklyEmission()` is set to be called from controller, but there is no function in controller that's calling this.

## Vulnerability Details
`setWeeklyEmission()` can't be called, hence period emission can't be set.
## Impact
period emission (weekly) can't be set.
## Tools Used
Manual

## Recommendations
Implement a function a controller, that's calling the `setWeeklyEmission()`
## <a id='L-06'></a>L-06. zero slippage in `LendingPool.sol::_withdrawFromVault()` can lead to DOS.            



## Summary

The internal function `_withdrawFromVault()` is used to withdraw tokens from vault, incase lending pool's crvUSD balance is low.

```js
    function _withdrawFromVault(uint256 amount) internal {
        // @audit - hardcoded slippage can lead to fund stuck forever, as loss margin is set to 0.
        curveVault.withdraw(amount, address(this), msg.sender, 0, new address[](0));
        totalVaultDeposits -= amount;
    }
```
The params for withdraw function are :-
```js
    /**
     * @notice Withdraws assets from the vault
     * @param assets Amount of assets to withdraw
     * @param receiver Address to receive the assets
     * @param owner Owner of the shares
     * @param maxLoss Maximum acceptable loss in basis points
     * @param strategies Optional specific strategies to withdraw from
     * @return shares Amount of shares burned
     */
    function withdraw(
        uint256 assets,
        address receiver,
        address owner,
        uint256 maxLoss,
        address[] calldata strategies
    ) 
```

The `maxLoss` as mentioned, it's "Maximum acceptable loss in basis points", it's acts like slippage protection or clearnce value, but this can lead to DOS of this function, as there is no scope for loss margin, the market must align with requirment, which is not ideal.

In other words if market fluctates very little, the `_withdrawFromVault` will revert due to 0 tolerance in loss.

## Vulnerability Details

loss margin set to 0, can lead to DOS.

## Impact

- `_withdrawFromVault` is used inside `_rebalanceLiquidity`.
- `_rebalanceLiquidity` is used inside `borrow()`, `withdraw()` and `deposit()` functions of lending pool contract.
- means there is high probability that `borrow()`, `withdraw()` and `deposit()` also lead to DOS.

## Tools Used

Manual

## Recommendations

Set loss margin to non-zero value like 5% or 6% etc.

## <a id='L-07'></a>L-07. An attacker can call `veRAACToken.sol::recordVote()` on behalf of voter, leading to unintended vote to a proposal.            



## Summary

The `recordVote()` function is used to track user vote, but it can be called by anyOne.

```js
    function recordVote(
        address voter, 
        uint256 proposalId
    ) external {
        if (_hasVotedOnProposal[voter][proposalId]) revert AlreadyVoted();
        _hasVotedOnProposal[voter][proposalId] = true;
        
        uint256 power = getVotingPower(voter);
        emit VoteCast(voter, proposalId, power);
    }
```
## Vulnerability Details

- An attacker can call `recordVote()` function without voter consent.
- Means even if voter doesn't wishes to vote; his vote is being used in proposal favor.
- An attacker can create a malicious proposal and then do above process.
- Causing threat to protocol's system.

## Impact

- voting on malicious proposal without user consent .

## Tools Used

Manual

## Recommendations

replace `_hasVotedOnProposal[voter][proposalId]` with `_hasVotedOnProposal[msg.sender][proposalId]`
## <a id='L-08'></a>L-08. Incorrect initialization of minBoost or maxBoost in `BaseGuage` contract's constructor.            



## Summary

The constructor initialization in `BaseGuage.sol` is as follow -

```js
    constructor(
        address _rewardToken,
        address _stakingToken, // $mynotes - _stakingToken is veRAAC.
        address _controller,
        uint256 _maxEmission,
        uint256 _periodDuration
    ) {
        rewardToken = IERC20(_rewardToken);
        stakingToken = IERC20(_stakingToken);
        controller = _controller;
        
        // Initialize roles
        _grantRole(DEFAULT_ADMIN_ROLE, msg.sender);
        _grantRole(CONTROLLER_ROLE, _controller);
        
        // Initialize boost parameters
        boostState.maxBoost = 25000; // 2.5x
@->     boostState.minBoost = 1e18; // @audit - incorrect initilization.
        boostState.boostWindow = 7 days;

        uint256 currentTime = block.timestamp;

        uint256 nextPeriod = ((currentTime / _periodDuration) * _periodDuration) + _periodDuration;
        
        // Initialize period state
        periodState.periodStartTime = nextPeriod;
        periodState.emission = _maxEmission;

        TimeWeightedAverage.createPeriod(
            periodState.votingPeriod, // empty object
            nextPeriod,  // $mynotes - startTime 
            _periodDuration, // $mynotes - duration
            0, // $mynotes - initialValue
            10000 // VOTE_PRECISION // $mynotes - weight
        );
    }
```
The maxBoost = 25000 and minBoost = 1e18, which is totally incorrect as minimum value is greater than maximum value.


## Vulnerability Details

This incorrect initilization of min and max boost will lead to reverting of `calculateBoost()`, because of line -
```js
uint256 boostRange = params.maxBoost - params.minBoost;
```
As, (25000 - 1e18) will always revert. 

Code snippet of `calculateBoost()` function.
```js
    function calculateBoost(
        uint256 veBalance,
        uint256 totalVeSupply,
        BoostParameters memory params
    ) internal pure returns (uint256) {
        // Return base boost (1x = 10000 basis points) if no voting power
        if (totalVeSupply == 0) {
            return params.minBoost;
        }

        // Calculate voting power ratio with higher precision
        uint256 votingPowerRatio = (veBalance * 1e18) / totalVeSupply;
        // Calculate boost within min-max range
@ ->    uint256 boostRange = params.maxBoost - params.minBoost;
        uint256 boost = params.minBoost + ((votingPowerRatio * boostRange) / 1e18);
        
        // Ensure boost is within bounds
        if (boost < params.minBoost) {
            return params.minBoost;
        }
        if (boost > params.maxBoost) {
            return params.maxBoost;
        }
        
        return boost;
    }
```

## Impact
1. `calculateBoost()` function is necessary for determining reward for user. if this function will revert, it means user funds
stuck.
2. Also boost update function will not work, as it's also dependent on `calculateBoost()`.

## Tools Used
Manual

## Recommendations
Change -
```diff
- boostState.maxBoost = 25000; // 2.5x
+ boostState.maxBoost = 25 * 1e17; // 2.5x
```
## <a id='L-09'></a>L-09. `FeeCollector.sol::updateFeeType` will not work for all fee types.            



## Summary

`BASIS_POINTS = 10_000`

**FeeCollector.sol::updateFeeType()** ->

```js
    function updateFeeType(uint8 feeType, FeeType calldata newFee) external override {
        if (!hasRole(FEE_MANAGER_ROLE, msg.sender)) revert UnauthorizedCaller();
        if (feeType > 7) revert InvalidFeeType();
        
@ ->    if (newFee.veRAACShare + newFee.burnShare + newFee.repairShare + newFee.treasuryShare != BASIS_POINTS) {
            revert InvalidDistributionParams();
        }
        
        feeTypes[feeType] = newFee;
        emit FeeTypeUpdated(feeType, newFee);
    }
```

for feeTypes 6 and 7 it will not work as -

```js
        // Buy/Sell Swap Tax (2% total)
        feeTypes[6] = FeeType({
            veRAACShare: 500,     // 0.5%
            burnShare: 500,       // 0.5%
            repairShare: 1000,    // 1.0%
            treasuryShare: 0
        });
        
        // NFT Royalty Fees (2% total)
        feeTypes[7] = FeeType({
            veRAACShare: 500,     // 0.5%
            burnShare: 0,
            repairShare: 1000,    // 1.0%
            treasuryShare: 500    // 0.5%
        });
```

for feeType 6, sumation will be 2000 != BASIS\_POINTS; and for 7 sumation will also be 2000 != BASIS\_POINTS.

## Vulnerability Details

## Impact

The function will revert for feeType 6 and 7, and they can never be updated.

## Tools Used

Manual

## Recommendations

Do something for type 6 and 7 as well.



