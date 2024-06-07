**StreamedVesting::stakeTo4Year() function accepts dust amounts to be locked in the vest**

**description**
stakeTo4Year() function of StreamedVesting contract allows to lock any amount in a vest for 4 years as long as the amount is greater than 0.

There is an overhead for computing rewards for all the vests and hence, allowing dust to be locked is neither beneficial to end user or to the protocol.

Refer to the below function, where any amount greater than 0 can be locked for 4 years.

```
 function stakeTo4Year(uint256 id) external whenNotPaused {
        VestInfo memory vest = vests[id];
        require(msg.sender == vest.who, "not owner");

        uint256 lockAmount = vest.amount - vest.claimed;

        // update the lock as fully claimed
        vest.claimed = vest.amount;
        vests[id] = vest;

        // check if we can give a 20% bonus for 4 year staking
        uint256 bonusAmount = bonusPool.calculateBonus(lockAmount);
        if (underlying.balanceOf(address(bonusPool)) >= bonusAmount) {
            underlying.transferFrom(// @audit moves into this contract, should the bonus be given to vest owner ?
                address(bonusPool),
                address(this),
                bonusAmount
            );
            lockAmount += bonusAmount;
        }

        // create a 4 year lock for the user
        locker.createLockFor(lockAmount, 86400 * 365 * 4, msg.sender);
    }
 ```

The locker.createLockFor checks for the amount to be greater than 0 in the function below. Locking dust amounts into lockers for 4 years should be prevented. There should be a minimum threshold amount to lock for 4 years. Only amounts above the minimum threshold should be allowed to lock, otherwise the transaction should be reverted.

```
 function _createLock( // @audit done
        uint256 _value,
        uint256 _lockDuration,
        address _to
    ) internal returns (uint256) {
        uint256 unlockTime = ((block.timestamp + _lockDuration) / WEEK) * WEEK; // Locktime is rounded down to weeks

        require(_value > 0, "value = 0"); // dev: need non-zero value
```
**Solution**:

Recommendation is to configure a minimum threshold amount for locking. It can a constant or an admin managed state variable. Only amounts greater than the minimum threshold should be allowed for locking.

Amount below the minimum threshold should revert in stakeTo4Year() function.

example:

   // state variable
   uint256 minLockThreshold = 100

    function stakeTo4Year(uint256 id) external whenNotPaused {
        VestInfo memory vest = vests[id];
        require(msg.sender == vest.who, "not owner");

        uint256 lockAmount = vest.amount - vest.claimed;

        require(lockAmount >=minLockThreshold,"Amount is below Min threshold to be locked");
   
note the require condition above which will revert if the minLockThreshold condition is not met.