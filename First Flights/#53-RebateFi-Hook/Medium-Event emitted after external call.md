## Root + Impact
### Description
The function makes an external call to an ERC20 token before emitting the event, if transfer is failed event may still be emitted.
```Solidity
    /// @notice Withdraws tokens from the hook contract
    /// @param token Address of the token to withdraw
    /// @param to Address to send the tokens to
    /// @param amount Amount of tokens to withdraw
    /// @dev Only callable by owner
    function withdrawTokens(address token, address to, uint256 amount) external onlyOwner {
        IERC20(token).transfer(to, amount); // @> External call before state changes/events - potential reentrancy vector
        emit TokensWithdrawn(to, token , amount); // @> Event emitted after external call
    }
```

## Risk
### Likelihood:
The function only emits an event after the external call, so direct state corruption is limited

### Impact:
Event still be emitted regarding transfer status failed or success may give false information to other function.

### Proof of Concept
```Solidity
// create mock token to represent failed transfer
contract MockFailingToken is ERC20, Ownable {
    constructor() ERC20("RebateFi", "ReFi") Ownable(msg.sender) {
        _mint(msg.sender, 1_000_000 * 10 ** decimals());
    }
​
    function mint(address to, uint256 amount) external onlyOwner {
        _mint(to, amount);
    }
    function transfer(address to, uint256 amount) public override returns (bool) {
        if(amount == 0.5 ether){
            return false; 
        }
        return true;
    }
}
```
​
Using existing test suite, replace reFiToken with MockFailingToken and add this function
```Solidity
    event TokensWithdrawn(address indexed token, address indexed to, uint256 amount);
    function test_WithdrawTokens_Failed() public {
        // First, send some tokens to the hook contract
        uint256 mintAmount = 1 ether;
        reFiToken.mint(address(rebateHook), mintAmount);
        
        uint256 initialBalance = reFiToken.balanceOf(address(this));
        console.log("initial balance ", initialBalance);
        uint256 hookBalanceBefore = reFiToken.balanceOf(address(rebateHook));
        
        // Withdraw but the transfer is failed
        uint256 withdrawAmount = 0.5 ether;
        vm.expectEmit();
        emit TokensWithdrawn(address(this), address(reFiToken), withdrawAmount);
        rebateHook.withdrawTokens(address(reFiToken), address(this), withdrawAmount);
        
        uint256 finalBalance = reFiToken.balanceOf(address(this));
        console.log("final balance ", finalBalance);
        uint256 hookBalanceAfter = reFiToken.balanceOf(address(rebateHook));
        
   
    }
```

The test will passed but for logs output it will show that the transfer is failed
```
Logs:
initial balance 1000999900000000000000000
final balance 1000999900000000000000000
```

### Recommended Mitigation
It is recommended to follow Checks Effects Interactions patterns CEI pattern to ensure state updates and events are emitted before external calls to ensure accurate information. Move the events above the transfers

It may be ideal to make use of Reentrancy Guards e.g OpenZeppelin nonreentrant modifiers on affected functions

It may be ideal to whitelist allowed tokens for token pool and not allow callback, hook, tokens such as ERC777, ERC1363,
```diff
function withdrawTokens(address token, address to, uint256 amount) 
    external 
    onlyOwner 
    nonReentrant 
{
+    // Emit event first to follow CEI pattern
+    emit TokensWithdrawn(to, token, amount);
+    // Then perform transfer with proper error handling
    bool success = IERC20(token).transfer(to, amount);
+    if (!success){revert();}
-    emit TokensWithdrawn(to, token, amount);
}```