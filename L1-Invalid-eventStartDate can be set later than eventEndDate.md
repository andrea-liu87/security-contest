## Root + Impact
### Description
By mistakes owner can set the eventEndDate earlier than eventStartDate

```Solidity
 constructor (IERC20 _asset, uint256 _participationFeeBsp, uint256 _eventStartDate, address _participationFeeAddress, uint256 _minimumAmount, uint256 _eventEndDate) ERC4626 (_asset) ERC20("BriTechLabs", "BTT") Ownable(msg.sender) {
         if (_participationFeeBsp > PARTICIPATIONFEEBSPMAX){
            revert limiteExceede();
         }
         
         participationFeeBsp = _participationFeeBsp;
        //root cause in bellow 2 lines code
@>    eventStartDate = _eventStartDate;
@>    eventEndDate = _eventEndDate;
         participationFeeAddress = _participationFeeAddress;
         minimumAmount = _minimumAmount;
         _setWinner = false;
    }
```
​
## Risk
### Likelihood:

It will occur if Owner give input in the BriVault constructor by mistakenly

### Impact:

- Owner cant setWinner()
- user cant withdraw()

### Proof of Concept
To see the impact of the error, in the briVault.t.sol inside setup function, we can set the variable as bellow :
```
        eventStartDate = block.timestamp + 4 days;
        eventEndDate = block.timestamp + 2 days;
```

### Recommended Mitigation
```diff
 constructor (IERC20 _asset, uint256 _participationFeeBsp, uint256 _eventStartDate, address _participationFeeAddress, uint256 _minimumAmount, uint256 _eventEndDate) ERC4626 (_asset) ERC20("BriTechLabs", "BTT") Ownable(msg.sender) {
         if (_participationFeeBsp > PARTICIPATIONFEEBSPMAX){
            revert limiteExceede();
         }
         
         participationFeeBsp = _participationFeeBsp;
         eventStartDate = _eventStartDate;
         eventEndDate = _eventEndDate;
+       if(eventEndDate <= eventStartDate){
+       revert endDateShouldBeLaterThanStartDate();
+        }
         participationFeeAddress = _participationFeeAddress;
         minimumAmount = _minimumAmount;
         _setWinner = false;
    }
```
​
