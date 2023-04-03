### Low 1
https://github.com/code-423n4/2023-03-mute/blob/main/contracts/amplifier/MuteAmplifier.sol#L180
RescueTokens doesn't have checks for fee0 & fee1 tokens. Admin might accidentally withdraw fee tokens that are supposed to be for the stakers:
https://github.com/code-423n4/2023-03-mute/blob/main/contracts/amplifier/MuteAmplifier.sol#L259-L260

### Low 2
https://github.com/code-423n4/2023-03-mute/blob/main/contracts/amplifier/MuteAmplifier.sol#L186
It's entirely possible for totalStakers = 0 during the staking period. If rescueToken is called incorrectly here, then there might not be enough reward balance to distribute if users start staking afterwards. This could mean that withdraw fails:
https://github.com/code-423n4/2023-03-mute/blob/main/contracts/amplifier/MuteAmplifier.sol#L247