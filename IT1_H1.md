**StableCoinPrice cannot be updated in the lockManager**


**Description:** 
The StableCoinPrice price is maintained in the LockManager manually through a proposal and approval framework. The "pricefeed" roles have entitlements to propose a revised StableCoin Price which will then be approved by other with similar role. The StableCoin price update is submitted as a proposal using the structure below. Note the mapping for approvals and disapprovals:

```
struct StableCoinUpdateProposal {
    uint32 proposedDate;
    address proposer;
    address[] contracts;
    uint256 proposedPrice;
    mapping(address => bool) approvals;
    mapping(address => bool) disapprovals;
    uint8 approvalsCount;
    uint8 disapprovalsCount;
}
```

When a struct contains a storage mapping, deleting the struct does not delete the entries inside the mapping. So, if the structure is again reused, there is a risk of stale data being used in subsequent interactions. When the USD price proposal is submitted by address 0x1234, the approvals mapping for 0x1234 is flagged as true.

stableCoinUpdateProposal.approvals[msg.sender] = true;
Assuming that the proposal hits the threshold to update the USD price and executed, post execution, the proposal is deleted as below in _execUSDPriceUpdate. The delete will not be able to clear the mapping entries and hence, 0x1234 will remain in the approvals mapping.

```
function _execStableCoinPriceUpdate() internal {
    if (
        stableCoinUpdateProposal.approvalsCount >= SCAPPROVE_THRESHOLD &&
        stableCoinUpdateProposal.disapprovalsCount < SCDISAPPROVE_THRESHOLD
    ) {
        // ...
        delete stableCoinUpdateProposal;
    }
}
```

As the next StableCoin price proposal is submitted, 0x1234 will not be able to approve it because of the following condition in approveStableCoinPrice since the delete did not clear the mapping entry and hence, it will be accounted as already approved.

```
if (stableCoinUpdateProposal.approvals[msg.sender])
      revert SCProposalAlreadyApprovedError();
```

This means that once a given address approved or disapproved a proposal, any interactions on following proposals will revert, effectively preventing the USD price updates.

**Recommendation(s)**: 
Consider using an array to track votes, or change to a nested mapping in the struct to include the proposal round. This will ensure that previous votes won't remain on a new proposal.