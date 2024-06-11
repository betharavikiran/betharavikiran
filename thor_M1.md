### THORChain_Router::batchTransferOutAndCallV5 reuses msg.value across a batch and will revert ###

## Impact
batchTransferOutAndCallV5 function attempts to use msg.value multiple times based on the batch size leading to revert as the contract will not have the balance.

## Proof of Concept
BatchTransferOutAndCallV5 is a batch processing function that allows for executing of swaps in a batch. The function accepts array of TransferOutAndCallData to perform the execution, refer below.   

```
   function batchTransferOutAndCallV5(
    TransferOutAndCallData[] calldata aggregationPayloads
  ) external payable nonReentrant {
    for (uint i = 0; i < aggregationPayloads.length; ++i) {
      _transferOutAndCallV5(aggregationPayloads[i]);
    }
  }
```
The TransferOutAndCallData has fromAsset and fromAmount along with toAsset for swapping. Refer to the below structure.

```
   struct TransferOutAndCallData {
    address payable target;
    address fromAsset;
    uint256 fromAmount;
    address toAsset;
    address recipient;
    uint256 amountOutMin;
    string memo;
    bytes payload;
    string originAddress;
  }
```
But, the bugging logic lies in the _transferOutAndCallV5, where if fromAsset is zero address, them from address is identified as Ether and the call is made on target contract with msg.value for swapOut.

The msg.value is the total Ether value passed into the batch function, but
the actual amount applicable should only be `fromAmount` passed in the structure.
As the array can have many instructions for swapping, if the first one uses msg.value, the subsequent calls for fromAsset == 0 will fail as there will not be enough ether left in the contract to perform the subsequent swaps.

The below logic is buggy due to the amount passed against the value parameter in the below target.call function.  

```
   function _transferOutAndCallV5(
    TransferOutAndCallData calldata aggregationPayload
  ) private {
    if (aggregationPayload.fromAsset == address(0)) {
      // call swapOutV5 with ether
      (bool swapOutSuccess, ) = aggregationPayload.target.call{
        value: msg.value
      }(
       ....suppressed
    }
```

## Tools Used
Manual review

## Recommended Mitigation Steps
a) Modify the _transferOutAndCallV5 function to use aggregationPayload.fromAmount
instead of msg.value to support batch function flow as well.

b) In the entry functions, add validation to check if msg.value covers the fromAmount for all entries with fromAsset == 0. If not, revert the transaction.