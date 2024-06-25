### Price corrective interactions by participants should be allowed when spot price diverges from Target price

**description**: As the price is an external factor controlled by external factor and could move away from target price by external forces. As the spot price moves away from the target price, the corrective actions should be allowed to realign the price back closer to target price.

Example, if the 
```
  spotPrice = 90000000
  targetPrice=100000000
  maxDifference=50000000
```
As the difference between `spotPrice` and `targetPrice` is greater than `maxDifference`, in order to correct the spotPrice which is below targetPrice, burning should be allowed so that XXXStable tokens will be taken out of circulation. As the XXXStable supply reduces, the spot price should start to align back to target price. 

In the opposite case, when spotPrice is greater than targetPrice, adding more XXXStable tokens to supply should be allowed.

But, `onlyWhenMintableOrBurnable()` blocks both the `mint()` and `burn()` operations when the spotPrice moves away from  targetPrice beyond maxDifference. The correction in price at this point is by selling the reserves for Eth by calling `executeArbitrageWithReserveSell()` function.

```solidity
   modifier onlyXXXXWhenMintableOrBurnable() {
    uint256 ethPrice = priceFeedAggregator.peek(WETH);

    uint256 XXXStableSpotPrice = _calculateXXXStableSpotPrice(ethPrice);
    //(90000000,100000000, 50000000)
    // _absDiff(price1, price2) <= delta;
    //  100000000 <=50000000 
    if (!_almostEqualAbs(XXXStableSpotPrice, XXXStable_TARGET_PRICE, maxMintBurnPriceDiff)) {
==>     revert PriceIsNotPegged();
    }

    (, uint256 reserveDiff, uint256 reserveValue) = _getReservesData();
    if (reserveDiff > Math.mulDiv(reserveValue, maxMintBurnReserveTolerance, MAX_PRICE_TOLERANCE)) {
      revert ReserveDiffTooBig();
    }

    _;
  }


  function _almostEqualAbs(uint256 price1, uint256 price2, uint256 delta) internal pure returns (bool) {
   //  100000000 <=50000000 will be false 
    return _absDiff(price1, price2) <= delta;
  }
```

But, if the spotPrice was already high, lets say for demo
```
  spotPrice = 151000000
  targetPrice = 100000000
  maxDifference = 50000000
```
In this case, `executeArbitrageWithReserveSell` will convert assets to reserve increasing the spot price further. This would stall the protocol as the reserves are already consumed and `mint()` and `burn()` functions are blocked by the `onlyXXXXWhenMintableOrBurnable()` modifier.

**recommendation**: 
seperate the conditions for when `mint()` and `burn()` should be blocked
   a) When `spotPrice` is greater than `targetPrice`, there is a possibility to add more tokens to supply which will adjust the price
   b) When `spotPrice` is less than `targetPrice`, `executeArbitrageWithReserveSell()` will help, but providing option to burn gives more flexibility incase of unexpected circumstances.
