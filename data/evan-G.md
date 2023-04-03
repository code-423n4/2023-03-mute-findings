### Gas 1:
https://github.com/code-423n4/2023-03-mute/blob/main/contracts/amplifier/MuteAmplifier.sol#L103-L104
I believe this does the exact same thing as `value = (endTime.sub(_mostRecentValueCalcTime)).mul(perSecondReward);`

### Gas 2:
https://github.com/code-423n4/2023-03-mute/blob/main/contracts/amplifier/MuteAmplifier.sol#L98
perSecondReward doesn't change after firstStakeTime is set. So, it can be a global variable initialized here:
https://github.com/code-423n4/2023-03-mute/blob/main/contracts/amplifier/MuteAmplifier.sol#L90
