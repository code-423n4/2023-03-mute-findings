# Mute Low Risk and Non-Critical Issues
## Summary
| Risk      | Title | File | Instances
| ----------- | ----------- | ----------- | ----------- |
| L-01      | Use fixed compiler version | - | 3 |
| L-02      | `RedeemEvent` emits wrong `to` address | dMute.sol | 1 |
| L-03      | Replicate price checks from constructor in setter functions | MuteBond.sol | 2 |
| L-04      | Ownable: Does not implement 2-Step-Process for transferring ownership | MuteAmplifier.sol | 1 |
| L-05      | Check that staking cannot occur when `endTime` is reached | MuteAmplifier.sol | 1 |
| N-01      | Remove `require` statements that are always true | dMute.sol | 2 |
| N-02      | Remove `SafeMath` library | - | 3 |
| N-03      | Event parameter names are messed up | MuteBond.sol | - |
| N-04      | Event is never emitted | - | 2 |
| N-05      | Move `payoutFor` calculation into `else` block | MuteBond.sol | 1 |
| N-06      | Remove redundant check in `stake` function | MuteAmplifier.sol | 1 |


## [L-01] Use fixed compiler version
All in scope contracts use `^0.8.0` as compiler version.  
They should use a fixed version, i.e. `0.8.12`, to make sure the contracts are always compiled with
the intended version.  

