### DepositHelper contract does not implement EIP-712 rules for deposit/redeem with signature ###

### Description ###
Signatures should be protected against replay attack with a nonce and block.chainid and domain to ensure the same signature cannot be replayed across domains or chains.

But the below functions ETHDepositHelper does not take into account the domain or chainid.

```
function deposit(uint256 amount, address vault, uint256 deadline, uint8 v, bytes32 r, bytes32 s) external {
        SafeERC20.safePermit(_eETH, _msgSender(), address(this), amount, deadline, v, r, s);
        _deposit(amount, vault);
    }


function redeem(uint256 amount, address vault, uint256 deadline, uint8 v, bytes32 r, bytes32 s) external {
        SafeERC20.safePermit(IERC20Permit(vault), _msgSender(), address(this), amount, deadline, v, r, s);
        _redeem(amount, vault);
    }

```

### Recommendation ###
EIP-712 should be followed to ensure the signatures are not replayed.