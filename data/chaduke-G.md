G1. Define ``week_time`` and ``max_lock`` as public private constants to save gas. 

[https://github.com/code-423n4/2023-03-mute/blob/64521eff60cebca4d7c43f670bdc1414f8704fd0/contracts/dao/dMute.sol#L50-L51](https://github.com/code-423n4/2023-03-mute/blob/64521eff60cebca4d7c43f670bdc1414f8704fd0/contracts/dao/dMute.sol#L50-L51)

G2. I don't see how multiplication by 1e18 and then division by 1e18 will help minimizing precision loss here. So maybe we can save gas instead:

[https://github.com/code-423n4/2023-03-mute/blob/64521eff60cebca4d7c43f670bdc1414f8704fd0/contracts/dao/dMute.sol#L57](https://github.com/code-423n4/2023-03-mute/blob/64521eff60cebca4d7c43f670bdc1414f8704fd0/contracts/dao/dMute.sol#L57)

```diff
-  uint256 base_tokens = _amount.mul(_lock_time.mul(10**18).div(max_lock)).div(10**18);
+  uint256 base_tokens = (_amount.mul(_lock_time)).div(max_lock);
```

G3. No need to check these conditions, they will always be true. They can be eliminated to save gas. 

[https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/dao/dMute.sol#L99](https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/dao/dMute.sol#L99)
 
G4. Here we only need to reset the time

[https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/dao/dMute.sol#L105](https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/dao/dMute.sol#L105)

```diff
- _userLocks[msg.sender][index] = UserLockInfo(0,0,0);
+ _userLocks[msg.sender][index].time = 0;
```
G5. Drop the first condition since it will be always true.
[https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/amplifier/MuteAmplifier.sol#L139](https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/amplifier/MuteAmplifier.sol#L139)

```diff
- require(_mgmt_fee >= 0 && _mgmt_fee <= 1000, "MuteAmplifier: invalid _mgmt_fee");
+ require(_mgmt_fee <= 1000, "MuteAmplifier: invalid _mgmt_fee");
```

G6. This line can be eliminated since the check will always pass due to L205 and the checks in ``initializeDeposit()`` (L166). 

[https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/amplifier/MuteAmplifier.sol#L206](https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/amplifier/MuteAmplifier.sol#L206)

G7. There is no need for the if-condition since it is always be true due to the check at L231 inside the function ``_applyReward()``. That check will ensure ``lpTokenOut > 0``.

[https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/amplifier/MuteAmplifier.sol#L233-L237](https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/amplifier/MuteAmplifier.sol#L233-L237)

G8. This safety check can be eliminated since we can infer ``epochStart <= block.timestamp`` will always be true. This is because each time we will only add 5% of the ``timeElapsed`` to ``epochStart``. as a result, ``epochStart`` will be approaching near ``block.timestamp`` but will never exceed it.

[https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/bonds/MuteBond.sol#L189-L190](https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/bonds/MuteBond.sol#L189-L190)

Consider how ``epochStart`` was calculated: The previous two lines show how to calculate ``epochStart``: 

G9. ``fee0 <= totalFees0 && fee1 <= totalFees1`` are always going to be true, so there is no need to check them.

[https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/amplifier/MuteAmplifier.sol#L334-L342](https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/amplifier/MuteAmplifier.sol#L334-L342)

We can see that these two checks are always be true because totalFees0 and totalFees1 are changed ONLY at lines L116 and L117 of the update() function, and from the way that fee0 and fee1 are obtained in function ``_applyReward()``, ``fee0`` and ``fee1`` will always be a partial amount of ``totalFees0`` and ``totalFees1``. Therefore, the checks will always be true. There is no need to check. 

In addition, checking them against balance of the two fees are not necessary either since the safeTransfer() will do that automatically.

```diff
- if ((fee0 > 0 || fee1 > 0) && fee0 <= totalFees0 && fee1 <= totalFees1) {
+        if(fee0 > 0){
            address(IMuteSwitchPairDynamic(lpToken).token0()).safeTransfer(msg.sender, fee0);
+           totalClaimedFees0 = totalClaimedFees0.add(fee0);
+        }  
+        if(fee1 > 0) {
            address(IMuteSwitchPairDynamic(lpToken).token1()).safeTransfer(msg.sender, fee1);

-            totalClaimedFees0 = totalClaimedFees0.add(fee0);
            totalClaimedFees1 = totalClaimedFees1.add(fee1);
+        }
            emit FeePayout(msg.sender, fee0, fee1);
        }
``` 

G10. This line can be eliminated since the next line will check this condition anyway (``transferFrom()``).

[https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/dao/dMute.sol#L69](https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/dao/dMute.sol#L69)

G11. One can retrieve the value from ``info.totalLP`` to save gas:

[https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/amplifier/MuteAmplifier.sol#L425](https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/amplifier/MuteAmplifier.sol#L425)

```diff
- uint256 totalCurrentStake = totalStake();
+  uint256 totalCurrentStake = info.totalLP;
```
