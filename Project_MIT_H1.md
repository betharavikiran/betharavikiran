### Adapter::bridgeAsset() can be called by attacker to transact on one side of the bridge ###

### Description ###
bridgeAsset() function is used to send msg to the destination chain via bridge.
This function is being called by Crosschain deposit manager host in the _handle() function. The handle function has all the checks before the message is sent to the bridge.

The vulnerability is that for all the below three adaptors, there is no access restriction for bridgeAsset() function. This means any one can call this function without meeting the other checks implemented in dispatch() and handle() functions.

The cross bridge communication functions should be restrictions with permissions so that only qualified contracts can call after all checking is done. Leave this open ended means, attacker can dispatch messages on one side of the bridge breaking the symmetry of tokens across the bridges. Its means, without send tokens from one chain, there is a possibility to make a deposit on the other chain.

ArbitrumBridgeAdapter
OptimismBridgeAdapter
PolygonZkEvmBridgeAdapter

```
  function bridgeAsset(address destAddr, address l1Asset, address, uint256 amount) external {
        bridge.outboundTransfer(l1Asset, destAddr, amount, maxGas, gasPriceBid, "");
    }
function bridgeAsset(address destAddr, address l1Asset, address l2Asset, uint256 amount) external {
        bridge.depositERC20To(l1Asset, l2Asset, destAddr, amount, minGasLimit, "");
    }
 function bridgeAsset(address destAddr, address l1Asset, address, uint256 amount) external {
        bridge.bridgeAsset(destNetwork, destAddr, amount, l1Asset, forceUpdateGlobalExitRoot, "");
    }
```

Hence each of these bridgeAsset() function on adaptors should be restricted to CCDM Host contract.

### Recommendation ###
Restrict the bridgeAsset() so that it can only be called by Crosschain deposit manager host(CCDM Host) contract.