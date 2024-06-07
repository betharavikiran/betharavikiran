### Launchpad::getCurrentPhaseInfo() logic for reading the current phase seems buggy ###

### Description ###
getCurrentPhaseInfo() should be returning the current active phase of the launchpad.
The decision is made based on the current time and phase’s endtime. This issue is marked critical as it impact the operation of launchpad based on which phase the users are interacting with the protocol.

As long as the current time is less than phase’s end time, the current phase should be considered active.
Instead the logic returns the next phase.

Refer to the logic below.

 for (uint256 i = 0; i < length; i++) { 
            phaseInfo = phaseInfos[i];
            if (currentBlockTimestamp < phaseInfo.endTime) return (i + 1, phaseInfo); // @audit why i +1 ?
        }
Looking at the logic, there are two issues:

a) when current time is less than current phase end time in the loop, the current phase should be returned
b) incase of the condition meeting the last element of the error, returning i+1 will result in referring non existing elements in the array.

### Recommendation ###
Revise the logic as below to get the current active phase.

 for (uint256 i = 0; i < length; ++i) { 
            phaseInfo = phaseInfos[i];
            if (currentBlockTimestamp < phaseInfo.endTime) return (i , phaseInfo);
 }