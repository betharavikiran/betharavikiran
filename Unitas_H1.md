NFT Tokens can be stuck permanently

AviERC721Bridge::bridgeERC721To() does not validate for the msg.sender to be an EOA only.
As such, it is possible that the deposit of the ERC721 is made by a Smart contract. While ,locking into the bridge,
ERC721::transferFrom() is called which bypasses the check for the address receiving the token.

hence on bridging, the token will be successfully deposited into the bridge.

Later when finalizeBridgeERC721() is initiated, the deposited NFT should be returned back to the depositor account. To transfer the deposited token back to the owner, the below function is called.

   // When a withdrawal is finalized on L1, the L1 Bridge transfers the NFT to the
   // withdrawer.
   IERC721(_localToken).safeTransferFrom(address(this), _to, _tokenId);

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
