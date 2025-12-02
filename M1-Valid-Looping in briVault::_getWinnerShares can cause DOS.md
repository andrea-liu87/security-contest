Loop in briVault::_getWinnerShares to write to storage variable briVault::totalWinnerShares can lead to expensive gas or exceed gas limit when calling the function

### Description
briVault::_getWinnerShares loops through the briVault::userAddress[] array to store the amount the user bet on the winning country. 
Store operations are expensive and if the briVault::userAddress[] become large, looping through it can exceed the gas limit.

```Solidity
// Root cause in the codebase with @> marks to highlight the relevant section
    function _getWinnerShares() internal returns (uint256) {
@>      for (uint256 i = 0; i < usersAddress.length; ++i) {
            address user = usersAddress[i];
@>          totalWinnerShares += userSharesToCountry[user][winnerCountryId];
        }
        return totalWinnerShares;
    }
```

## Risk
### Likelihood: Medium
Reason 1: This can happen when a lot of players participates to the game.
Reason 2: This can happen when an attacker join the game multiple times using the same stakedAsset (this is also a vulnerability) and gets pushed multiple times (also a vulnerability) into the briVault::userAddress. However, the attacker will lose some fees and gas fee during the attack with no explicit reward.

### Impact: High
Impact 1: Denial of service will make the owner unable to call briVault::selectWinner() to select winner and make the game unable to continue.

### Proof of Concept
This proof of concept is to be run inside the provided briVault.t.sol 
using forge test --mt testDOS -vvvv --gas-limit 45000000.The user1 deposited 5 tokens and joined the game 40000 times.Then when the owner tries to call briVault::setWinner(), the evm reverts with error: EvmError: ReentrancySentryOOG because it has exceeded the block gas limit of 45000000 on Ethereum.
```Solidity
//briVault.t.sol
    function testDOS() public {
        vm.pauseGasMetering();
        vm.startPrank(owner);
        briVault.setCountry(countries);
        vm.stopPrank();
        //USER 1 DEPOSITS AND JOIN EVENT
        vm.startPrank(user1);
        mockToken.approve(address(briVault), 5 ether);
        briVault.deposit(5 ether, user1);
        vm.stopPrank();
        for (uint256 i = 0; i < 40000; i++) {
            vm.prank(user1);
            briVault.joinEvent(10);
        }
        vm.warp(eventEndDate + 1);
        vm.roll(block.number + 1);
        vm.prank(owner);
      //resume gas metering to simulate
        vm.resumeGasMetering();
        briVault.setWinner(10);
    }
```

### Recommended Mitigation
Consider actively tracking the number of shares in each country during joinEvent and cancelParticipation using a mapping and check if the user is already in the briVault:userAddress with a mapping when joining to not push the same user several times.

