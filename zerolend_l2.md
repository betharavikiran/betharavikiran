**ZeroLockerTimelock::constructor() with empty proposers, executors will leave the contract non functional**

**Description**
ZeroLockerTimelock's constructor does not validate for atleast one entry of proposer and executor in the arrays passed.

```
   constructor(
        uint256 minDelay,
        address[] memory proposers,
        address[] memory executors,
        address admin
    ) {

    ...
       // register proposers and cancellers
        for (uint256 i = 0; i < proposers.length; ++i) {
            _setupRole(PROPOSER_ROLE, proposers[i]);
            _setupRole(CANCELLER_ROLE, proposers[i]);
        }

        // register executors
        for (uint256 i = 0; i < executors.length; ++i) {
            _setupRole(EXECUTOR_ROLE, executors[i]);
        }
```

Incase the proposers array or executors array was passed empty, then the ZeroLockerTimelock contract will not function as it cannot accept new schedules or execute them when due.

As such, in order for this contract to function, there is atleast a need for one proposer and executor at the least should be passed in the constructor. The constructor should validate for existence of atleast one entry each to successfully deploy the contract.

Also, the ZeroLockerTimelock does not provision for management of proposer and executor role at a later point. In order to manage this critical functionality, the admin should have ability to assign and revoke roles when needed.

**Recommendation**:
 It is recommended to add admin functions to assign and revokes these roles for the accounts. Add validation in the constructor to have atleast one proposer and one executor,