## Root + Impact
### Description
The withdrawTokens function allows the contract owner to withdraw ERC20 tokens from the contract. Normally, ERC20's transfer function returns a boolean indicating whether the transfer was successful or not.

The specific issue is that the return value from IERC20(token).transfer(to, amount) is completely ignored, which means failed transfers will go undetected. Some ERC20 tokens (like USDT) do not revert on failure but instead return false, and this function would not catch such failures.
```Solidity
/// @notice Withdraws tokens from the hook contract
    /// @param token Address of the token to withdraw
    /// @param to Address to send the tokens to
    /// @param amount Amount of tokens to withdraw
    /// @dev Only callable by owner
    function withdrawTokens(address token, address to, uint256 amount) external onlyOwner {
        IERC20(token).transfer(to, amount); // @> Return value ignored - no validation of transfer success
        emit TokensWithdrawn(to, token , amount);
    }
```

## Risk
### Likelihood:
The contract interacts with various ERC20 tokens, some of which may return false instead of reverting on transfer failure

Transfer failures can occur due to insufficient balances, transfer pauses, blacklisted addresses, or token-specific restrictions

### Impact:

Failed transfers result in lost tokens that remain stuck in the contract despite the emission of a successful withdrawal event

Users receiving incorrect state information through events, leading to confusion and potential accounting errors

Contract state becomes inconsistent with actual token balances

### Proof of Concept
```Solidity
import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";
​
// Create mock ERC20 token that returns false on transfer failure
contract MockFailingToken is ERC20, Ownable {
    constructor() ERC20("RebateFi", "ReFi") Ownable(msg.sender) {
        _mint(msg.sender, 1_000_000 * 10 ** decimals());
    }
​
    function mint(address to, uint256 amount) external onlyOwner {
        _mint(to, amount);
    }
    function transfer(address to, uint256 amount) public override returns (bool) {
        return false; // Simulates a token that returns false on failure
    }
}```

On the current test suite, change mock ReFiToken into the MockFailingToken

```Solidity
contract TestReFiSwapRebateHook is Test, Deployers, ERC1155TokenReceiver {
​
MockERC20 token;
-MockERC20 reFiToken;
+MockFailingToken reFiToken;
  ReFiSwapRebateHook public rebateHook;
​
  Currency ethCurrency = Currency.wrap(address(0));
  Currency tokenCurrency;
  Currency reFiCurrency;
​
  address user1 = address(0x1);
  address user2 = address(0x2);
​
  uint160 constant SQRT_PRICE_1_1_s = 79228162514264337593543950336;
function setUp() public {
  // Deploy the Uniswap V4 PoolManager
  deployFreshManagerAndRouters();
​
  // Deploy the ERC20 token
  token = new MockERC20("TOKEN", "TKN", 18);
  tokenCurrency = Currency.wrap(address(token));
​
  // Deploy the ReFi token
- reFiToken = new MockERC20();
+reFiToken = new MockFailingToken();
  reFiCurrency = Currency.wrap(address(reFiToken));
​
​
// Mint tokens to test contract and users
token.mint(address(this), 1000 ether);
token.mint(user1, 1000 ether);
token.mint(user2, 1000 ether);
​
reFiToken.mint(address(this), 1000 ether);
reFiToken.mint(user1, 1000 ether);
reFiToken.mint(user2, 1000 ether);
​
​
// Get creation code for hook
bytes memory creationCode = type(ReFiSwapRebateHook).creationCode;
bytes memory constructorArgs = abi.encode(manager, address(reFiToken));
​
// Find a salt that produces a valid hook address
uint160 flags = uint160(
    Hooks.BEFORE_INITIALIZE_FLAG | 
    Hooks.AFTER_INITIALIZE_FLAG | 
    Hooks.BEFORE_SWAP_FLAG
);
​
(address hookAddress, bytes32 salt) = HookMiner.find(
    address(this),
    flags,
    creationCode,
    constructorArgs
);
​
// Deploy the hook with the mined salt
rebateHook = new ReFiSwapRebateHook{salt: salt}(manager, address(reFiToken));
require(address(rebateHook) == hookAddress, "Hook address mismatch");
​
// Approve tokens for the test contract
token.approve(address(swapRouter), type(uint256).max);
token.approve(address(modifyLiquidityRouter), type(uint256).max);
reFiToken.approve(address(rebateHook), type(uint256).max);
reFiToken.approve(address(swapRouter), type(uint256).max);
reFiToken.approve(address(modifyLiquidityRouter), type(uint256).max);
​
// Initialize the pool with ReFi token using DYNAMIC_FEE_FLAG
(key, ) = initPool(
    ethCurrency,
    reFiCurrency,
    rebateHook,
    LPFeeLibrary.DYNAMIC_FEE_FLAG,
    SQRT_PRICE_1_1_s
);
​
// Add liquidity
uint160 sqrtPriceAtTickUpper = TickMath.getSqrtPriceAtTick(60);
​
uint256 ethToAdd = 0.1 ether;
uint128 liquidityDelta = LiquidityAmounts.getLiquidityForAmount0(
    SQRT_PRICE_1_1,
    sqrtPriceAtTickUpper,
    ethToAdd
);
​
modifyLiquidityRouter.modifyLiquidity{value: ethToAdd}(
    key,
    ModifyLiquidityParams({
        tickLower: -60,
        tickUpper: 60,
        liquidityDelta: int256(uint256(liquidityDelta)),
        salt: bytes32(0)
    }),
    ZERO_BYTES
);
}
​
function test_WithdrawTokens_Success() public {
  // First, send some tokens to the hook contract
  uint256 transferAmount = 1 ether;
  reFiToken.transfer(address(rebateHook), transferAmount);
​
  uint256 initialBalance = reFiToken.balanceOf(address(this));
  uint256 hookBalanceBefore = reFiToken.balanceOf(address(rebateHook));
​
  // Withdraw a reasonable amount
  uint256 withdrawAmount = 0.5 ether;
  rebateHook.withdrawTokens(address(reFiToken), address(this), withdrawAmount);
​
  uint256 finalBalance = reFiToken.balanceOf(address(this));
  uint256 hookBalanceAfter = reFiToken.balanceOf(address(rebateHook));
​
  // finalBalance should equal initialBalance + withdrawAmount (we transferred away `transferAmount` earlier)
  assertEq(finalBalance, initialBalance + withdrawAmount, "Should receive withdrawn tokens");
  assertEq(hookBalanceAfter, hookBalanceBefore - withdrawAmount, "Hook balance should decrease");
}
```
​
### Recommended Mitigation
```diff
function withdrawTokens(address token, address to, uint256 amount) external onlyOwner {
-    IERC20(token).transfer(to, amount);
+    bool success = IERC20(token).transfer(to, amount);
+    require(success, "Token transfer failed");
    emit TokensWithdrawn(to, token , amount);
}
```

Alternatively, use OpenZeppelin's SafeERC20 for more robust handling:
```diff
+import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
​
+using SafeERC20 for IERC20;
​
function withdrawTokens(address token, address to, uint256 amount) external onlyOwner {
-    IERC20(token).transfer(to, amount);
+    IERC20(token).safeTransfer(to, amount);
    emit TokensWithdrawn(to, token , amount);
}```