 ### Message::encodeBridge() function uses a wrong MsgType, when encoding the data packet for bridge ###

### Description ###
The encodeBridge() helper function in crosschain dm::message library uses an incorrect message type for encoding the bridge message packet.

The functions for encoding and decoding are segregated for message types.

The message type is critical on how it will be handled and hence important to use the correct message type.

enum MsgType {
    Deposit,
    Refund,
    Bridge
}
Refer to the code below where the first parameter to abi.encodePacked should be MsgType.Bridge instead of MsgType.Deposit

```
  function encodeBridge(MsgBridge memory msg_) internal pure returns (bytes memory) {
        return abi.encodePacked(MsgType.Deposit, msg_.receiver, msg_.token, msg_.amount);
    }
```
### Recommendation ###
Revise the encodeBridge function as below.

```
  function encodeBridge(MsgBridge memory msg_) internal pure returns (bytes memory) {
        return abi.encodePacked(MsgType.Bridge, msg_.receiver, msg_.token, msg_.amount);
    }
```    