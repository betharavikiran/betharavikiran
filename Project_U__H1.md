**Attacker can steal reward tokens**

**description**
The rewardDistributor keeps track of the reward tokens for a given account. The rewards are updated for the account based on the type of participation as a borrower or supplier. The updation of the rewards is via updateReward() on the RewardDistributor/V3 contract.

```
contract XXXRewardDistributorV3 is Initializable, Ownable, IXXXRewardDistributorV3 {
    /// @notice the Reward distribution speed of each iToken
    mapping(address => uint256) public distributionSpeed;
   
    struct DistributionState {
        // Token's last updated index, stored as a mantissa
        uint256 index;
        // The block number the index was last updated at
        uint256 block;
        // The block timestamp the index was last updated at
        uint256 timestamp;
    }

    /// @notice the Reward distribution supply state of each iToken
    mapping(address => DistributionState) public distributionSupplyState;

    /// @notice the Reward distribution state of each account of each iToken
    mapping(address => mapping(address => uint256))
        public distributionSupplierIndex;
    /// @notice the Reward distribution state of each account of each iToken

    /// @notice the Reward distribution speed supply side of each iToken
    mapping(address => uint256) public distributionSupplySpeed;


    /**
     * @notice Add the iToken as receipient
     * @dev Admin function, only controller can call this
     * @param _iToken the iToken to add as recipient
     * @param _distributionFactor the distribution factor of the recipient
     */
    function _addRecipient(
        address _iToken,
        uint256 _distributionFactor
    ) external override onlyController {
        distributionFactorMantissa[_iToken] = _distributionFactor;
        distributionSupplyState[_iToken] = DistributionState({
            index: 0,
            block: block.number, //
            timestamp: block.timestamp
        });
        distributionBorrowState[_iToken] = DistributionState({
            index: 0,
            block: block.number,
            timestamp: block.timestamp
        });

        emit NewRecipient(_iToken, _distributionFactor);
    }

   

    /**
     * @notice Update the iToken's  Reward distribution state
     * @dev Will be called every time when the iToken's supply/borrow changes
     * @param _iToken The iToken to be updated
     * @param _isBorrow whether to update the borrow state
     */
    function updateDistributionState(
        address _iToken,
        bool _isBorrow
    ) external override {
        // Skip all updates if it is paused
        if (paused) {
            return;
        }

 ===>   _updateDistributionState(_iToken, _isBorrow);
    }

    function _updateDistributionState(
        address _iToken,
        bool _isBorrow
    ) internal virtual {
        require(controller.hasiToken(_iToken), "Token has not been listed");

        DistributionState storage state = _isBorrow
            ? distributionBorrowState[_iToken]
            : distributionSupplyState[_iToken]; // .index = 0

        uint256 _speed = _isBorrow
            ? distributionSpeed[_iToken]
            : distributionSupplySpeed[_iToken]; // 2

        uint256 _blockNumber = block.number; 
        uint256 _deltaBlocks = _blockNumber.sub(state.block); // 5145703  6144976
         // time stamp = 1717488333

        if (_deltaBlocks > 0 && _speed > 0) {
            uint256 _totalToken = _isBorrow
                ? IiToken(_iToken).totalBorrows().rdiv(
                    IiToken(_iToken).borrowIndex()
                )
                : IERC20Upgradeable(_iToken).totalSupply();
            uint256 _totalDistributed = _speed.mul(_deltaBlocks);

            // Reward distributed per token since last time
            uint256 _distributedPerToken = _totalToken > 0
                ? _totalDistributed.rdiv(_totalToken)
                : 0;

 ===>       state.index = state.index.add(_distributedPerToken);
        }

        state.block = _blockNumber;
    }

    /**
     * @notice Update the account's Reward distribution state
     * @dev Will be called every time when the account's supply/borrow changes
     * @param _iToken The iToken to be updated
     * @param _account The account to be updated
     * @param _isBorrow whether to update the borrow state
     */
    function updateReward(
        address _iToken,
        address _account,
        bool _isBorrow
    ) external override {
        _updateReward(_iToken, _account, _isBorrow);
    }

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
 ===>        _iTokenIndex = distributionSupplyState[_iToken].index; // 100
            _accountIndex = distributionSupplierIndex[_iToken][_account];
            _accountBalance = IERC20Upgradeable(_iToken).balanceOf(_account);

            // Update the account state to date
            distributionSupplierIndex[_iToken][_account] = _iTokenIndex;
        }

 ===>   uint256 _deltaIndex = _iTokenIndex.sub(_accountIndex);
        uint256 _amount = _accountBalance.rmul(_deltaIndex);

        if (_amount > 0) {
 ===>       reward[_account] = reward[_account].add(_amount);

            emit RewardDistributed(_iToken, _account, _amount, _accountIndex);
        }
    }
}

```




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



