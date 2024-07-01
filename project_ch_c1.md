### Burn function is not effective to make the correction in price

description: When the price of Eth changes, it impacts the value of collateral held by the XXX Protocol and also the total number of StableCoin in circulation. If the collateral value falls, the protocol will have to burn StableCoin equivalent amount of fall to keep the peg.

  function burn(uint256 amount) external whenBurnNotPaused nonReentrant onlyWhenMintableOrBurnable returns (uint256) {
    uint256 ethPrice = priceFeedAggregator.peek(WETH);

    stableCoin.safeTransferFrom(msg.sender, address(this), amount);
    amount -= (amount * mintBurnFee) / MAX_FEE;
    stableCoin.burn(amount);

    uint256 ethAmountToRedeem = Math.mulDiv(amount, USC_TARGET_PRICE, ethPrice);

    reserveHolder.redeem(ethAmountToRedeem);
    IERC20(WETH).safeTransfer(msg.sender, ethAmountToRedeem);

    emit Burn(msg.sender, amount, ethAmountToRedeem);
    return ethAmountToRedeem;
  }
Let's demonstrate with an example.

 	Fee rate	1.00%	 
 	 	 	 
Events 	Collateral	ETH/USD Price	Stable coin
 	1000	2000	2000000
Eth Price change	1900000	1900	 
Tokens to burn for price correction	 	 	100000
 	 	 	 
Transaction to burn 100000	 	 	 
Burning Stablecoin after fee adjustment	 	 	99000
(1 % Fee)	 	 	 
New Stable coin supply	 	 	1901000
 	 	 	 
Ethereum to Redeem	52.1052631578947	 	 
New balance	947.894736842105	 	1901000
New Value	1801000	 	1901000
As mentioned in the table, there is a reserve of 1000 Eth tokens and Eth price is 2000 Stablecoin.

Hence, total collateral is 1000 * 2000 = 2000000. With USC pegged to 1 USD, there should be 2000000 Stablecoin tokens in circulation.

As the Eth price falls to 1900, the total collateral held by XXXX protocol is 1900000. That means 100000 Stablecoin tokens should be burnt to maintain the peg.

When 100000 Stablecoin tokens are burnt, it will use 1900 as the Eth price.

a) Since XXX protocol is charging 1% fee, instead of burning 100000 Stablecoin tokens, it will only burn 99000 Stablecoin tokens. So, after burning, the Stablecoin token supply will be 1901000. It is more by 1000 Stablecoin that the protocol kept as fee.

[Note] This is already more than 1900 * 1000 ETH tokens.

b) The burn logic takes 99000 Stablecoin and computes the ETH to redeem. This amount is (99000 * 1)/1900 = 52.1052631578947. This much Eth will be redeemed from collateral and sent back to the account burning the Stablecoin.

[Note] This means, now the ETH reserves are not 1000 any more, but instead they are 1000 - 52.1052631578947= 947.894736842105 Eth.

c) Post the burn transactions, if we check the new valuation of collateral vs Stablecoin supply, it will be

Total Collateral = 947.894736842105 * 1900 = 1801000
Stablecoin in circulation = 2000000 - 99000 = 1901000

Post the burn, the collateral is still not sufficient to cover the Stablecoin in supply to hold the peg.

Lets compare the Stablecoin price before and after.

Before Burn:
Total Collateral Value/ Stablecoin in Circulation = 1900000/ 2000000 = 0.95

After Burn:
Total Collateral Value/ Stablecoin in Circulation = 1801000/ 1901000 = 0.94739611

Infact, it further deteriorates after burning the Stablecoin tokens.

impact: Stablecoin moves further away from the peg.

recommendation(s):
