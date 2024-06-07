**assert is used instead of require for data validation in Zerolocker contract**


**Description**
assert() is used in Zerolocker contract across a number of places for data validation. This is an incorrect usage. For all data validations, require() should be used instead.

assert() is used to check for code that should never be false. Failing assertion probably means that there is a bug. It should be used only for invariant checking and not for data validation. assert() causes panic error.

require() reverts if the mentioned condition is not met and is designed to be used for errors in inputs or external components.

Gas treatment: The key difference between the two comes from how the gas is treated by these two reversion mechanisms.

assert() will use up all the remaining gas and revert the changes.
require() will also revert the changes, but refunds the gas back to the caller
This is a critical reason why all assets() used for data validation should be replaced with require()

Examples from Zerolocker contract.

Below are example of where assert should be replaced with require as they are more of input data validation.

```
     function _addTokenTo(address _to, uint256 _tokenId) internal {
        // Throws if `_tokenId` is owned by someone
        assert(idToOwner[_tokenId] == address(0));
  function _removeTokenFrom(address _from, uint256 _tokenId) internal {
        // Throws if `_from` is not the current owner
        assert(idToOwner[_tokenId] == _from);
     function _clearApproval(address _owner, uint256 _tokenId) internal { // @audit
        // Throws if `_owner` is not the current owner
        assert(idToOwner[_tokenId] == _owner);
  function setApprovalForAll(  //@audit done
        address _operator,
        bool _approved
    ) external override {
        // Throws if `_operator` is the `msg.sender`
        assert(_operator != msg.sender);
 function _mint(address _to, uint256 _tokenId) internal returns (bool) { //@audit done
        // Throws if `_to` is zero address
        assert(_to != address(0));
  address from = msg.sender;
          if (_value != 0 && depositType != DepositType.MERGE_TYPE) {
            assert(underlying.transferFrom(from, address(this), _value));
        }
   assert(underlying.transfer(msg.sender, value));
 ```  