[https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/amplifier/MuteAmplifier.sol#L2](https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/amplifier/MuteAmplifier.sol#L2)  

[https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/bonds/MuteBond.sol#L2](https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/bonds/MuteBond.sol#L2)  

[https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/dao/dMute.sol#L2](https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/dao/dMute.sol#L2)  

## [L-02] `RedeemEvent` emits wrong `to` address
In the `dMute.RedeemTo` function the `RedeemEvent` is emitted ([Link](https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/dao/dMute.sol#L128)).  

It is defined as:  
[Link](https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/dao/dMute.sol#L25)  
```solidity
event RedeemEvent(address to, uint256 unlockedAmount, uint256 burnAmount);
```

So the first parameter should be the `to` address.  

When the event is emitted the first parameter is `msg.sender`. The issue is that the `to` address and `msg.sender` can be different. So the event in some cases contains wrong information.  

Fix:  
```diff
diff --git a/contracts/dao/dMute.sol b/contracts/dao/dMute.sol
index 59f95b7..98a65d5 100644
--- a/contracts/dao/dMute.sol
+++ b/contracts/dao/dMute.sol
@@ -125,7 +125,7 @@ contract dMute is dSoulBound {
         _burn(msg.sender, total_to_burn);
 
 
-        emit RedeemEvent(msg.sender, total_to_redeem, total_to_burn);
+        emit RedeemEvent(to, total_to_redeem, total_to_burn);
     }
 
     function GetUserLockLength(address account) public view returns (uint256 amount) {
```

## [L-03] Replicate price checks from constructor in setter functions
In the `MuteBond` constructor it is checked that `maxPrice >= startPrice` ([Link](https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/bonds/MuteBond.sol#L82)).  

These checks are not implemented in the [`setStartPrice`](https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/bonds/MuteBond.sol#L109-L113) and [`setMaxPrice`](https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/bonds/MuteBond.sol#L99-L103) setter functions.  

It is recommended to add the check to both setter functions such that it is ensured the setter functions do not cause `startPrice` and `maxPrice` to be set to bad values.  

Fix:  
```diff
diff --git a/contracts/bonds/MuteBond.sol b/contracts/bonds/MuteBond.sol
index 96ee755..49b87f9 100644
--- a/contracts/bonds/MuteBond.sol
+++ b/contracts/bonds/MuteBond.sol
@@ -98,6 +98,7 @@ contract MuteBond {
      */
     function setMaxPrice(uint _price) external {
         require(msg.sender == customTreasury.owner());
+        require(_price >= startPrice, "starting price < min");
         maxPrice = _price;
         emit MaxPriceChanged(_price);
     }
@@ -108,6 +109,7 @@ contract MuteBond {
      */
     function setStartPrice(uint _price) external {
         require(msg.sender == customTreasury.owner());
+        require(maxPrice >= _price, "starting price < min");
         startPrice = _price;
         emit StartPriceChanged(_price);
     }
```

## [L-04] Ownable: Does not implement 2-Step-Process for transferring ownership
`MuteAmplifier` inherits from the `Ownable` contract.  
This contract does not implement a `2-Step-Process` for transferring ownership.  
So ownership of the contract can easily be lost when making a mistake when transferring ownership.  

Consider using the `Ownable2Step` contract from OZ ([https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/access/Ownable2Step.sol](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/access/Ownable2Step.sol)) instead.  

## [L-05] Check that staking cannot occur when `endTime` is reached
The [`MuteAmplifier.stake`](https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/amplifier/MuteAmplifier.sol#L203-L223) function should require that the current timestamp is smaller than `endTime` even when the call to `stake` is the first that ever occurred.  
Currently the check is only made in the case that the call to `stake` is not the first.  
The check should be made in both cases.  
This is because when staking occurs when `block.timestamp >= endTime`, no rewards will be paid out. Additionally the user needs to pay the management fee on his `LP token` stake. So there is really no point in allowing users to do it because it only hurts them.  

Fix:  
```diff
diff --git a/contracts/amplifier/MuteAmplifier.sol b/contracts/amplifier/MuteAmplifier.sol
index 9c6fcb5..460c408 100644
--- a/contracts/amplifier/MuteAmplifier.sol
+++ b/contracts/amplifier/MuteAmplifier.sol
@@ -202,13 +202,12 @@ contract MuteAmplifier is Ownable{
      */
     function stake(uint256 lpTokenIn) external virtual update nonReentrant {
         require(lpTokenIn > 0, "MuteAmplifier::stake: missing stake");
+        require(block.timestamp < endTime, "MuteAmplifier::stake: staking is over");
         require(block.timestamp >= startTime && startTime !=0, "MuteAmplifier::stake: not live yet");
         require(IERC20(muteToken).balanceOf(address(this)) > 0, "MuteAmplifier::stake: no reward balance");
 
         if (firstStakeTime == 0) {
             firstStakeTime = block.timestamp;
-        } else {
-            require(block.timestamp < endTime, "MuteAmplifier::stake: staking is over");
         }
 
         lpToken.safeTransferFrom(msg.sender, address(this), lpTokenIn);
```

## [N-01] Remove `require` statements that are always true
The following two `require` statements always pass:  

[https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/dao/dMute.sol#L99-L100](https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/dao/dMute.sol#L99-L100)  

`lock_info.amount` and `lock_info.tokens_minted` are of type `uint256` so they cannot be `< 0`.  
In fact the check that `tokens_to_mint > 0` in the `LockTo` function even ensures that `lock_info.amount` and `lock_info.tokens_minted` are greater `0`.  

## [N-02] Remove `SafeMath` library
All 3 in-scope contracts use the `SafMath` library for simple addition, subtraction, multiplication and division.  

This causes an unnecessary overhead since beginning from Solidity version `0.8.0` arithemtic operations revert by default on overflow / underflow.  

By looking into the implementation of the `SafeMath` library you can also see that it is just a wrapper around the basic arithmatic operations and does not add any checks (at least for the functions used in the in-scope contracts) ([Link](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/math/SafeMath.sol)).  

## [N-03] Event parameter names are messed up
In the `MuteBond` contract, the following events are defined:  

[Link](https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/bonds/MuteBond.sol#L12-L18)  
```solidity
event BondCreated(uint deposit, uint payout, address depositor, uint time);
event BondPriceChanged(uint internalPrice, uint debtRatio);
event MaxPriceChanged(uint _price);
event MaxPayoutChanged(uint _price);
event EpochDurationChanged(uint _payout);
event BondLockTimeChanged(uint _duration);
event StartPriceChanged(uint _lock);
```

Some of the parameter names do not accurately describe the variable that is actually emitted. E.g. for the `EpochDurationChanged` event, it is the new epoch duration that is emitted and not a `_payout` variable.  

There is also an inconsistency with the `MaxPayoutChanged` and `StartPriceChanged` events. So I recommend to use more descriptive names in the event emissions for these 3 events.  


## [N-04] Event is never emitted
There are two events defined that are never emitted. They can be removed to make the code cleaner.  

[https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/bonds/MuteBond.sol#L13](https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/bonds/MuteBond.sol#L13)  

[https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/amplifier/MuteAmplifier.sol#L22](https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/amplifier/MuteAmplifier.sol#L22)  


## [N-05] Move `payoutFor` calculation into `else` block
[https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/bonds/MuteBond.sol#L155](https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/bonds/MuteBond.sol#L155)  

In case the `if` block is entered, `payout` is calculated twice.  
Therefore the `payoutFor` call can be moved into the `else` block to clean up the code and save a little bit of Gas.  

Fix:
```diff
diff --git a/contracts/bonds/MuteBond.sol b/contracts/bonds/MuteBond.sol
index 96ee755..b462992 100644
--- a/contracts/bonds/MuteBond.sol
+++ b/contracts/bonds/MuteBond.sol
@@ -152,11 +152,12 @@ contract MuteBond {
      */
     function deposit(uint value, address _depositor, bool max_buy) external returns (uint) {
         // amount of mute tokens
-        uint payout = payoutFor( value );
+        uint payout;
         if(max_buy == true){
           value = maxPurchaseAmount();
           payout = maxDeposit();
         } else {
+          payout = payoutFor( value );
           // safety checks for custom purchase
           require( payout >= ((10**18) / 100), "Bond too small" ); // must be > 0.01 payout token ( underflow protection )
           require( payout <= maxPayout, "Bond too large"); // size protection because there is no slippage
```

## [N-06] Remove redundant check in `stake` function
The following check is redundant:  

[Link](https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/amplifier/MuteAmplifier.sol#L206)  
```solidity
require(IERC20(muteToken).balanceOf(address(this)) > 0, "MuteAmplifier::stake: no reward balance");
```

This check is redundant because before this check there is [another check](https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/amplifier/MuteAmplifier.sol#L205) that `startTime!=0` which means the [`initializeDeposit`](https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/amplifier/MuteAmplifier.sol#L160-L172) function has been called which ensures the `MUTE` balance is not zero.  

There are edge cases where the current check would apply, e.g. when staking occurs after the `endTime`. But the current check is not sufficient in this case because there could just be a little excess `MUTE` balance in the contract and the user would still not get rewards. So I recommend to remove the existing check and the edge cases will be addressed by the other changes I propose in this report.  

