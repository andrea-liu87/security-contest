## Root + Impact
### Description
The ReFiSwapRebateHook::TokenWithdraw event is emitted with incorrectly ordered parameters, causing a mismatch between the event declaration and the actual emitted values.
```Solidity
// event declaration
event TokensWithdrawn(address indexed token, address indexed to, uint256 amount);
​
//withdraw function
function withdrawTokens(address token, address to, uint256 amount) external onlyOwner {
IERC20(token).transfer(to, amount);
emit TokensWithdrawn(to, token , amount); //@>Event parameter is different with Even declaration
}
```
​
The emitted event reverses the token and to parameters, resulting in inaccurate event logs.

## Risk
### Likelihood:
The incorrect parameter ordering is deterministic and will occur every time withdrawTokens is invoked.

### Impact:
Off-chain consumers (indexers, analytics systems, monitoring tools, or integrators) relying on event logs will interpret the event data incorrectly.
This can lead to:

Misidentification of the token being withdrawn.

Incorrect attribution of withdrawal recipients.

Potential downstream logic errors in systems that consume these events to trigger automated processes.

In on-chain contexts, event logs may also be used by other contracts during optimistic or proof-based systems. Incorrect logs reduce auditability and can complicate debugging, forensics, or state reconstruction.

### Proof of Concept
Using the existing test suite, the failing behavior can be reproduced by including the following test:

Run with forge test --match-test test_WithdrawTokens_Event -vvvv

​```Solidity
function test_WithdrawTokens_Event() public {
        // First, send some tokens to the hook contract
        uint256 mintAmount = 1 ether;
        reFiToken.mint(address(rebateHook), mintAmount);
        
        uint256 initialBalance = reFiToken.balanceOf(address(this));
        uint256 hookBalanceBefore = reFiToken.balanceOf(address(rebateHook));
        
        // Withdraw 
        uint256 withdrawAmount = 0.5 ether;
        vm.expectEmit();
        emit TokensWithdrawn(address(reFiToken), address(this), withdrawAmount);
        rebateHook.withdrawTokens(address(reFiToken), address(this), withdrawAmount);
    }
```

The test output confirms the mismatch:
```
FAIL: TokensWithdrawn param mismatch at token: expected=0x212224D2F2d262cd093eE13240ca4873fcCBbA3C, got=0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496, 
to: expected=0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496, got=0x212224D2F2d262cd093eE13240ca4873fcCBbA3C] test_WithdrawTokens_Event() (gas: ```


### Recommended Mitigation
```diff
 function withdrawTokens(address token, address to, uint256 amount) external onlyOwner {
        IERC20(token).transfer(to, amount);
-       emit TokensWithdrawn(to, token , amount);
+       emit TokensWithdrawn(token, to, amount);
    }
```