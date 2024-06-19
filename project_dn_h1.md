### Newly accrued rewards will be lost during stake/harvest ###


Description : Token reward rate and rewards accrued if any are update everytime new tokens are staked or harvested through user interaction. The `updateReward()` modifier with flag passed as `true` updates the reward per Token and also computes additional rewards earned due to lapse of time between now and last time the reward rate was updated. 

```solidity
modifier updateRewardXXX(bool updateEarned) {
        StakedXXXXStorage storage $ = _getStakedXXXXXoStorage();

        $.rewardPerTokenStored = rewardPerToken();
==>     $.lastUpdateTime = lastTimeRewardApplicable(); //@audit - here last update time is already update to now or applicable time

        if (updateEarned) {
            $.rewards = earned(); // @audit - time lapsed will be zero as last update time was already update and hence zero additional rewards
            $.rewardPerTokenPaid = $.rewardPerTokenStored;
        }
        _;
}
```
The `earned` function computes the additional rewards earned during the lapsed time as long as the current time is before the finish time(time until when the rewards will be distributed). In the `rewardPerToken` function, the lapsed time is accounted to compute the additional rewards on top of  `$.rewardPerTokenStored`.

```solidity
   function earned() public view returns (uint256) {
        StakedXXXXStorage storage $ = _getStakedXXXXXoStorage();
        return
            (($.totalStaked * (rewardPerToken() - $.rewardPerTokenPaid)) /
                1e18) + $.rewards;
    }
  
    function rewardPerToken() public view returns (uint256) {
       StakedXXXXStorage storage $ = _getStakedXXXXXoStorage();
        uint256 _totalStaked = $.totalStaked;

        if (_totalStaked == 0) {
            return $.rewardPerTokenStored;
        }

        return
            $.rewardPerTokenStored +
            ((((lastTimeRewardApplicable() - $.lastUpdateTime) * $.rewardRate) * // audit - as $.lastUpdateTime is already synced to lastTimeRewardApplicable(), this will result in 0 increment of rewards earned.
                1e18) / _totalStaked);
    }

```
But as the `$.lastUpdateTime` was already synced to the `lastTimeRewardApplicable()` in the `updateReward` modifier, before entering the logic to compute additional rewards,  there will be no time lapse accounted as time difference is `0`. And hence `0` additional rewards will be accounted on top of already stored `$.rewardPerTokenStored`.

```solidity
            $.rewardPerTokenStored +
            ((((lastTimeRewardApplicable() - $.lastUpdateTime) * $.rewardRate) *
                1e18) / _totalStaked);
```
This leads to staker losing their earned tokens.


Recommendation(s):
To account the newly earned rewards correctly, update the` $.lastUpdateTime` after computing the earned rewards logic as below.

```solidity
 modifier updateReward(bool updateEarned) {
         StakedXXXXStorage storage $ = _getStakedXXXXXoStorage();

        $.rewardPerTokenStored = rewardPerToken();
       
        if (updateEarned) {
            $.rewards = earned();
            $.rewardPerTokenPaid = $.rewardPerTokenStored;
        }
       $.lastUpdateTime = lastTimeRewardApplicable(); // Audit
        _;
    }
```

