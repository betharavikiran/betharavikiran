**Anyone can increase their shareholding of Liquid-staking tokens leading to Insolvency**

**Description**: 
The base class of Liquid-staking token contract, XXXXERC20RebaseUpgradeable keeps track of the total shares and shares at each account level. Every time new shares are minted via _mintShares() function, the appropriate account is credited with the new shares and the cumulative balance for total shares held by the contract is also update.

Shares impacts the proportional distribution of yield to the share holders, and hence correct tracking is critical.

**Vulnerability**:
The vulnerability arises from the _transferSharesX() function of the XXXXERC20RebaseUpgradeable contract where the owner of the shares can transfer his shares to another account. Any one can abuse this _transferSharesX() function by calling the function with _recipient as msg.sender himself.

```
     function _transferSharesX(
        address _recipient,
        uint256 _shares
    ) external returns (uint256) {
```

When the recipient passed is msg.sender, the _transferSharesX() function will have sender and recipient with the same address.

Lets say,
_sender = 0xAAAA;
_recipient = 0xAAAA;

Lets also assume 0xAAAA has 1000 shares in his account. Lets also assume that total Shares is 10000.

Now, if the 0xAAAA initiates a transfer for 1000 shares to 0xAAAA account, due to the _transferShares() function implementation,
the shares for 0xAAAA will be doubled to 2000 shares.

```
   function _transferSharesX(
        address _sender,
        address _recipient,
        uint256 _shares
    ) internal {
        if (_sender == address(0) || _recipient == address(0))
            revert Errors.ZeroAddress();
        if (_recipient == address(this)) revert Errors.NotAllowed();

        XXXXERC20RebaseStorage storage $ = _getXXXXERC20RebaseStorage();

        uint256 currentSenderShares = $.shares[_sender];
        uint256 currentRecipientShares = $.shares[_recipient];

        if (_shares > currentSenderShares) revert Errors.InvalidAmount();

        $.shares[_sender] = currentSenderShares - _shares; // audit
        $.shares[_recipient] = currentRecipientShares + _shares; // audit
    }
```    
In the above implementation, the current holding for both the senders and recipients are cached in the memory variables and hence
results in double account.

 uint256 currentSenderShares = $.shares["0xAAAA"];
 uint256 currentRecipientShares = $.shares["0xAAAA"];
Note how, the currentSenderShares is 1000 and currentRecipientShares will also be 1000.

 $.shares[0xAAAA] = currentSenderShares - _shares; // this will result in 0xAAAA having a balance of 0
 $.shares[_recipient] = currentRecipientShares + _shares; // But, this will result in 0xAAAA having a balance of 1000 + 1000 = 2000.
So, in the above line, the shares held by 0xAAAA after execution of transferShares() would be 2000.
0xAAAA can repeat this call any number of times and boost his share holdings. Lets say, he does this 4 times.

After 1st iteration, 2000
After 2nd iteration, 4000
After 3rd iteration, 8000
After 4th iteration, 16000

After these 4 iteration, the total shares held by 0xAAAA would be 16000, when the total minted was only 10000 as stated above.
So, when 0xAAAA would attempt to redeem, there will not be enough assets to honor the redemption process.

An evil actor can steal all the staked assets leaving the other users to fail when redeeming.

**Recommendation(s)**:
Two steps:
a) add a validation in the _transferShares() to not allow sender and recipient to be same address.
b) Change the map updating logic to below

$.shares[0xAAAA] =- _shares; 
$.shares[0xAAAA] =+ _shares;  
