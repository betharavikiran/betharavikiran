### NFT Tokens can be stuck permanently ###

XXXERC721Bridge::bridgeERC721To() does not validate for the msg.sender to be an EOA only.
As such, it is possible that the deposit of the ERC721 is made by a Smart contract. While ,locking into the bridge,
ERC721::transferFrom() is called which bypasses the check for the address receiving the token.


```
   function bridgeERC721To(
        address _localToken,
        address _remoteToken,
        address _to,
        uint256 _tokenId,
        bytes calldata _extraData
    )
        external
        payable
    {
        require(_to != address(0), "ERC721Bridge: nft recipient cannot be address(0)");

===>    _initiateBridgeERC721(_localToken, _remoteToken, msg.sender, _to, _tokenId, _extraData);
    }


     function _initiateBridgeERC721(
        address _localToken,
        address _remoteToken,
        address _from,
        address _to,
        uint256 _tokenId,
        bytes calldata _extraData
    )
        internal
        override
    {
        require(_remoteToken != address(0), "L1ERC721Bridge: remote token cannot be address(0)");
        require(msg.value == flatFee, "L1ERC721Bridge: bridging ERC721 must include sufficient ETH value");

        (bool sent, ) = payable(LIQUIDITY_POOL).call{value: msg.value}("");
        require(sent, "L1ERC721Bridge: failed to send ETH to liquidity pool");

        // Lock token into bridge
        deposits[_localToken][_remoteToken][_tokenId] = true;
===>    IERC721(_localToken).transferFrom(_from, address(this), _tokenId);

        // Send calldata into L2
        emit ERC721BridgeInitiated(_localToken, _remoteToken, _from, _to, _tokenId, _extraData);
    }
       
```

hence on bridging, the token will be successfully deposited into the bridge.

Later when finalizeBridgeERC721() is initiated, the deposited NFT should be returned back to the depositor account. To transfer the deposited token back to the owner, the below function is called.

   // When a withdrawal is finalized on L1, the L1 Bridge transfers the NFT to the
   // withdrawer.
   IERC721(_localToken).safeTransferFrom(address(this), _to, _tokenId);

```
   function finalizeBridgeERC721(
        address _localToken,
        address _remoteToken,
        address _from,
        address _to,
        uint256 _tokenId,
        bytes calldata _extraData
    )
        external
        onlyRole(DEFAULT_ADMIN_ROLE)
    {
        require(paused() == false, "L1ERC721Bridge: paused");
        require(_localToken != address(this), "L1ERC721Bridge: local token cannot be self");

        // Checks that the L1/L2 NFT pair has a token ID that is escrowed in the L1 Bridge.
        require(
            deposits[_localToken][_remoteToken][_tokenId] == true,
            "L1ERC721Bridge: Token ID is not escrowed in the L1 Bridge"
        );

        // Mark that the token ID for this L1/L2 token pair is no longer escrowed in the L1
        // Bridge.
        deposits[_localToken][_remoteToken][_tokenId] = false;

        // When a withdrawal is finalized on L1, the L1 Bridge transfers the NFT to the
        // withdrawer.
===>    IERC721(_localToken).safeTransferFrom(address(this), _to, _tokenId);

        // slither-disable-next-line reentrancy-events
        emit ERC721BridgeFinalized(_localToken, _remoteToken, _from, _to, _tokenId, _extraData);
    }
   
```   

here, in this case, the to address is the receipent of the token. Since the transfer from the bridge contract to the receipient is made via safeTransferFrom() function, this function will validate if the receiving address implements IERC721Receiver interface.
If the receiving address does not implement IERC721Receiver, the transaction will revert.
Safely transfers tokenId token from from to to, checking first that contract recipients are aware of the ERC721 protocol to prevent tokens from being forever locked.

[ref](https://docs.openzeppelin.com/contracts/5.x/api/token/erc721#IERC721-safeTransferFrom-address-address-uint256-)

As the validation for the depositor can potentially be bypassed during deposit, but during withdrawal, due to safeTransferFrom(), the check for the receipient to be aware of ERC721 protocol is enforcing, the tokens will be permanently stuck in the bridge contract.

**Validation steps**

1.Deposit the NFT token into the bridge contract using a smart contract by calling bridgeERC721To().

2.The smart contract should not be implementing IERC721Receiver interface

3.Trying to withdraw the deposited token back by calling finalizeBridgeERC721()

4.The transaction should revert.
