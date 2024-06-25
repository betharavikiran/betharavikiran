### Slashing should impact the price of XXXStable to that extent of collateral lost, but instead losses will be passed to the protocol

**description**: When a slash occurs in `Beacon layer` due to violation by a validator, a portion of the LST tokens are removed reduction in the total balance for the account. Since `LST tokens` is the collateral against which the XXXStable minted and is in circulation, to hold the peg, the `XXXStable` to that extent should be burned/removed from circulation to honour the collateral ratio.

SInce at the time of deposits, the `XXXStable` are minted in favour of the original depositor, the protocol does not have any control on those XXXStable. So, the concern is whose 'XXXStableUSC' should be burnt.

```solidity
   function mint( // public function
    address token,
    uint256 amount
  ) external whenMintNotPaused nonReentrant onlyWhenMintableOrBurnable returns (uint256) {
   ....
    return _mint(amountAfterFee, token);
  }

  function _mint(uint256 amount, address token) private returns (uint256) { // internal function
   ...
    XXXStable.mint(msg.sender, XXXStableAmountToMint);
    return XXXStableAmountToMint;
  }
```
As there is no option to burn across holders of `XXXStable`, the approach is to devalue the `XXXStable` in circulation to the extent of collateral lost via slashing.  While the underlying collateral was slashed, it **will not** impact the XXXStable price as the reserve value is computed based on state variable `totalDeposited` and not the actual balance after slashing. Refer the how the `totalReserveValue` is computed by summing up the `reserve` across LSTs. Hence, slash does not impact the `XXXStable' price.

Note the below reserve computation for each LST is based on `totalDeposited` state variables.

**RebasingAdapter**:

```solidity
   function getReserveValue() external view returns (uint256) {
    uint256 ethPrice = priceFeedAggregator.peek(ExternalContractAddresses.WETH);
===> uint256 totalDepositedValue = Math.mulDiv(totalDeposited, ethPrice, 1e18);
    return totalDepositedValue;
  }
```
**ReserveHolderV2**:
```solidity
   function getReserveValue() public view returns (uint256) {
    uint256 totalReserveValue;

    for (uint256 i = 0; i < reserveAssets.length; i++) {
      totalReserveValue += reserveAdapters[reserveAssets[i]].getReserveValue();
    }

    return totalReserveValue;
  }
```

Because of the above, the `XXXStable` price will not have any impact and as XXXStable is redeemed by individual uses, the slash losses will eventually be passed on to the protocol.

**impact**: any slash related loss will be passed to the protocol.

**recommendation**: 
When the `totalDeposited` falls below the `balance`, resetting the `totalDeposited` to `new balance` will help.
Should consider if impact on new deposits or redemptions if any.
