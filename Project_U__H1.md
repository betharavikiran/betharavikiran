**Attacker can steal reward tokens**

**description**
The rewardDistributor keeps track of the reward tokens for a given account. The rewards are updated for the account based on the type of participation as a borrower or supplier. The updation of the rewards is via updateReward() on the RewardDistributor/V3 contract.

**demo**
The computation is as below.

Based on the parameters passed into the updateReward() external function, if the rewards are for borrow or supplier, the creditable rewards are computed using the account level claim vs token claims so far.

An attacker can abuse the updateReward and claim tokens he is not eligible.

Lets say,

  _iToken = 0xAAA
  _account = 0xATTACKER
  _isBorrow = false
  distributionSupplyState[0xAAA].index = 100;

and the attacker was never a supplier and hence

distributionSupplierIndex[0xAAA][0xATTACKER] = 0;

*So, to abuse the system, the attacker mints/buys 10 tokens of 0xAAA for his account 0xATTACKER. This balance should exist in the account for stealing the tokens.
*

After this, he calls updateReward() function on RewardDistributor/V3 contract and should have as below.


iTokenIndex[100] = distributionSupplyState[0xAAA].index;

The default value for uint256 is 0, and hence distribution suppplier index for attacker account for the token
should be 0.


accountIndex[0] = distributionSupplierIndex[0xAAA][0xATTACKER];

He managed to get an account balance of 10 tokens against his account.


 accountBalance[10] = IERC20Upgradeable(_iToken).balanceOf(_account);

So, the amount of tokens attacker can claim will be as below.


   uint256 _deltaIndex[100] = iTokenIndex.sub(_accountIndex);  //  [100-0]

   uint256 _amount[1000] = accountBalance.rmul(_deltaIndex);  // [10*100] 

The attacker is able to claim 1000 tokens without being a supplier and hence he will be able to claim tokens he is not
eligible for.

**Validation steps**
1) Identify an iToken[0xAAA] to attack, the iToken identified should be configured in the controller.
2) Attacker account[0xATTACKER] should have some balance of iTokens.
3) The current distributionSupplyState index for the iTokens should be greater than 0, say 10 or 100
4) The distributionSupplierIndex for the iToken and Attacker account should be 0 as by default.
5) once all the 4 conditions are true, call updateReward() function on the RewardDistributor/V3 contract with below parameters.
updateReward(0xAAA, 0xATTACKER,false)



```
   
    function _updateRewardX(
        address _iToken,
        address _account,
        bool _isBorrowX
    ) internal {
        require(_account != address(0), "Invalid account address!");
        require(controller.hasiToken(_iToken), "Token has not been listed");

        uint256 _iTokenIndex;
        uint256 _accountIndex;
        uint256 _accountBalance;
        if (_isBorrowX) {
            _iTokenIndex = distributionBorrowState[_iToken].index;
            _accountIndex = distributionBorrowerIndex[_iToken][_account];
            _accountBalance = IiToken(_iToken)
                .borrowBalanceStored(_account)
                .rdiv(IiToken(_iToken).borrowIndex());

            // Update the account state to date
            distributionBorrowerIndex[_iToken][_account] = _iTokenIndex;
        } else {
==>         _iTokenIndex = distributionSupplyState[_iToken].index;
            _accountIndex = distributionSupplierIndex[_iToken][_account];
            _accountBalance = IERC20Upgradeable(_iToken).balanceOf(_account);

            // Update the account state to date
            distributionSupplierIndex[_iToken][_account] = _iTokenIndex;
        }

        uint256 _deltaIndex = _iTokenIndex.sub(_accountIndex);
        uint256 _amount = _accountBalance.rmul(_deltaIndex);

        if (_amount > 0) {
            reward[_account] = reward[_account].add(_amount);

            emit RewardDistributed(_iToken, _account, _amount, _accountIndex);
        }
    }
```
