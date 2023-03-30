QA1. payout() lacks the emit of event of ``emit Payout(msg.sender, reward, remainder);``, which is emitted at L245 of the ``withdraw()`` function. Without this event, the reward payout cannot be consistently monitored. 

[https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/amplifier/MuteAmplifier.sol#L287-L295](https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/amplifier/MuteAmplifier.sol#L287-L295)


The fix is as follows:

```diff
if (reward > 0) {
            uint256 week_time = 1 weeks;
            IDMute(dToken).LockTo(reward, week_time ,msg.sender);

            userClaimedRewards[msg.sender] = userClaimedRewards[msg.sender].add(
                reward
            );
            totalClaimedRewards = totalClaimedRewards.add(reward);
+           emit Payout(msg.sender, reward, remainder);    
        }
```

QA2: There is an edge case for ``state()`` in which one can still stake after the stake is over:

[https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/amplifier/MuteAmplifier.sol#L203-L223](https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/amplifier/MuteAmplifier.sol#L203-L223)

This is because of the following statements:

```javascript
 if (firstStakeTime == 0) {
            firstStakeTime = block.timestamp;
        } else {
            require(block.timestamp < endTime, "MuteAmplifier::stake: staking is over");
        }
```
So when ``firstStakeTime == 0 and block.timestamp > endTime``, it is still possible to stake. In other words, the function never check for the first staker whether the staking is over or not, it always allows the first staker to stake. 

Mitigation:
```diff
if (firstStakeTime == 0) {
            firstStakeTime = block.timestamp;
} 
- else {
            require(block.timestamp < endTime, "MuteAmplifier::stake: staking is over");
-        }

```