### Incorrect computation Reward due to incorrect handling of decimal resolution ###
The logic for computation is rewards is not correctly handling the rewards for tokens of different resolution.
```
   function _updateReward(
        address _iToken,
        address _account,
        bool _isBorrow
    ) internal {
        require(_account != address(0), "Invalid account address!");
        require(controller.hasiToken(_iToken), "Token has not been listed");

        uint256 _iTokenIndex;
        uint256 _accountIndex;
        uint256 _accountBalance;
        if (_isBorrow) {
            _iTokenIndex = distributionBorrowState[_iToken].index;
            _accountIndex = distributionBorrowerIndex[_iToken][_account];
            _accountBalance = IiToken(_iToken)
                .borrowBalanceStored(_account)
                .rdiv(IiToken(_iToken).borrowIndex());

            // Update the account state to date
            distributionBorrowerIndex[_iToken][_account] = _iTokenIndex;
        } else {
            _iTokenIndex = distributionSupplyState[_iToken].index;
            _accountIndex = distributionSupplierIndex[_iToken][_account];
            _accountBalance = IERC20Upgradeable(_iToken).balanceOf(_account);

            // Update the account state to date
            distributionSupplierIndex[_iToken][_account] = _iTokenIndex;
        }

        uint256 _deltaIndex = _iTokenIndex.sub(_accountIndex);
===>    // Lets say, balance was 1000 and delta is 10, it will compute differently for
===>    // different tokens based on decimal resolution.

==>     uint256 _amount = _accountBalance.rmul(_deltaIndex);

        if (_amount > 0) {
            reward[_account] = reward[_account].add(_amount);

            emit RewardDistributed(_iToken, _account, _amount, _accountIndex);
        }
    }
```
The issue is in the rmul which does not consider the decimal resolution of the token.

```
    uint256 private constant BASE = 10 ** 18;

    function rmul(uint256 x, uint256 y) internal pure returns (uint256 z) {
        z = x.mul(y).div(BASE);
    }
```


**Demo**
DAI has 18 decimal places
USDT has 6 decimal places

Lets say, user had 1000 tokens each as his balance for the above tokens.

Lets say, delta index was 10.

Then, considering rmul divides by decimal resolution 10**18

Reward computation formulae:
============================
uint256 _amount = _accountBalance.rmul(_deltaIndex);

DAI:
  _amount = 1000,000000000000000000 * 10/ 1000000000000000000;
  _amount = 10000 rewards
  
USDT:
  _amount = 1000,000,000 * 10/ 1000000000000000000;
  _amount = 0.00000001 rewards 
 
As the tokens can have different decimal resolutions, dividing by 10**18 is incorrect. Instead, it should be decimals of the token.

That means
DAI:
  _amount = 1000,000000000000000000 * 10/ itoken.decimals(); 10**18
  _amount = 10000 rewards
  
USDT:    
  _amount = 1000,000000 * 10/ itoken.decimals(); 10**6
  _amount = 10000 rewards





USDT

