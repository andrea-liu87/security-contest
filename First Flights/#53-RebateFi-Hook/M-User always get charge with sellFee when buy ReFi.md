User will always get charged by SellFee when buy ReFi

### Description
When perform swap from ETH to ReFi, user will always get charged by SellFee
```Solidity
function _beforeSwap(
        address sender,
        PoolKey calldata key,
        SwapParams calldata params,
        bytes calldata
    ) internal override returns (bytes4, BeforeSwapDelta, uint24) {
        
        bool isReFiBuy = _isReFiBuy(key, params.zeroForOne);
        
        uint256 swapAmount = params.amountSpecified < 0 
                ? uint256(-params.amountSpecified) 
                : uint256(params.amountSpecified);
​
        uint24 fee;
​
  //The input isReFiBuy will always be false
   @>   if (isReFiBuy) {
        fee = buyFee;    
            emit ReFiBought(sender, swapAmount);
        } else {
            fee = sellFee;
            uint256 feeAmount = (swapAmount * sellFee) / 100000;
            emit ReFiSold(sender, swapAmount, feeAmount);
        }
    
        return (
            BaseHook.beforeSwap.selector,
            BeforeSwapDeltaLibrary.ZERO_DELTA,
            fee | LPFeeLibrary.OVERRIDE_FEE_FLAG
        );
    }
​
​
function _isReFiBuy(PoolKey calldata key, bool zeroForOne) internal view returns (bool) {
        //IsReFiCurrency0 will always return 0 because currency0 will not always ReFi
        bool IsReFiCurrency0 = Currency.unwrap(key.currency0) == ReFi;
        
        if (IsReFiCurrency0) {
            return zeroForOne;
        } else {
            return !zeroForOne;
        }
    }
```

## Risk
### Likelihood:
The event will always occured all the time

### Impact:
User will always get charged by SellFee when buy ReFi which is sometimes will be higher than BuyFee.

### Proof of Concept
See this function into RebateFiHook.t.sol

Run the function with forge test --match-test test_BuyReFi_wrongFee

From the logs we can see the transaction fee is refer to BuyFee == 300
```
Logs:
  user ETH intial balance  1000000000000000000
  user ReFi initial balance  1000000000000000000000
  final ETH  balance:  990000000000000000
  Final ReFi balance:  1000009967023479114567
  user1 get ReFi  9967023479114567
  transaction fee  32976520885433
```

```Solidity
function test_BuyReFi_wrongFee() public {
    // Verify fee configuration is zero for buys
    (uint24 buyFee, ) = rebateHook.getFeeConfig();
    assertEq(buyFee, 0, "Buy fee should 0");
​
    uint256 ethAmount = 0.01 ether;
    // Fund user and record initial balances
    vm.deal(user1, 1 ether);
  
    // Record initial balances after funding
    uint256 initialEthBalance = user1.balance;
    console.log("user ETH intial balance ", initialEthBalance);
    uint256 initialReFiBalance = reFiToken.balanceOf(user1);
    console.log("user ReFi initial balance ", initialReFiBalance);
    
    vm.startPrank(user1);
​
    SwapParams memory params = SwapParams({
        zeroForOne: true, // ETH -> ReFi
        amountSpecified: -int256(ethAmount),
        sqrtPriceLimitX96: TickMath.MIN_SQRT_PRICE + 1
    });
​
    PoolSwapTest.TestSettings memory testSettings = PoolSwapTest.TestSettings({
        takeClaims: false,
        settleUsingBurn: false
    });
​
    // Perform swap
    swapRouter.swap{value: ethAmount}(key, params, testSettings, ZERO_BYTES);
    vm.stopPrank();
​
    // Check balances after swap
    uint256 finalEthBalance = user1.balance;
    uint256 finalReFiBalance = reFiToken.balanceOf(user1);
​
    // Verify fee configuration remains zero for buys
    (uint24 currentBuyFee, ) = rebateHook.getFeeConfig();
    assertEq(currentBuyFee, 0, "Buy fee should remain 0");
​
    console.log("final ETH  balance: ", finalEthBalance);
    console.log("Final ReFi balance: ", finalReFiBalance);
    console.log("user1 get ReFi ", finalReFiBalance - initialReFiBalance);
   // transaction fee : (1e16*0.3%) = 3e13
    console.log("transaction fee ", ethAmount - (finalReFiBalance - initialReFiBalance));
    // if the fee is 0.0% so we should receive approximately 0.01 of ReFi
    assertApproxEqAbs(finalReFiBalance - initialReFiBalance, ethAmount, 0.00001 ether);
    // if the fee is 0.3%(sellFee) so we should receive approximately 0.00997 of ReFi
   //assertApproxEqAbs(finalReFiBalance - initialReFiBalance, uint256(.997e16), 0.00001 ether);
   ```

### Recommended Mitigation
```diff
function _isReFiBuy(PoolKey calldata key, bool zeroForOne) internal view returns (bool) {      
-        bool IsReFiCurrency0 = Currency.unwrap(key.currency0) == ReFi;
        
-        if (IsReFiCurrency0) {
            return zeroForOne;
-        } else {
-            return !zeroForOne;
-        }
    }
```