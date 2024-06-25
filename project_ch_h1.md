### Slashing in beacon layer will cause a revert while claiming rewards

description: In Ethereum’s PoS ecosystem, slashing refers to the process of penalizing a validator for misbehaving. Validators, entrusted with the tasks of attesting and proposing blocks, are essential for maintaining the network’s smooth operations. If a validator is found to behave maliciously or dishonestly, it may be slashed, which means that a portion of its deposit is destroyed by the network as a punishment.

Irrespective the liquidity staking platform, whether it is offering rebasing LST or non rebasing LST, the slash will adversely impact the available balance.

The problem arises in the claimRewards() function in both XXXRebasingLstAdapter and XXXNonRebasingLstAdapter contracts, where the logic assumes that the available balance will always be greater than totalDeposited. Essentially, it is true as long as there is no slashing. But, once a slash is applied by the beacon layer, the balance should be less than totalDeposited tracked in the contract.

Hence, the xxxclaimRewards() function will revert for XXXRebasingLstAdapter and XXXNonRebasingLstAdapter contracts.

XXXRebasingLstAdapter:

   function xxxclaimRewards(address receiver) external onlyReserveHolder returns (uint256) {
    uint256 balance = IERC20(asset).balanceOf(address(this));
===>  uint256 reward = balance - totalDeposited;
    IERC20(asset).safeTransfer(receiver, reward);

    emit ClaimRewards(receiver, reward);
    return reward;
  }
XXXNonRebasingLstAdapter:

function xxxclaimRewards(address receiver) external onlyReserveHolder returns (uint256) {
    IChainlinkEthAdapter chainlinkEthAdapter = IChainlinkEthAdapter(address(priceFeedAggregator.priceFeeds(asset)));
    uint256 assetEthExchangeRate = chainlinkEthAdapter.exchangeRate();

    uint256 totalBalanceInEth = Math.mulDiv(
      IERC20(asset).balanceOf(address(this)),
      assetEthExchangeRate,
      IERC20Metadata(asset).decimals()
    );
===> uint256 totalRewardsInEth = totalBalanceInEth - totalDeposited;
    uint256 totalRewardsInAsset = Math.mulDiv(totalRewardsInEth, 1e18, assetEthExchangeRate);

    IERC20(asset).safeTransfer(receiver, totalRewardsInAsset);

    emit ClaimRewards(receiver, totalRewardsInAsset);
    return totalRewardsInAsset;
  }
impact: Reward claiming functions will revert

recommendation(s): Update the logic to first check if balance is greater than totalDeposited.

If balance is greater than totalDeposited, there are rewards to claim and process the rewards.
if balance is less than totalDeposited, there is nothing accrued so far and hence nothing to claim and skip the processing and return 0.
