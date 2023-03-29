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
 
G4. Here we only need to rest the time

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

