## Missing isActive Check in _withdraw() Function
- Severity: Low


### Targets
_withdraw(uint256 assets, uint256 shares, address receiver, address owner) in WERC7575Vault.sol. 


### Description
The internal _withdraw() function lacks validation of the vault's _isActive flag before processing withdrawal requests. While the function includes checks for zero addresses and zero amounts, it omits the crucial validation that the vault is in an active state. This inconsistency creates a security gap where withdrawals could bypass administrative pause controls.

The issue is particularly significant because the corresponding _deposit() function properly includes the if (!_isActive) revert VaultNotActive(); check, creating an asymmetric security model.  


### Root cause
The omission of `if (!_isActive) revert VaultNotActive();` at the beginning of the _withdraw() function. This creates an inconsistency with the _deposit() function which properly enforces the active state check.


### Proof of Concept
```solidity
// Current vulnerable implementation
function _withdraw(uint256 assets, uint256 shares, address receiver, address owner) internal {
    // Missing: if (!_isActive) revert VaultNotActive();
    // ... existing checks proceed ...
    _shareToken.burn(owner, shares);
    SafeTokenTransfers.safeTransfer(_asset, receiver, assets);  // Funds transferred even when paused
}
```
Attack Scenario:
- Admin pauses vault (_isActive = false) during emergency or maintenance
- Attackers notice the missing check in _withdraw()
- They call withdrawal functions (which internally call _withdraw())
- Funds are transferred despite the vault being paused
- Emergency pause mechanism is effectively bypassed

### Impact

1. Emergency Response Bypass: Administrators cannot effectively pause the vault during security incidents or upgrades
2. Funds at Risk: Assets can be withdrawn during paused states when the vault might be in an inconsistent state
3. Trust Erosion: Users expect consistent pause behavior across all vault operations
4. Regulatory Compliance Risk: If the vault operates in regulated environments, inconsistent pause behavior could violate compliance requirements
5. Asymmetric Protection: Deposits are protected but withdrawals are not, potentially enabling manipulation

### Mitigation
Add the missing _isActive check at the beginning of the _withdraw() function:

```solidity
function _withdraw(uint256 assets, uint256 shares, address receiver, address owner) internal {
    // Add this check (consistent with _deposit())
    if (!_isActive) revert VaultNotActive();  // FIX: Add missing check
    
    if (receiver == address(0)) {
        revert IERC20Errors.ERC20InvalidReceiver(address(0));
    }
    if (owner == address(0)) {
        revert IERC20Errors.ERC20InvalidSender(address(0));
    }
    if (assets == 0) revert ZeroAssets();
    if (shares == 0) revert ZeroShares();

    _shareToken.spendSelfAllowance(owner, shares);
    _shareToken.burn(owner, shares);
    SafeTokenTransfers.safeTransfer(_asset, receiver, assets);
    emit Withdraw(msg.sender, receiver, owner, assets, shares);
}
```